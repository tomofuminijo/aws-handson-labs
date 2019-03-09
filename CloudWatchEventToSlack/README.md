# CloudWatch Events からSlack にメッセージを飛ばすLambda 関数を作成する

このハンズオンでは、CloudWatch Events を利用して任意のイベントが発生したら、Slack にメッセージを飛ばすLambda 関数を作成します。

## 作成するアーキテクチャ

作成するのは以下のような内容です。
 
 EC2 のstate change (起動・停止・終了など) -> CloudWatch Events -> Lambda 関数 -> Slack

 ## AWS 料金
- 最低で 0.24 USD
  - KMS のカスタムキー(CMK) が必要となり、1 USD / Month の料金となります。従量課金ですので、削除すれば時間割でそれ以上は料金はかかりませんが、削除は最短で7 日かかりますので、最低7日分 ≒ 0.24 USD かかります。


## Slack の準備
1. Slack でワークスペースを準備します。(既存のワークスペースでも可能です。)  
   以下のURL にアクセスすることでワークスペースの作成が可能です。  
   https://slack.com/create

2. サークスペース内に、利用するチャンネルを用意します。(既存のものでも構いません)
3. 以下のURL にアクセスします。  
   https://<your-workspace>.slack.com/apps
4. アプリの検索で `Incoming Webhook` を検索します。
5. **チャンネルへの投稿** でチャンネルを選択し、**Incoming Webhook インテグレーションの追加** ボタンをクリックします。
6. **Webhook URL** をコピーしておきます。後でLambda 関数に設定します。

## KMS カスタマー管理キーの作成

上記で用意したWebHook URLが漏れてしまうと、誰でもメッセージを投げることができてしまうため、Lambda ファンクションには暗号化して保存しておきます。その際にカスタマー管理キーが必要になりますので作成しておきます。

1. マネジメントコンソールにログインします。

2. **サービス** -> **KMS** を選択します。
3. 画面左の **ナビゲーションペイン** で **カスタマー管理型のキー** をクリックします。
4. **カスタマー管理型のキー** にて、**キーの作成** をクリックします。
5. **エイリアスと説明の追加** 画面にて、以下の内容を入力します。
    - エイリアス: `lambda/cloudwatcheventstoslack` or 任意の名前
    - 説明: 任意
6. **Next** ボタンをクリックします。
7. **タグの追加** は任意で入力し、**Next** をクリックします。（不要であれば何も入力せずに **Next** をクリック）
8. **キー管理者** で必要な管理者を選択します。(IAM ユーザ or IAM ロール)
9. **Next** ボタンをクリックします。
10. **キーの使用アクセス許可を定義** は何も設定せずに、**Next** をクリックします。(Lambda で利用しますが、Lambda 側に設定するIAM ロールで使用許可します。)
11. **キーポリシーの表示と編集** 画面で **完了** をクリックします。
12. **カスタマー管理型のキー** 画面の一覧に作成したキーが表示されていることを確認します。

## Lambda 関数の作成

今回は、AWS で予め用意されているLambda 関数の**設計図** (予め用意されているサンプルコードと設定のプリセット) を利用します。

1. マネジメントコンソールにログインします。
   
2. **サービス** -> **Lambda** をクリックします。
3. **Lambda** サービス画面にて、**関数の作成** ボタンをクリックします。
4. **関数の作成** 画面にて、**設計図の使用** をクリックします。
5. **検索** フィールドに、**slack** を入力してEnter をクリックします。
6. **cloudwatch-alarm-to-slack-python3** を選択します。
7. **基本的な情報** 部分にて以下の内容を入力します。
   - 関数名: `CloudWatchEventsToSlack`  or 任意の名称
   - 実行ロール: **AWS ポリシーテンプレートから新しいロールを作成** (デフォルト)
   - ロール名: `CloudWatchEventsToSlackRole` or 任意の名前
   - ポリシーテンプレート: 特に何も入力せず
   - SNS トリガー: **削除** ボタンをクリック (今回は使用しないため)
8. **Lambda 関数のコード** 部分 にて以下の内容を入力します。
   - ソースコード部分はそのまま (あとで変更)
   - 環境変数: 
     - slackeChannel には 先ほど作成したチャネル名を入力(`#cloudwatch`など)
     - kmsEncryptedHookUrl には、先程コピーした **Webhook URL** の **https** を取り除いた部分を入力
   - **暗号化の設定** を展開して以下を入力
     - **伝送中の暗号化のためのヘルパーの有効化** にチェック
     - **伝送中に暗号化する AWS KMS キー** にて、**lambda/cloudwatcheventstoslack** を選択
     - **保管時に暗号化する AWS KMS キー** は **(デフォルト) aws/lambda** のまま
   - 環境変数の **kmsEncryptedHookUrl** の **暗号化** ボタンをクリックして暗号化する
8. **関数の作成** ボタンをクリックします。
9. **CloudWatchEventsToSlack** (or 任意に設定した名前)画面に遷移します。
10. 画面中央の **関数コード** 内の`def lambda_handler(event, context): ` メソッド部分のコードを下記に書き換えます。

    ```python
    def lambda_handler(event, context):
    logger.info("Event: " + str(event))
    slack_message = {
        'channel': SLACK_CHANNEL,
        'text': str(event)
    }

    req = Request(HOOK_URL, json.dumps(slack_message).encode('utf-8'))
    try:
        response = urlopen(req)
        response.read()
        logger.info("Message posted to %s", slack_message['channel'])
    except HTTPError as e:
        logger.error("Request failed: %d %s", e.code, e.reason)
    except URLError as e:
        logger.error("Server connection failed: %s", e.reason)
    ```

    今回選択した設計は、CloudWatch Alarm からSNS 経由でイベントを取得する実装なので、その部分を書き換えました。上記処理は単純に引数でもらったevent ログに出した後に、そのままSlack に投げているだけです。Slack には json 形式のイベント情報がそのまま表示されます。もし人が読みやすいようにメッセージを整形する場合は上記コードを書き換えることで実現できます。

11. 画面右上の **保存** ボタンをクリックします。

## Lambda 関数のテスト

1. **保存** ボタンの左の **テスト** ボタンをクリックします。
   
2. **テストイベントの設定** 画面が表示されます。
3. **イベント名** に `CloudWatchEvent` と入力して、その下の `JSON` 部分を以下で書き換えます。
   
   ```json
    {
        "version": "0", 
        "id": "12488fb8-73a8-d016-5dcc-79de03adc621", 
        "detail-type": "EC2 Instance State-change Notification", 
        "source": "aws.ec2", 
        "account": "12345678901234", 
        "time": "2019-03-04T07: 59: 22Z", 
        "region": "ap-northeast-1", 
        "resources": [
            "arn:aws:ec2:ap-northeast-1:12345678901234:instance/i-xxxxxxxxxxxxxxx"
            ], 
        "detail": {
            "instance-id": "i-xxxxxxxxxxxxxxx", 
            "state": "pending"
        }
    }
   ```
4. **作成** ボタンをクリックします。

5. もう一度、**テスト** ボタンをクリックします。
6. **実行結果: 成功(ログ)** と表示されて、Slack にメッセージが飛んでいることを確認します。  
    ***Note:*** メッセージが届かない場合: グ出力内に **Server connection failed: [Errno -2] Name or service not known** と出力されている箇所があったら、**kmsEncryptedHookUrl** の設定が間違っていないか確認してください。特に頭の `https` は不要なので、ついていないことを確認します。


    以上で、Lambda の作成は完了です。

## CloudWatch Events の設定

今回は、EC2 インスタンスのState 変化に応じてSlack にメッセージが飛ぶように設定します。

1. **サービス** から **CloudWatch Events** を選択します。
   
2. **ナビゲーションペイン** で、**ルール** をクリックします。
3. **ルール** 画面にて、以下の**ルールの作成** ボタンをクリックします。
4. **ステップ1: ルールの作成** の**イベントソース** にて、以下のように入力します。
   - **イベントパターン** を選択（デフォルトで選択状態)
   - **サービス名** で `EC2` を入力
   - **イベントタイプ** で、`EC2 Instance State-change Notification` を選択
   - その他はデフォルトのまま
5. **ターゲット** にて、**ターゲットの追加** をクリックして以下のように入力します。
   - **Lambda 関数** を選択
   - **関数の選択** にて、`CloudWatchEventsToSlack` or 任意に設定したLambda 関数名 を選択
6. 画面右下の **設定の詳細** ボタンをクリックします。
7. **ルールの定義** 画面にて、**名前** に `EC2StateChangeRule` (or 任意の名前) を入力して、**ルールの作成** ボタンをクリックします。
8. **ルール EC2StateChangeRule が作成されました。** と表示されたことを確認します。


## テストしてみる

今回はEC2 のState Change イベントに反応しますので、EC2 の起動操作や停止操作などを実施してみてください。操作方法は割愛します。
Slack にメッセージが届くことを確認してください。

## 後片付け

作成したものをすべて削除したい場合は以下の手順で実施してください。

### CloudWatch Events Rule の削除

1. CloduWatch サービス画面の**ナビゲーションペイン** にて**ルール** を選択
2. **EC2StateChangeRule** を選択して、**アクション** > **削除**

### Lambda 関数の削除

1. Lambda サービスを選択
2. 関数の一覧で、**CloudWatchEventsToSlack** を選択して、**アクション** > **削除**

### IAM ロールの削除

1. IAM サービスを選択
2. **ナビゲーションペイン** にて、**ロール** を選択
3. **CloudWatchEventsToSlackRole** を検索し選択肢、**削除** ボタンをクリック
4. **ロールの削除** 画面にて、**はい、削除します** ボタンをクリック

### CMS の削除

1. KSM サービスを選択
2. **ナビゲーションペイン** にて、**カスタマー管理型のキー** をクリック
3. **lambda/cloudwatcheventstoslack** を選択して、**キーのアクション** > **キーの削除をスケジュール** をクリック
4. **キーの削除をスケジュール** ダイアログにて、**待機期間(日数)** に `7` を入力
5. ダイアログ下の、**7 日でこのキーを削除することを確認します。** にチェックを入れて **削除をスケジュールする** ボタンをクリック
6. **カスタマー管理型のキー** 画面にて、対象キーのステータスが**保留中の削除** となっていることを確認(7日後に削除されます)

## CloudWatch Logs の削除

1. CloduWatch サービス画面の**ナビゲーションペイン** にて**ログ** を選択
2. **/aws/lambda/CloudWatchEventsToSlack** にチェックを入れて、**アクション** > **ロググループの削除** をクリック
3. **はい、削除します** をクリック

以上です。
