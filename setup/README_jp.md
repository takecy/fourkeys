# インストールガイド

このガイドでは、GitHubまたはGitLabプロジェクトにFour Keysを設定する方法について説明します。主な手順は以下の通りです。

1.  [セットアップスクリプトの実行](#running-the-setup-script)
1.  によってGitHubまたはGitLabのレポと統合する。
    1.  [変更データの収集](#collecting-changes-data)
    1.  [デプロイメントデータの収集](#collecting-deployment-data)
    1.  [インシデントデータの収集](#collecting-incident-data)

## Before you begin
> 四つの鍵のインストールには、[Cloud Shell](https://cloud.google.com/shell)を使用することをお勧めします。
1. GCloud SDK](https://cloud.google.com/sdk/install)をインストールします。
1. Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)をインストールする。
1. 課金が有効なGoogle Cloudプロジェクトでオーナーになる必要があります。現在アクティブなプロジェクトを使用するか、Four Keysで使用するために特別に新しいプロジェクトを作成することができます。

> :information_source。現在アクティブなgcloudプロジェクトと同じ課金アカウントを使用して新しいプロジェクトを作成するには、次のコマンドを実行します。
> ``sh
> エクスポート PARENT_PROJECT=$(gcloud config get-value project)
> エクスポート PARENT_FOLDER=$(gcloud projects describe ${PARENT_PROJECT} --format="value(parent.id)")
> エクスポート BILLING_ACCOUNT=$(gcloud beta billing projects describe ${PARENT_PROJECT} --format="value(billingAccountName)")
> export FOURKEYS_PROJECT=$(printf "fourkeys-%06d" $((RANDOM%999999)))
gcloud projects create ${FOURKEYS_PROJECT} --folder=${PARENT_FOLDER} > gcloud projects create ${FOURKEYS_PROJECT} --folder=${PARENT_FOLDER
gcloud beta billing projects link ${FOURKEYS_PROJECT} --billing-account=${BILING_ACCOUNT} > gcloud beta billing projects link --billing-account=${BILING_ACCOUNT
> echo "プロジェクトが作成されました。"$fourkeys_project
> 
> ```


## セットアップスクリプトの実行

1.  このリポジトリのトップレベル・ディレクトリから、以下のセットアップ・スクリプトを実行します。

    ``bash
    cd セットアップ
    スクリプト setup.log -c ./setup.sh
    ```
1.  セットアップスクリプトの質問に答えてください。

    * Four Keys をインストールするプロジェクトのプロジェクト ID と地域情報を入力します。
    * 設定するイベントソースを選択してください...
        * どのバージョン管理システムを使用していますか？
            * VCSの統合をスキップする場合は "その他 "を選択してください。
        * どのCI/CDシステムを使用していますか？
            * CICDシステムに適したオプションを選択するか、CICDとの統合をスキップする場合は "other "を選択してください。
        * セットアップ中に利用できないイベントソースを統合するには、 `/README.md#extending-to-other-event-sources` を参照してください）_。
    * モックデータを作成しますか？(y/N)
        * Yes を選択した場合、スクリプトが実行され、GitLab または GitHub の模擬イベントがイベントハンドラに送信されます。 これにより、ダッシュボードにモックデータが入力されます。 モックデータには、ソースに "mock "というワークが含まれます。セットアップスクリプトを使わなくても、モックデータを生成することができます。モックデータの生成](../readme.md)を参照してください。
            * ダッシュボードからモックデータを除外するには、SQLスクリプトを更新して、mockという単語が含まれるソースをフィルタリングするようにします。WHERE source not like "%mock"`.

### 変更を加える
セットアップスクリプトを実行した後、ある時点でインフラストラクチャに変更を加えたいと思うかもしれません。あるいは、Four Keysのレポ自体が新しい設定で更新されるかもしれません。Terraformの外でリソースに変更を加えた場合、その変更は追跡されず、Terraformで管理することはできません。これには、pub/subトピック、サブスクリプション、パーミッション、サービスアカウント、サービスなどが含まれます。そのため、インフラの変更は全て Terraform ファイルを更新して、`terraform apply` を使って Terraform を再実行することをお勧めします。変更内容を確認するプロンプトが表示されるので、よく確認してから `yes` と入力して次に進みます。
> Tip: Tip: このレポの設定は時間とともに進化し続けます。もし継続的なアップデートを適用したいのであれば、**追跡した Terraform ファイルを修正しないでください**。代わりに、[Terraform Override Files](https://www.terraform.io/docs/language/files/override.html) の使用を検討してください。これにより、次に上流からプルするときに潜在的なマージの衝突を引き起こすことなく、ニーズに合わせてインフラをカスタマイズすることができます。


### セットアップの説明
セットアップスクリプトは `README.md` で説明されているサービスアーキテクチャを作成するために多くのことを行います。その中には、少しのbashスクリプトと、多くの[Terraform](https://www.terraform.io/intro/)が含まれます。

ステップバイステップで、何が起こっているかを説明します。
1. 1. `setup.sh` は、Terraform に提供する設定変数を決定するために、システムとユーザーから情報を収集することから始めます。
1. いくつかの環境変数を設定し、Terraform への入力を含む `terraform.tfvars` ファイルをディスクに書き込みます。
1. そして、インフラストラクチャのプロビジョニングを行う `install.sh` を起動します。
1. 1. `install.sh` は `gcloud builds submit` コマンドを実行して、Cloud Run サービスで使用されるアプリケーションコンテナを構築します。
1. そして、Terraform を起動し、設定ファイル (末尾は `.tf`) を処理して、必要な全てのインフラを、指定したクラウド・プロジェクトにプロビジョニングします。
1. モックデータを生成するように設定した場合、スクリプトは ["data generator" python application](/data_generator/) を呼び出し、先ほど作成したイベントハンドラサービスにいくつかの合成ウェブフックイベントを送信します。
1. 最後に、スクリプトは、ウェブフックの設定やダッシュボードへのアクセスなど、次のステップに関する情報を表示します。

### Terraform の状態の管理
Terraformはインフラストラクチャに関する情報を、バックエンドと呼ばれる永続的なステートストレージに保持します。デフォルトでは、この情報は `terraform.tfstate` という名前のファイルに保存され、Terraform を実行したディレクトリに保存されます。このローカルなバックエンドは一度だけのセットアップには適していますが、Four Keysのインフラを維持し使用する予定がある場合は、リモートバックエンドを選択することをお勧めします。(あるいは、Terraformを初期設定にのみ使用し、継続的な修正には`gcloud`やCloud Consoleのような他のツールを使用することもできます。)

> Terraform の状態を堅牢に保存するためにリモートバックエンドを使用する方法については、以下を参照してください。[Terraform Language: Backends](https://www.terraform.io/docs/language/settings/backends/index.html) を参照してください。

### Terraformが作成したリソースのパージ
Terraform のセットアップ中に何か問題が発生した場合、`terraform destroy` を実行して、作成されたリソースを削除することができるかもしれません。しかし、Terraform の状態がプロジェクトと一致しなくなり、Terraform がリソースを認識しない状態になることがあります (ただし、リソースが存在するとその後のインストールがうまくいかなくなります)。このような場合、通常はGCPプロジェクトを削除し、新しいプロジェクトを立ち上げることが最良の選択です。それができない場合は、プロジェクト内の4つのキーリソースをすべて強制削除することができます。
``shell
./ci/project_cleaner.sh --project=<your_fourkeys_project> です。
```


## ライブレポとの連携
セットアップスクリプトはモックデータを作成することができますが、ライブプロジェクトと自動的に統合することはできません。 チームのパフォーマンスを測定するには、デプロイが行われている GitHub あるいは GitLab のライブリポジトリに統合する必要があります。そうすれば、4つの主要なメトリクスを測定し、変更、成功したデプロイメント、失敗したデプロイメントがメトリクスにどのような影響を与えるかを実験することができます。

Four Keysをライブリポジトリと統合するには、次のことが必要です。

1.  [変更データの収集](#collecting-changes-data)
1.  [デプロイメントデータの収集](#collecting-deployment-data)
1.  [インシデントデータの収集](#collecting-incident-data)

## The Four Keysの旧バージョンからの移行
現在では非推奨の [bash-based setup process](deprecated/) を使って作成した The Four Keys の既存のインストールがあり、新しいアップストリームリリースでインストールを最新に保ちたい場合、クラウドリソースを Terraform の制御下に置く必要があります。最も簡単な方法は、既存のリソースをすべて破棄し、Terraformに新しいリソースを作成させ、将来的に管理させることです。以下はそのための手順です(あなたのインストールに合わせて適宜修正してください)。
1. 1. [データのエクスポート](https://cloud.google.com/bigquery/docs/exporting-data#console) from `events_raw`
    * 削除する予定のプロジェクトのバケットにデータをエクスポートした場合は、プロジェクトを削除する前に必ずダウンロードしてください！_。
1. 既存のクラウドリソースを削除する
    * フォーキー専用のプロジェクトがある場合、そのプロジェクトを削除するだけです。
1. このフォルダの `setup.sh` を実行する。
    * インストール設定時に、モックデータを生成しないことを選択する。
1. セットアップが完了したら 
    1. 新しく生成された Webhook URL とシークレットを使用して、VCS/CICD システムから Webhook 配信を再設定します。
    1. events_raw` データをインポートします。
        1. [データを `events_raw_import` という名前の一時テーブル](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-csv#console) にロードする。
            * インポート時に手動でスキーマを指定する（カラムヘッダを削除する）必要がある場合があります。
        1. インポートしたデータを `events_raw` テーブルにコピーします。
            * `INSERT INTO events_raw (SELECT * FROM events_raw_import)`
        1. 一時テーブルを削除する

### 変更データの収集

#### GitHubでの操作手順

1.  GitHubのレポからスタート
1.  自分のレポ（またはフォークしたレポ）に移動し、**設定**をクリックします。
1.  左側から **Webhooks** を選択します。
1.  Webhookを追加する**をクリックします。
1.  Four Keysサービスのイベントハンドラーエンドポイントを取得します。
    bash
    echo $(terraform output -raw event_handler_endpoint)
    ```
1.  Add Webhook** インターフェースで、**Payload URL** のイベントハンドラーエンドポイントを使用します。
1.  1. 以下のコマンドを実行し、Google Secrets Managerから秘密を取得します。
    ``bash
    echo $(terraform output -raw event_handler_secret)
    ```
1.  シークレットを **Secret** と書かれたボックスに入れます。
1.  コンテンツタイプ**で、**application/json**を選択します。
1.  1. **Send me everything** を選択します。
1.  Webhookの追加**をクリックします。

#### GitLabの手順

1.  あなたのレポに移動し、**設定**をクリックします。
1.  メニューから **Webhooks** を選択します。
1.  1. 以下を実行して、Four Keys サービスのイベントハンドラーエンドポイントを取得します。
    ``bash
    echo $(terraform output -raw event_handler_endpoint)
    ```
1.  Payload URL** には、Event Handler のエンドポイントを使用します。
1.  以下のコマンドを実行し、Google Secrets Managerからsecretを取得します。
    ``bash
    echo $(terraform output -raw event_handler_secret)
    ```
1.  シークレットトークン**と書かれたボックスにシークレットを入れる。
1.  すべてのチェックボックスを選択します。
1.  SSL検証を有効にする**を選択したままにします。
1.  Webhookの追加**をクリックします。

### デプロイメントデータの収集

1.  使用しているCI/CDシステムについて、Webhookイベントをイベントハンドラに送信するように設定します。 

#### GitHubやGitlabのマージにCircleCIをデプロイするための設定
                             
1.  レポに `.circleci.yaml` ファイルを追加します。
    ```
    バージョン: 2.1
    executors:
      デフォルト
        ...
    ジョブ
      をビルドします。
        実行者: デフォルト
        のステップになります。
          - 実行: メイクビルド
      デプロイします。
        実行者: デフォルト
        の手順で行います。
          - 実行：メイクデプロイ
    ワークフローを
      バージョン: 2
      build_and_deploy_on_master を使用します。# 'deploy'を名前に含むワークフローは、デプロイメントビューを構築するためのクエリで使用されます。
        のジョブがあります。
          - build:
              名前: ビルド
              フィルター: &master_filter
                ブランチ
                  のみ: マスター
          - デプロイします。
              名前: デプロイ
              フィルタを使用します。*master_filter
              が必要です。
                - ビルド
    ```
 
この設定は、`master` ブランチへの `push` の際にデプロイを開始します。

### インシデントデータの収集

Four Keys は、GitLab や GitHub issues を使ってインシデントを追跡しています。 

#### インシデントを作成する

1.  課題を開く
1.  タグ `Incident` を追加します。
1.  課題の本文に、`root cause'と入力します。{コミットのSHA}`を入力します。

インシデントが解決されたら、課題を閉じます。Four Keysは、デプロイ時から課題をクローズするまでのインシデントを測定します。

