# AWS Copilot

## 概要

AWS Copilotは、Amazon ECS CLIの後継に当たるコマンドラインツールです。

このAWS Copilotコマンドを用いて、VPC、セキュリティグループ、インターネットゲートウェイ、パブリックサブネット、プライベートサブネット、ECSクラスター、IAMを作成します。



## 前提条件

* Copilot v1.7.1以降（必要に応じてインストールガイドを参照）
  * https://aws.github.io/copilot-cli/docs/getting-started/install/
    * macOSの場合は、brewが使えます。
* Docker Desktop (または、Linux環境では、Docker Engine)

## 概念

Application -> Environment -> Service

portal -> prod or test -> api

## AWS Profileを設定

```bash
export AWS_PROFILE="admin-role"
```

AdministratorAccessポリシーが付与されたロールを用います。
値は環境によって異なります。

## copilot initでApplicationを作成

copilot initでApplication・Environment・Serviceを一気に作れるので、ここではtest環境（STG環境）を作成する。

```bash
copilot init \
--app portal \
--deploy \
--dockerfile ./Dockerfile \
--name api \
--type "Load Balanced Web Service"
```

約5~10分で、APIエンドポイントで作成される。

```
✔ Deployed web, you can access it at http://~~.ap-northeast-1.elb.amazonaws.com.
```

ローカルのプロジェクトフォルダーに、.workspaceファイルが生成されることで、これ以降のcopilotコマンドはApplicationを .workspaceから参照するようになります。


## copilot envでEnvironmentにProd環境を追加

```
copilot env init \
--name prod \
--prod \
--default-config \
--profile $AWS_PROFILE
```

## ローカルにServiceを作成するyamlを生成する

```
copilot svc init \
--app portal \
--dockerfile ./Dockerfile \
--name api \
--svc-type "Load Balanced Web Service"
```

### manifest.yamlを用いて（test環境、もしくは、Prod環境）のサービスにデプロイする

```
copilot svc deploy --name api --env test
```

```
copilot svc deploy --name api --env prod
```

Serviceデプロイの手順は以下の通りです。

1. ローカルのDockerfileをビルドしてコンテナイメージを作成
2. --tagの値、または最新のgit sha（gitディレクトリで作業している場合）を利用してタグ付け
3. コンテナイメージをECRにプッシュ
4. ManifestファイルとアドオンをCloudFormationにパッケージ
5. ECSタスク定義とサービスを作成 / 更新

## PipelineをつなげるリポジトリをCodeCommitに作成する

```
aws codecommit create-repository \
--repository-name portal \
--repository-description "My portal repository" \
> ./secret-portal-codecommit-result.json
```

結果はjsonファイルに保存する。

## AWS CLIでCodeCommitのリポジトリに初回コミットをする。

このコマンドで、リポートリポジトリに対して、mainブランチを作成し、そこにBLANK.mdを追加します。

```
aws codecommit put-file \
--repository-name portal \
--branch-name main \
--file-path BLANK.md \
--file-content file://./BLANK.md \
--name "yusuke.ono" \
--email "yusuke.ono@example.com" \
--commit-message "I added a quick readme for our new team repository."
```


## CodeCommitのリポジトリをクローンする

CodeCommitのリポジトリに接続する方法はいくつかありますが、ここではgit-remote-codecommitを用いてアクセスします。git-remote-codecommitは、IAMユーザーを新たに作成する必要はなく、上記のAdministratorAccessポリシーが付与されたロールを持つIAMユーザーでGitにアクセスできます。詳細は以下のURLをご参照ください。

https://docs.aws.amazon.com/ja_jp/codecommit/latest/userguide/setting-up-git-remote-codecommit.html

```
git clone codecommit::ap-northeast-1://portal
```

（先にCodeCommitリポジトリを生成したほうがいいかも）

## copilot pipelineでPipelineを追加する

```
copilot pipeline init \
--app portal \
--environments "test,stg,prod" \
--git-branch main \
--url https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/portal
```

パイプライン雛形ファイルが、でき上がる。
copilot/pipeline.yml
copilot/buildspec.yml

result.

```
Required follow-up actions:
- Commit and push the buildspec.yml, pipeline.yml, and .workspace files of your copilot directory to your repository.
- Run `copilot pipeline update` to create your pipeline.
```

ymlファイルをCodeCommitにpushして、`copilot pipeline update`コマンドを実行して、パイプラインを作成します。

これ以降において、CodeCommitにpushすると、CodePipelineにてビルドが自動的に動作するようになります。進行状況は、CodePipelineで確認できます。

なお、Prod環境へのデプロイについては、手動承認が必須となります。手動承認については、CodePipelineで実施できます。

## Applicationを確認

```
copilot app show
```

## Application内にあるEvrironmentを確認

```
copilot env ls
```

## Evironmentに含まれるものを確認

```
copilot env show
```

## サービスを確認

```
copilot svc ls \
--app portal
```

## Appicationを更新？

```
copilot app upgrade \
--name portal
```

## Pipelineを削除

```
copilot pipeline delete 
```

## Appicationを削除

注意！動作中のServiceやEnvrionmentも一緒に削除されます。

```bash
copilot app delete
```

### リポジトリ削除

```
aws codecommit delete-repository \
--repository-name portal
```

Prod環境も削除されるので取扱注意。（もしかしてcopilotで作成したクラスターが全部消えるかもしれない。）

# TODO

* ALBをHTTPSにする

## 環境変数を設定する

https://aws.github.io/copilot-cli/ja/docs/developing/environment-variables/

* ユースケース
  * Envrionmentに応じて、ログレベルを切り替えたい場合。
  * Envrionmentに応じて、DBの接続先を切り替えたい場合。
  
## 環境変数を設定する（クレデンシャルの場合）

https://aws.github.io/copilot-cli/ja/docs/developing/secrets/

https://aws.github.io/copilot-cli/ja/docs/commands/secret-init/

* ユースケース
  * DBの接続ID/PWを設定したい場合。

## AWSリソースを追加したい

https://aws.github.io/copilot-cli/ja/docs/developing/additional-aws-resources/

## ストレージサービスを追加したい

S3、DynamoDB、Auroraしか対応していない。それ以外は、上記のAWSリソースを追加したいを参照し、CloudFormationテンプレートを記述する必要がある。

## オートスケールとかの設定変更したい

copilot/manifest.yml変えてdeployするだけ。

## 既存のVPCにECSクラスターを作成したい

https://aws.github.io/copilot-cli/docs/commands/env-init/

* ユースケース
  * 謎の大人の事情がある場合。

## lasted運用を見直した方がいいらしい

https://speakerdeck.com/hamadakoji/anatafalsezu-zhi-nizui-shi-nakontenadepuroifang-fa-toha-ecsniokerudepuroizui-xin-ji-neng-tenkosheng-ri?slide=26

## Copilotの機能で実現できないこと

https://speakerdeck.com/tobachi/introduction-to-amazon-ecs-and-aws-copilot-for-container-beginners-on-akiba-dot-aws-online-number-2?slide=53

https://speakerdeck.com/iselegant/the-cooking-of-aws-copilot

## ECSにおけるデプロイ戦略

https://speakerdeck.com/hamadakoji/anatafalsezu-zhi-nizui-shi-nakontenadepuroifang-fa-toha-ecsniokerudepuroizui-xin-ji-neng-tenkosheng-ri

## ECSにおけるログ可視化

* ユースケース
  * コンテナー単位でメトリクスを取得できる

https://dev.classmethod.jp/articles/container-insights-ga/
