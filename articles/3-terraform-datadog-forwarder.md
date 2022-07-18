---
title: "Datadogへログを送るLambdaをTerraformでシンプルに構築する"
emoji: "🐶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Terraform','AWS','Datadog','techblog']
published: false
---

こんにちは、[株式会社スマートラウンド](https://jobs.smartround.com/)のSREの[@shonansurvivors](https://twitter.com/shonansurvivors)です。

当社はインフラにAWSを使っており、ALBのログはS3バケットに保存するようにしています。

S3に保存したALBログはAthenaなどを使って分析することもできるのですが、Datadogに送ることで非常に簡単に可視化することができ、パフォーマンス分析などに大いに役立っています。

下図のような、Datadogにログを送るLambdaのことをDatadog公式ではDatadog Forwarderと呼んでいます。

![](/images/article3/datadog-forwarder-structure.png)

このDatadog Forwarderおよび一連の関連リソースを構築する手段として、Datadog公式からは[CloudFormationテンプレート](https://docs.datadoghq.com/ja/logs/guide/forwarder/#cloudformation)が提供されています。また、このCloudFormationテンプレートを[Terraformでラップして使用する方法](https://docs.datadoghq.com/ja/logs/guide/forwarder/#terraform)についても解説されています。

当社はIaCにTerraformを採用しており、Datadog ForwarderもTerraformで構築したのですが、公式とは少し違う方法で構築しました。

よりシンプルかつ運用しやすいコードになったと思うので本記事で解説させていただきます。

# 工夫したポイント

- Datadog APIキーを保管するSecrets Managerを自作せず、Datadog公式のCloudFormationで作らせた
- Datadog APIキーをTerraformのコードに含めないようにした

# 環境

- Datadog Forwarder 3.51.0

# サンプルコード

```hcl
resource "aws_cloudformation_stack" "datadog_forwarder" {
  name         = "datadog-forwarder"
  capabilities = ["CAPABILITY_IAM", "CAPABILITY_NAMED_IAM", "CAPABILITY_AUTO_EXPAND"]
  parameters = {
    DdApiKey     = "dummy" # 今回の記事のポイント その1
    DdSite       = "app.datadoghq.com"  # 使用中のDatadogサイトに合わせて、us3.datadoghq.com, us5.datadoghq.comなどに適宜変更してください
    FunctionName = "datadog-forwarder"
  }
  template_url = "https://datadog-cloudformation-template.s3.amazonaws.com/aws/forwarder/3.51.0.yaml" # バージョンは固定しました

  lifecycle {
    ignore_changes = [
      parameters["DdApiKey"] # 今回の記事のポイント その2
    ]
  }
}
```

# 解説

## Datadog公式の方法

まず、当社の方法ではなく、Datadog公式の方法から解説します。

Datadog公式では、まず最初にSecrets Managerを作成する流れとなっています。

```hcl
# Store Datadog API key in AWS Secrets Manager
variable "dd_api_key" {
  type        = string
  description = "Datadog API key"
}

resource "aws_secretsmanager_secret" "dd_api_key" {
  name        = "datadog_api_key"
  description = "Encrypted Datadog API Key"
}

resource "aws_secretsmanager_secret_version" "dd_api_key" {
  secret_id     = aws_secretsmanager_secret.dd_api_key.id
  secret_string = var.dd_api_key
}

output "dd_api_key" {
  value = aws_secretsmanager_secret.dd_api_key.arn
}
```

このコードを`terraform apply`すると、以下のように変数`dd_api_key`の値を聞かれますので、Datadog APIキーの値をコンソールに入力します。

```shell
var.dd_api_key
  Datadog API key

  Enter a value: 
```

applyが完了したら、以下のコードの`DdApiKeySecretArn`に前述のSecrets ManagerのARNを渡した上で、こちらもapplyします。

```hcl
resource "aws_cloudformation_stack" "datadog_forwarder" {
  name         = "datadog-forwarder"
  capabilities = ["CAPABILITY_IAM", "CAPABILITY_NAMED_IAM", "CAPABILITY_AUTO_EXPAND"]
  parameters   = {
    DdApiKeySecretArn  = "REPLACE ME WITH THE SECRETS ARN"
    DdSite             = "datadoghq.com"
    FunctionName       = "datadog-forwarder"
  }
  template_url = "https://datadog-cloudformation-template.s3.amazonaws.com/aws/forwarder/latest.yaml"
}
```

なお、公式では前述の2つのTerraformコードを別ディレクトリに配置することを想定しているようです。

> Separating the configurations of the API key and the forwarder means that you don’t need to provide the Datadog API key when updating the forwarder.
> https://docs.datadoghq.com/logs/guide/forwarder/#terraform

そうすることで、以後CloudFormation側のコードをplan/applyする必要が生じた際に、変数`dd_api_key`の値を毎回聞かれるということが無くなります。

## 当社の方法: DdApiKeySecretArnではなくDdApiKeyを指定する

このような公式の方法に対し、当社では`DdApiKeySecretArn`ではなく`DdApiKey`を指定するようにしました。

そうすることで、このCloudFormationは`DdApiKey`に渡された値を持つSecrets Managerを作成してくれます(興味のある方はCloudFormationテンプレートの[中身](https://github.com/DataDog/datadog-serverless-functions/blob/aws-dd-forwarder-3.51.0/aws/logs_monitoring/template.yaml)を追ってみてください)。

なお、`DdApiKey`には実際のAPIキーの値を記述するのではなく、`"dummy"`にしておきます。これはAPIキーのような機微情報をTerraformのコード上に含めたくないためです。

```hcl
resource "aws_cloudformation_stack" "datadog_forwarder" {
  name         = "datadog-forwarder"
  capabilities = ["CAPABILITY_IAM", "CAPABILITY_NAMED_IAM", "CAPABILITY_AUTO_EXPAND"]
  parameters = {
    DdApiKey     = "dummy"
    DdSite       = "app.datadoghq.com"
    FunctionName = "datadog-forwarder"
  }
  template_url = "https://datadog-cloudformation-template.s3.amazonaws.com/aws/forwarder/3.51.0.yaml"

  lifecycle {
    ignore_changes = [
      parameters["DdApiKey"]
    ]
  }
}
```

上記をapplyしたら、マネジメントコンソールかAWS CLIを使って、作成されたSecrets Managerの値をAPIキーの値で別途上書きします。

上書きしたことでAWS上のSecrets Managerの値と、Terraformコード上の`DdApiKey`の値に差異が生まれますが、lifecycleブロックのignore_changesを指定しているので、次回以降のplan/apply時に差分として検知されることはありません。

以上の方法により、公式のように別ディレクトリにSecrets Managerのコードを書く必要は無くなり、シンプルな構成になりました。

# おまけ: S3からLambdaを起動するための設定

ついでに、ログを保存しているS3からLambdaを起動するためのAWSリソースを作成するコードも載せておきます。こちらも参考にしてみてください。

```hcl
resource "aws_cloudformation_stack" "datadog_forwarder" {
  # 略
}

data "aws_s3_bucket" "alb_access_logs" {
  bucket = "ALBログを保存しているS3バケット名"
}

resource "aws_s3_bucket_notification" "alb_access_logs" {
  bucket = data.aws_s3_bucket.alb_access_logs.id

  lambda_function {
    id = "datadog-forwarder"
    events = [
      "s3:ObjectCreated:Put"
    ]
    lambda_function_arn = aws_cloudformation_stack.datadog_forwarder.outputs.DatadogForwarderArn
  }
}

resource "aws_lambda_permission" "datadog_forwarder_s3_alb_access_logs" {
  action         = "lambda:InvokeFunction"
  function_name  = aws_cloudformation_stack.datadog_forwarder.outputs.DatadogForwarderArn
  principal      = "s3.amazonaws.com"
  source_account = "AWSアカウントID"
  source_arn     = data.aws_s3_bucket.alb_access_logs.arn
  statement_id   = "datadog-forwarder-s3-alb-access-logs"
}
```

# おわりに

以上、Datadog ForwarderをTerraformでシンプルに構築する方法の解説でした。これからDatadogのLog Managementの機能を利用しようと考えている方の参考になれば幸いです。

# 採用情報

株式会社スマートラウンドではエンジニアを募集中です！正社員はもちろん、副業でのジョインも歓迎です。少しでもご興味あればお気軽にご応募ください。Meetyでのカジュアル面談も可能です。

https://jobs.smartround.com/
