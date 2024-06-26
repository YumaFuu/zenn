---
title: "ecspressoで作るRDSの踏み台Fargate"
emoji: "💨"
type: "tech"
topics: ["ECS", "aws", "ecspresso", "fargate"]
published: true
publication_name: "ispec_inc"
---

# はじめに

ECS Fargate経由でプライベートなSubnetにあるRDSにアクセスする方法をまとめます

使う時だけTaskを起動するので、余計にEC2インスタンスを起動しておく必要がなくなったり、パッケージの管理等もなくなったりいいことづくしです✌️

使うツールは以下の通り
- こんな素晴らしいツールあったの？でお馴染み [ecspresso](https://github.com/kayac/ecspresso)
- [ECS Exec](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/ecs-exec.html)のポートフォワード機能

最終的にlocalhostに向けて以下のようにアクセスできるようになります

```bash
$ mysql -uuser -ppassword --host 127.0.0.1
```

CLI生まれCLI育ちの僕でもRDBはGUIクライアントを使いたいのでこの方法はとても有用だと思います(sshの場合はCLIでしかアクセスできない)


# ECS Execとは

実行中のコンテナに入ってコマンドを実行できるもの(docker execのECS版)です。
今回はその中のポートフォワード機能を使います

[参考: デバッグ用にAmazon ECS Exec を使用](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/ecs-exec.html)

# 実装

## IAM Role

TaskRoleに以下のSSMのアクセス権限が必要ですので事前にアタッチしておきます

SSM AgentをバインドマウントしてSSMのセッションマネージャーと通信するらしいです
[参考: アーキテクチャ](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/ecs-exec.html#ecs-exec-architecture)


```json:policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowObjectAccess",
      "Effect": "Allow",
      "Action": [
        "ssmmessages:CreateControlChannel",
        "ssmmessages:CreateDataChannel",
        "ssmmessages:OpenControlChannel",
        "ssmmessages:OpenDataChannel",
      ],
      "Resource": "*"
    }
  ]
}

```

Roleを作成するサンプルスクリプト

https://github.com/YumaFuu/ecspresso-portforward/blob/main/create-task-role.sh

実行ロールは特になんのアクションも許可する必要ありません

[amazon-ecs-exec-checker](https://github.com/aws-containers/amazon-ecs-exec-checker)でECS Execが使えるかの確認ができます


## ecspresso.yaml

いつものyamlです

```yaml:ecspresso.yaml
region: ap-northeast-1
cluster: your-cluster
service_definition: service-def.jsonnet
task_definition: task-def.jsonnet
```

## task-def.jsonnet

sleepだけすればいいので C言語で書いたsleepするだけのイメージ [yumafuu/sleepy](https://github.com/YumaFuu/docker-sleepy) を使います
https://github.com/YumaFuu/docker-sleepy

約20KBなので軽くて良きです◎

嫌な方はsleepできるお好きなイメージに差し替えてください

```json:task-def.jsonnet
{
  family: "rds-bastion",
  cpu: "256",
  memory: "512",
  executionRoleArn: "arn:aws:iam::000000000000:role/bastion-task-exec-role"
  taskRoleArn: "arn:aws:iam::000000000000:role/bastion-task-role"
  networkMode: "awsvpc",
  containerDefinitions: [
    {
      name: "bastion",
      image: "yumafuu/sleepy:latest",
      command: [
        "600", // 10分だけ起動する
      ],
    }
  ],
}

```

## service-def.jsonnet

サービスは使いませんが、ネットワークやECS Execの設定を記述します

```json:service-def.jsonnet

{
  launchType: "FARGATE",
  enableExecuteCommand: true, // ECS Execをするのに必要！
  networkConfiguration: {
    awsvpcConfiguration: {
      securityGroups: [
        "sg-00000000000000000", // RDSへの通信を許可しておいてね
      ],
      subnets: [
        "subnet-00000000000000000"
      ],
    }
  },
}

```

## run.sh

実行のためのshellスクリプトです

```bash:run.sh
RDS_HOST=rds-cluster.cluster-xxxxxxxxx.ap-northeast-1.rds.amazonaws.com

# --wait-until=runningでTaskが起動するまで待つ
ecspresso run --wait-until=running

# 最新のTask IDを取得
id=$(
    ecspresso tasks --output=json | \
    jq -r '.containers[0].taskArn | split("/")[2]' | \
    head -1 \
)

# ポートフォワードする
ecspresso exec \
  --port-forward \
  -L 3306:$RDS_HOST:3306 \
  --id $id
```

* 追記(1/20)
--wait-untilとtasksコマンドを@fujiwaraさんにコメントで教えていだだきました！ありがとうございます🙇‍♂️

* 追記(5/25)
[v2.3.4](https://github.com/kayac/ecspresso/releases/tag/v2.3.4)にて`-L`オプションが追加されたので修正しました。


bashで実行したら20秒くらいでTaskが起動します

```bash
$ bash ./run.sh
# ....

Waiting for connections...
```
Waiting for connections... が出たらlocalhostにアクセスできるようになっています！

```bash
$ mysql -uuser -ppassword --host 127.0.0.1
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 247708
Server version: 8.0.28 Source distribution

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

admin@172.16.3.11 [(none)] 02:35 pm>
```

ちなみにcommandで指定した秒数でTaskが死ぬので、適宜変更が必要です

# ソースコード

以下にまとめておきましたので詳しくみたい方はこちらから！

https://github.com/YumaFuu/ecspresso-portforward

# おわりに

ecspresso本当に素晴らしい、、🤙

他にもecspressoの運用を以下で紹介してますので是非見てみてください！

https://zenn.dev/ispec_inc/articles/ecspresso-lt

