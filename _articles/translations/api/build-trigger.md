注意: `_www_`は現在非推奨のエンドポイントとなります。代わりに`_https://app.bitrise.io/app/APP-SLUG/build/start.json_`を使ってください。

Build Trigger APIを使えば簡単なAPIの呼び出しで新しいビルドを作ることができます。

`branch`, `tag`もしくは _git commit_ のようなパラメーターの定義をすることができ、また詳細ページに載せる _build message_ も定義することができます。

### 対話的なcURLコマンド設定機能

[bitrise.io](https://www.bitrise.io)のアプリ画面の `Start/Schedule a build` ボタンをクリックし、ポップアップの中にある `Advanced`タブに切り替えると下の方に「対話的なcURLコマンド設定機能」があります。ポップアップの中で指定したパラメーターをベースにしたcurlコマンドが表示されています。

---

## Trigger APIを呼び出してビルドを始めるやり方

JSONデータと一緒に `POST`リクエストでビルドトリガーを呼ばなければいけません。

### Build Trigger Token and App Slug

Bitrise Trigger APIを使うには`Build Trigger Token` と `App Slug`指定する必要があります。この二つはどちらともアプリ画面の`Code`タブから好きなタイミングで作り直すことができます。

---

### 古いAPIトークンのパラメーターについて

古い`_api_token_`パラメーターは非推奨です。`_build_trigger_token_`パラメーターを代わりに使ってください。

---

## JSONデータ

JSONデータには少なくとも以下のものが含まれている必要があります。

* `hook_info`というオブジェクト
  * `type`キーに bitriseの値
  * `build_trigger_token`キーに _Build Trigger Token_の値
* `build_param`というオブジェクト. `tag`, `branch`, もしくは `workflow_id`パラメーターのいずれかが少なくとも指定されている必要があります。

`branch`に_master_を指定した場合の最小のJSONデータのサンプルは以下のようになります。

    {
      "hook_info": {
        "type": "bitrise",
        "build_trigger_token": "..."
      },
      "build_params": {
        "branch": "master"
      }
    }

**To pass this JSON payload** you can either pass it as the **body** of the request **as string** (the JSON object serialized to string),
or if you want to pass it as an object (e.g. if you want to call it from JavaScript) then you have to include a root `payload`
element, or set the JSON object as the value of the `payload` POST parameter.

jQueryを用いた`payload`パラメーターの使用例は以下の通りです。

    $.post("https://app.bitrise.io/app/APP-SLUG/build/start.json", {
        "payload":{
            "hook_info":{
                "type":"bitrise",
                "build_trigger_token":"APP-API-TOKEN"
            },
            "build_params":{
                "branch":"master"
            }
        }
    })

## ビルドパラメーター

以下のパラメーターが `build_params` オブジェクトの中でサポートされています。

### Gitに関係するもの:

* `branch`(文字列): ビルドする元となるブランチ。In case of a standard git commit this is the branch of the commit.
  In case of a Pull Request build this is the source branch, the one the PR was started from.
* `tag`(文字列): ビルドするためのgitのタグ
* `commit_hash`(文字列): gitのcommit hash値
* `commit_message` (文字列): gitのコミットメッセージ(もしくはビルドメッセージ)

### Bitrise.ioに関係するもの

* `workflow_id`: (文字列): workflowIDを指定して強制することができます。もし定義されていない場合はプロジェクトの[Trigger Map config](/webhooks/trigger-map/)設定が使われます)
* `environments` (配列もしくはオブジェクト): `environments`についてのより多くの情報は[Specify Environment Variables](#specify-environment-variables)を見てください。
* `skip_git_status_report` (真偽値): gitへビルドの状態を伝えることをスキップすることができます

### Pull Requestに関係するもの:

* `branch_dest` (文字列): Pull Requestビルドの場合のみ使います。Pull Requestをマージする対象を指定します。例: `master`
* `pull_request_id` (数値): ソースコードのホスティングサービスでのPull Request IDを指定します (例: GithubのPRナンバー)
* `pull_request_repository_url` (文字列): Pull Requestがどこから送られてきたのかを示す repository url を指定します.
  例: forされたレポジトリであればforkしたURLを指定してください。例: `https://github.com/xyz/bitrise.git`.
* `pull_request_merge_branch` (string): the pre-merge branch, **if the source code hosting system supports & provides**
  the pre-merged state of the PR on a special "merge branch" (ref). おそらくGithubだけがサポートしています。
  例: `pull/12/merge`.
* `pull_request_head_branch` (string): the Pull Request's "head branch" (`refs/`) **if the source code hosting system supports & provides** this.
  This special git `ref` should point to the **source** of the Pull Request. Supported by GitHub and GitLab.
  Example: `pull/12/head` (github) / `merge-requests/12/head` (gitlab).

### 特定の環境変数

追加の _環境変数_ を定義することができます。

これらの環境変数は`_Secrets_` _と  `_App Env Vars_`の間で扱われます。
ということでビルド設定に関係する変数の定義を上書きすることはできません。(例: App Env Vars), Secretsだけ上書きすることができます。
より詳しくは以下を参照してください。
_[_Availability order of environment variables_](/bitrise-cli/most-important-concepts/#availability-order-of-environment-variables)

これらのパラメーターは**オブジェクトの配列**でなければいけません。
そして全ての配列の中のオブジェクトには`mapped_to`というキーが少なくとも必要となります。(環境変数のキーに(`$`)という記号を使うことはできません)
`value`キーも必要です(変数の値)。

デフォルトの環境変数の名前が含まれている場合は値を指定されたものに置き換えようとします。
この動作を無効にしたい場合は`is_expand`フラグを`false`にしてください。

例:

    "environments":[
      {"mapped_to":"API_TEST_ENV","value":"This is the test value","is_expand":true},
      {"mapped_to":"HELP_ENV","value":"$HOME variable contains user's home directory path","is_expand":false},
    ]

### Workflow to be used for the build

アプリの[Trigger Map](/webhooks/trigger-map/)
By default the Workflow for your Build will be selected based on the
`build_params` and your app's [Trigger Map](/webhooks/trigger-map/).
This is the same as how [Webhooks](/webhooks/) select the workflow for the build
automatically (based on the _Trigger Map_), and how you can
define separate Workflows for separate branches, tags or pull requests
without the need to specify the workflow manually for every build.

Trigger APIはこの選択を上書きすることができ、指定したワークフローを使うこともできます。

`build_params`に`workflow_id`パラメーターを加えるだけでよく、ビルドの際に使いたいワークフローを指定してください。

`branch`と`workflow_id`を指定した`build_params`の例:

    "build_params":{"branch":"master","workflow_id":"deploy"}'

## `curl` example generator

`Start/Schedule a build` ボタンをクリックし、ポップアップの中にある `Advanced`タブに切り替えると下の方に「対話的なcURLコマンド設定機能」があります。ポップアップの中で指定したパラメーターをベースにしたcurlコマンドが表示されています。

基本的なcurlの呼び出しは以下のようになります(`branch`キーに`master`が指定されています)。

    curl -H 'Content-Type: application/json' https://app.bitrise.io/app/APP-SLUG/build/start.json --data '{"hook_info":{"type":"bitrise","build_trigger_token":"APP-API-TOKEN"},"build_params":{"branch":"master"}}'

メモ: `_Content-Type_`ヘッダーに`_application/json_`という値を加えるのを忘れないでください。

より進んだ例: `deployment`ワークフローを使って**master**ブランチをビルドし、ビルドメッセージ(`commit_message`)を指定します。そしてテスト用の環境変数を設定します(`API_TEST_ENV`)
このコマンドは以下のようになります。

    curl  -H 'Content-Type: application/json' https://app.bitrise.io/app/APP-SLUG/build/start.json --data '{"hook_info":{"type":"bitrise","build_trigger_token":"APP-API-TOKEN"},"build_params":{"branch":"master","commit_message":"Environment in API params test","workflow_id":"deployment","environments":[{"mapped_to":"API_TEST_ENV","value":"This is the test value","is_expand":true}]}}'
