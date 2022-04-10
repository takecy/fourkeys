![フォーキー](images/fourkeys_wide.svg)

[![Four Keys YouTube Video](images/youtube-screenshot.png)](https://www.youtube.com/watch?v=2rzvIL29Nz0 "Measuring Devops: The Four Keys Project")

# 背景

6年間の研究を通して、[DevOps Research and Assessment (DORA)](https://cloud.google.com/blog/products/devops-sre/the-2019-accelerate-state-of-devops-elite-performance-productivity-and-scaling) チームは、ソフトウェアデリバリーのパフォーマンスを示す4つの主要なメトリクスを特定しました。Four Keysは、開発環境（GitHubやGitLabなど）からデータを収集し、これらの主要なメトリクスを表示するダッシュボードにまとめることができます。

これら4つの主要なメトリクスは以下の通りです。

* デプロイメントの頻度**。
* 変更のためのリードタイム
* サービス復旧までの時間**。
* 変更の失敗率**について

# フォーキーを使うべき人

以下のような場合、Four Keysをお使いください。

* チームのソフトウェアデリバリーパフォーマンスを測定したい場合。例えば、新しいツールや自動化されたテストカバレッジの影響を追跡したい場合や、チームのパフォーマンスの基準値が必要な場合などです。
* GitHub または GitLab にプロジェクトを持っている。
* プロジェクトにデプロイメントがある。

Four Keys は、デプロイメントを行うプロジェクトに適しています。GitHubやGitLabがリリースに関するデータをどのように表示しているかによって、リリースがありデプロイがないプロジェクト（たとえばライブラリなど）はうまく機能しません。

チームのソフトウェアデリバリパフォーマンスのベースラインを素早く確認するには、[DORA DevOps Quick Check](https://www.devops-research.com/quickcheck.html)を使うことも可能です。クイックチェックは、パフォーマンスを向上させるために取り組むべきDevOpsの能力も示唆しています。Four Keysプロジェクト自体は、以下のDevOps能力を向上させるのに役立ちます。

* [モニタリングと観測可能性](https://cloud.google.com/solutions/devops/devops-measurement-monitoring-and-observability)
* [ビジネス上の意思決定に資するモニタリングシステム](https://cloud.google.com/solutions/devops/devops-measurement-monitoring-systems)
* [ビジュアル管理機能](https://cloud.google.com/solutions/devops/devops-measurement-visual-management)

# 仕組み

1.  イベントは、クラウドランにホストされているWebhookターゲットに送信されます。イベントとは、開発環境（例えばGitHubやGitLab）で発生した、プルリクエストや新規課題のような測定可能なあらゆる事象を指します。Four Keysは測定するイベントを定義しており、プロジェクトに関連する他のイベントを追加することができます。
1.  Cloud Run ターゲットは、すべてのイベントを Pub/Sub にパブリッシュします。
1.  Cloud RunインスタンスはPub/Subトピックを購読し、軽いデータ変換を行い、データをBigQueryに入力します。
1.  BigQueryのビューでデータ変換を完了し、ダッシュボードに入力します。

この図は、Four Keysシステムの設計を示したものです。

![FourKeysの設計図](images/fourkeys-design.png)

# コード構成

* `bq-workers/`
  * 個々のBigQueryワーカーのコードが含まれます。 各データソースは、Pub/Subメッセージからデータをパースするロジックを持つ独自のワーカーサービスを持ちます。例えば、GitHubはGitHub-Hookshot Pub/Subトピックにプッシュされたイベントのみを参照する独自のワーカーを持っています。
* `connectors/`
  * Four Keys Dashboard を生成する DataStudio Connector のコードが含まれています。
* データジェネレータ/`（DATA_GENERATOR
  * GitHubやGitlabのモックデータを生成するためのPythonスクリプトが含まれています。
* `イベントハンドラ/` が含まれています。
  * Webhook を受け付けるパブリックサービスである `event_handler` のコードが含まれています。 
* `queries/`
  * 派生テーブルを作成するためのSQLクエリが含まれています。
* `setup/`
  * Four Keysパイプラインのセットアップとテイルダウンのコードが含まれています。データソースを拡張するためのスクリプトも含まれています。
* `shared/`
  * BigQuery にデータを挿入するための共有モジュールが含まれており、`bq-worker` が使用する。


# 使い方 

## Out of the box

_このプロジェクトはPython 3を使用し、Cloud BuildとGitHubイベントのデータ抽出をサポートしています。

1.  このプロジェクトをフォークする。
1.  以下のような自動化スクリプトを実行します（詳細は [setup README](setup/README.md) を参照してください）。
    1.  Cloud RunのWebhookターゲットとETLワーカーを作成し、デプロイします。
    1.  Pub/Subトピックとサブスクリプションを作成する。
    1.  Google シークレットマネージャーを有効にし、GitHub リポのシークレットを作成します。
    1.  BigQueryのデータセット、テーブル、ビューを作成します。
    1.  ブラウザタブを開き、データをDataStudioダッシュボードテンプレートに接続します。
1.  1.2.で作成したWebhookにイベントを送信するように開発環境をセットアップします。
    1.  GitHubのWebhookにsecretを追加します。

注意: トランクにマージする際に、Git の "Squash Merging" を使用しないように注意してください。これは、trunkへのコミットとあなたが開発したブランチのコミットとの間のリンクを壊してしまうため、これらのコミットに対する "Time to Change" を測定することができなくなります。この機能は、リポジトリの設定で無効にすることができます。
## モックデータの作成

セットアップスクリプトには、モックデータを生成するためのオプションが含まれています。モックデータを生成することで、Four Keysプロジェクトで遊んだりテストしたりすることができます。

このデータジェネレーターはGitHubのイベントをモック化し、"githubmock "というソースでテーブルに取り込みます。以下のようなイベントが作成されます。

* 5つのモックコミット、タイムスタンプは1週間前まで
  * 注：数は調整可能です。
* 1 関連するデプロイメント
* 関連するモックインシデント 
  * 注：デフォルトでは、デプロイメントの15%未満がモックインシデントを作成します。この閾値はスクリプトで調整できます。

セットアップスクリプトの外で実行するには

1. WebhookのURLとsecretが環境変数に保存されていることを確認します。

   ``sh
   export WEBHOOK={あなたのイベントハンドラのURL}をエクスポートします。
   export SECRET={あなたのイベントハンドラの秘密}をエクスポートします。
   ```

1. 以下のコマンドを実行します。

   ``sh
   python3 data_generator/generate_data.py --vc_system=github
   ```

   パイプラインで実行されているこれらのイベントを確認することができます。
   * イベントハンドラのログには、リクエストの成功が表示されます。
   * Pub/Subトピックは投稿されたメッセージを表示します。
   * BigQueryのGitHubパーサーはリクエストに成功したことを表示します。

1.  BigQuery で `events_raw` テーブルに直接クエリを発行することができます。

    ``sql
    SELECT * FROM four_keys.events_raw WHERE source = 'githubmock';
    ```

## イベントの再分類／クエリの更新

スクリプトでは、いくつかのイベントを "変更"、"デプロイ"、および "インシデント" とみなしています。例えば、インシデントに "incident" 以外のラベルを使いたい場合、これらのイベントの一つ以上を再分類したいかもしれません。表中のイベントの1つを再分類するために、プロジェクトのアーキテクチャやコードに変更を加える必要はありません。

1.  以下のテーブルについて、BigQuery のビューを更新します。

    * `four_keys.changes`
    * `four_keys.deployments`（フォーキー・ディプロイメント）。
    * `four_keys.incidents`

    ビューを更新するには、BigQuery UI 内ではなく、`queries` フォルダ内の `sql` ファイルを更新することをお勧めします。

1.  1. SQL を編集したら、`terraform apply` を実行して、テーブルにデータを入力するビューを更新します。

    ``sh 
    cd ./setup && terraform apply
    ```

注意事項 

* ダッシュボードに反映させるには、テーブル名を `changes`, `deployments`, `incidents` のいずれかにする必要があります。


## 他のイベントソースへの拡張

他のイベントソースを追加するには

1.  sources.py` の `AUTHORIZED_SOURCES` に追加します。
    1.  検証関数を作成する場合は、その関数もこのファイルに追加します。
1.  setup` ディレクトリにある `new_source.sh` スクリプトを実行します。このスクリプトは `new_source_template` を使用して、Pub/Sub トピック、Pub/Sub サブスクリプション、および新しいサービスを作成します。
    1.  1. 新しいサービスの `main.py` を更新して、データを適切にパースするようにします。
1.  1. BigQueryスクリプトを更新して、データを適切に分類する。

**共通のデータソースを追加した場合は、他の人がその機能を利用できるように、プルリクエストを提出してください。


## テストの実行
このプロジェクトでは、テストの管理にnoxを使用しています。noxfile` はプロジェクトでどのようなテストを実行するかを定義します。すべてのディレクトリにある `pytest` ファイルを実行し、すべてのディレクトリで linter を実行するように設定されています。

noxを実行するには、以下のようにします。

1.  noxがインストールされていることを確認します。

    ``sh
    pip install nox
    ```

1.  Use the following command to run nox:

    ```sh
    python3 -m nox
    ```

### Listing tests

To list all the test sessions in the noxfile, use the following command:

```sh
python3 -m nox -l
```

### Running a specific test

Once you have the list of test sessions, you can run a specific session with:

```sh
python3 -m nox -s "{name_of_session}" 
```

The "name_of_session" will be something like "py-3.6(folder='.....').  

# Data schema

### `four_keys.events_raw`

<table>
  <tr>
   <td><strong>Field Name</strong>
   </td>
   <td><strong>Type</strong>
   </td>
   <td><strong>Notes</strong>
   </td>
  </tr>
  <tr>
   <td>source
   </td>
   <td>STRING
   </td>
   <td>eg: github
   </td>
  </tr>
  <tr>
   <td>event_type
   </td>
   <td>STRING
   </td>
   <td>eg: push
   </td>
  </tr>
  <tr>
   <td> <strong>id*</strong>
   </td>
   <td>STRING
   </td>
   <td>Id of the development object. Eg, bug id, commit id, PR id
   </td>
  </tr>
  <tr>
   <td>metadata
   </td>
   <td>JSON
   </td>
   <td>Body of the event
   </td>
  </tr>
  <tr>
   <td>time_created
   </td>
   <td>TIMESTAMP
   </td>
   <td>The time the event was created
   </td>
  </tr>
  <tr>
   <td>signature
   </td>
   <td>STRING
   </td>
   <td>Encrypted signature key from the event. This will be the <strong>unique key</strong> for the table.  
   </td>
  </tr>
  <tr>
   <td>msg_id
   </td>
   <td>STRING
   </td>
   <td>Message id from Pub/Sub
   </td>
  </tr>
</table>

*indicates that the ID is generated by the original system, such as GitHub.

This table will be used to create the following three derived tables: 


#### `four_keys.deployments` 

_Note: Deployments and changes have a many to one relationship.  Table only contains successful deployments._


<table>
  <tr>
   <td><strong>Field Name</strong>
   </td>
   <td><strong>Type</strong>
   </td>
   <td><strong>Notes</strong>
   </td>
  </tr>
  <tr>
   <td>🔑deploy_id
   </td>
   <td>string
   </td>
   <td>Id of the deployment - foreign key to id in events_raw
   </td>
  </tr>
  <tr>
   <td>changes
   </td>
   <td>array of strings
   </td>
   <td>List of id’s associated with the deployment. Eg: commit_id’s, bug_id’s, etc.  
   </td>
  </tr>
  <tr>
   <td>time_created
   </td>
   <td>timestamp
   </td>
   <td>Time the deployment was completed
   </td>
  </tr>
</table>


#### `four_keys.changes`


<table>
  <tr>
   <td><strong>Field Name</strong>
   </td>
   <td><strong>Type</strong>
   </td>
   <td><strong>Notes</strong>
   </td>
  </tr>
  <tr>
   <td>🔑change_id
   </td>
   <td>string
   </td>
   <td>Id of the change - foreign key to id in events_raw
   </td>
  </tr>
  <tr>
   <td>time_created
   </td>
   <td>timestamp
   </td>
   <td>Time_created from events_raw
   </td>
  </tr>
  <tr>
   <td>change_type
   </td>
   <td>string
   </td>
   <td>The event type
   </td>
  </tr>
</table>


#### `four_keys.incidents`


<table>
  <tr>
   <td><strong>Field Name</strong>
   </td>
   <td><strong>Type</strong>
   </td>
   <td><strong>Notes</strong>
   </td>
  </tr>
  <tr>
   <td>🔑incident_id
   </td>
   <td>string
   </td>
   <td>Id of the failure incident
   </td>
  </tr>
  <tr>
   <td>changes
   </td>
   <td>array of strings
   </td>
   <td>List of deployment ID’s that caused the failure
   </td>
  </tr>
  <tr>
   <td>time_created
   </td>
   <td>timestamp
   </td>
   <td>変更点からの最小タイムスタンプ
   </td
  </tr
  <tr>
   <td>時間_解決済み
   </td
   <td>タイムスタンプ
   </td
   <td>インシデントが解決された時間
   </td
  </tr
</table>。


# ダッシュボード 

4つの鍵のダッシュボードの画像](images/dashboard.png)

ダッシュボードには、4つの指標すべてが日々のシステムデータとともに、過去90日間の現在のスナップショットとして表示されます。主要な指標の定義と色分けの説明は以下のとおりです。

メトリクスとダッシュボードの意図についてより深く理解するためには、[2019 State of DevOps Report](https://www.devops-research.com/research.html#reports)を参照してください。

Four Keysがこのダッシュボードの各指標を計算する方法の詳細については、[Four Keys Metrics calculation](METRICS.md) docを参照してください。


## キーメトリクスの定義

このFour Keysプロジェクトでは、キーメトリクスを以下のように定義しています。

**デプロイメント頻度**」。

* 毎日、毎週、毎月、毎年など、チームが本番環境へのリリースを成功させる頻度。

**変更のためのリードタイム**。

* コミットが本番環境に導入されるまでの時間の中央値。

**サービス復旧までの時間**。

* 障害の場合、障害の原因となったデプロイメントから修復までの時間の中央値。 修復は、関連するバグ/インシデントレポートのクローズによって測定される。

**変更の失敗率**。

* デプロイメント数に対する失敗の数。例えば、1日に4回のデプロイがあり、1回失敗した場合、25％の変更失敗率になる。

メトリクスの計算についての詳細は、[METRICS.md](METRICS.md) を参照してください。

## 色分け

ダッシュボードには、各メトリックスのパフォーマンスを示す色分けがされています。緑は強いパフォーマンス、黄色は中程度のパフォーマンス、赤は悪いパフォーマンスです。以下は、各メトリックスの色に対応するデータの説明です。

この色分けに使用したデータの範囲は、【2019 State of DevOps Report】(https://www.devops-research.com/research.html#reports)に記載されているエリート、高、中、低パフォーマーの範囲にほぼ沿っています。

**デプロイメント頻度**

* **紫:** オンデマンド(1日に複数回デプロイする)
* グリーン:** デイリー、ウィークリー 
* 黄色：毎月
* 赤：月1回から6ヶ月に1回の間。 
    * これは "年間" として表現されます。

**Lead Time to Change**

* 紫色：1日以内
* グリーン：1週間以内
* 黄色：1週間から1ヶ月の間
* 赤色：1ヶ月以上6ヶ月未満  
**赤:** 6ヶ月より大きいもの
    * "1年 "と表現しています。

**サービス復旧までの時間**

* 紫色：1時間未満
* 緑：1日以内
* 黄色：1週間未満
* 赤色：1週間から1ヶ月の間
    * "1ヶ月 "と表現しています。
* 赤：1ヶ月以上
    * これは "1年 "として表現されます。

**変更失敗率

* 緑：15％未満
* 黄色：16%～45
* 赤色：45％以上

以下の図は、【2019 State of DevOps Report】(https://www.devops-research.com/research.html#reports)に掲載されているもので、パフォーマーのカテゴリー別に各主要指標の幅を示したものです。

![エリート、高、中、低のソフトウェアデリバリーのパフォーマーの各主要指標の範囲を示す、「State of DevOps Report」からのチャートの画像](images/dora-chart.png)です。

免責事項: これは、正式にサポートされているGoogleの製品ではありません