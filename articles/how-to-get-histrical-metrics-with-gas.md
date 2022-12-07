---
title: "GASでAmazon Connectの履歴メトリクスを取得する方法"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GAS", "AmazonConnect", "GetMetricDataAPI"]
published: false
---

## **はじめに**

都内のWebベンチャー企業でインターンをしているbondoです。今回は、Amazon Connectの履歴メトリクスをGASで取得する方法をご紹介します。そもそも、「他にもっといい方法があるのでは？」といった事や「どんな箇所で躓きやすいのか？」といった観点にも触れているので、業務等でご活用できる記事になっているかと思います。

## 背景・目的

クライアント業務で電話対応を行った際に、従業員のパフォーマンスを計りたい！しかし、その為には、Amazon Connectの画面から履歴メトリクスをCSV形式でダウンロードし、その後、集計する必要があり、かなり大変。

更に補足として、画面からは30分毎であれば、3日分。1日毎であれば、1ヶ月分しか一度にダウンロードできない為、「30分単位でのパフォーマンスを解析したい！」となった時には、相当な労力が必要となる。

### 実現したい事

- 30分間隔で過去半年分の**履歴メトリクス＞電話番号**タイプのメトリクスを取得する

- 日別間隔で過去半年分の**履歴メトリクス＞電話番号**タイプのメトリクスを取得する

- 30分間隔で過去半年分の**履歴メトリクス＞エージェント**タイプのメトリクスを取得する

## 結論

結論から言うと、上記実現したい事を行う為には、下記以外に良い方法がなかったという結論になります。

1. Amazon Connectの履歴メトリクスをスケジューリングしてS3に吐き出す

2. GASからS3にアップしたCSVファイルのオブジェクトを取得する

3. スプレッドシートに吐き出して集計

正直、GetMetricData APIとかでさくっと取ってこれるイメージだったんですが、ところがどっこいって感じでした。つまり、今すぐに上記「実現したい事」に示した条件の履歴メトリクスデータうぃ取得したい場合は、手動で頑張るしかありません。なので、これから説明する方法は、「未来に向けて作業を楽にする」という視点で読んでい頂けると良いかと思います。

色々と試した結果を下記で解説しているので、ご参考になれば幸いです。

## 履歴メトリクスの取得方法

### ① GetMetricData APIを使う

おそらく、調べると一番よくでてくるのが[GetMetricData APIを使って取得する](https://dev.classmethod.jp/articles/amazon-connect-getmetricdata/)方法であると感じている。

GASで実装するとすると、以下のようになる。（htmlとgs間でObjectをやり取りしている）

```html
<!DOCTYPE html>
<html>
  <head>
    <base target="_top">
    <style>
      .wrapper{
        display:flex;
        justify-content:center;
        flex-direction:column;
        align-items:center;
      }
      button{
        font-weight:bold; 
        padding:5px 20px; 
        font-size:16px;
        background:#000;
        color:#fff;
        transition:all 0.2s;
        cursor:pointer;
        border-radius:5px;
        box-shadow:2px 2px 0px #333;
        margin-bottom:15px;
      }
      button:hover{
        opacity:0.8;
      }
      button:active{
        box-shadow:none;
        transform:translate(2px,2px);
      }
      #status{
        text-align:center;
        font-size:15px;
        font-weight:bold;
      }
      p{
        font-size:11px;
      }
      span{
        color:#880000;
        font-weight:bold;
        font-size:11px;
      }
      #result{
        text-align:center;
        margin-top:10px;
        font-weight:bold;
        font-size:11px;
      }
    </style>
  </head>
  <body>
    <div class="wrapper">
    <button id="btn">履歴メトリクスの取得</button>
    </div>
    <div id="status"></div>
    <div id="result"></div>
  </body>
  <script src="https://sdk.amazonaws.com/js/aws-sdk-2.1219.0.min.js"></script>
  <script
  src="https://code.jquery.com/jquery-3.2.1.min.js"
  integrity="sha256-hwg4gsxgFZhOsEEamdOYGBf13FyQuiTwlAQgxVSNgt4="
  crossorigin="anonymous"></script>
  <script type="text/javascript">
  $(function()
  {
    $("#btn").on('click', function(){
      // ボタンの非活性化
      $("#status").html("取得開始");
      $("#btn").prop("disabled",true);
      if($("#btn").prop("disabled",true)){
        $("#btn").css({
          "background":"#eee",
          "color":"#333",
          "box-shadow":"none",
          "transform":"translate(2px,2px)",
          "cursor":"not-allowed"
        });
      }

      // getSpreadsheetData SuccessHandler
      function onSuccess(body){
        AWS.config.region = 'ap-northeast-1';
        AWS.config.update({
          accessKeyId: "AccessKeyId",
          secretAccessKey: "SecretAccessKey",
        });
        // Create GetMetricData object
        const connect = new AWS.Connect();
        const historicalMetrics = [
          {Name: "ABANDON_TIME",Unit: "SECONDS",Statistic: "AVG"},  // 平均キュー中止時間
          {Name: "AFTER_CONTACT_WORK_TIME",Unit: "SECONDS",Statistic: "AVG"},  // 連絡作業後の時間
          {Name: "API_CONTACTS_HANDLED",Unit: "COUNT",Statistic: "SUM"},  // 対応したAPIのお問い合わせ
          {Name: "CALLBACK_CONTACTS_HANDLED",Unit: "COUNT",Statistic: "SUM"},  // 対応した問い合わせのコールバック
          {Name: "CONTACTS_ABANDONED",Unit: "COUNT",Statistic: "SUM"},  // 中止された問い合わせ
          {Name: "CONTACTS_AGENT_HUNG_UP_FIRST",Unit: "COUNT",Statistic: "SUM"},  // エージェントが先に切断した問い合わせ
          {Name: "CONTACTS_CONSULTED",Unit: "COUNT",Statistic: "SUM"},  // 相談した問い合わせ
          {Name: "CONTACTS_HANDLED",Unit: "COUNT",Statistic: "SUM"},  // 対応した問い合わせ
          {Name: "CONTACTS_HANDLED_INCOMING",Unit: "COUNT",Statistic: "SUM"},  // 対応した着信問い合わせ
          {Name: "CONTACTS_HANDLED_OUTBOUND",Unit: "COUNT",Statistic: "SUM"},  // 対応した発信問い合わせ
          {Name: "CONTACTS_HOLD_ABANDONS",Unit: "COUNT",Statistic: "SUM"},  // 保留中に顧客が切断した問い合わせ
          {Name: "CONTACTS_MISSED",Unit: "COUNT",Statistic: "SUM"},  // 問い合わせの不在着信
          {Name: "CONTACTS_QUEUED",Unit: "COUNT",Statistic: "SUM"},  // キューに保存された問い合わせ
          {Name: "CONTACTS_TRANSFERRED_IN",Unit: "COUNT",Statistic: "SUM"},  // 内部転送された問い合わせ
          {Name: "CONTACTS_TRANSFERRED_IN_FROM_QUEUE",Unit: "COUNT",Statistic: "SUM"},  // キューから転送された問い合わせ
          {Name: "CONTACTS_TRANSFERRED_OUT",Unit: "COUNT",Statistic: "SUM"},  // 外部転送された問い合わせ
          {Name: "CONTACTS_TRANSFERRED_OUT_FROM_QUEUE",Unit: "COUNT",Statistic: "SUM"},  // キューから転送された問い合わせ
          {Name: "HANDLE_TIME",Unit: "SECONDS",Statistic: "AVG"},  // 平均処理時間
          {Name: "HOLD_TIME",Unit: "SECONDS",Statistic: "AVG"},  // 顧客の平均保留時間
          {Name: "INTERACTION_AND_HOLD_TIME",Unit: "SECONDS",Statistic: "AVG"},  // エージェントの応答時間と顧客の保留時間
          {Name: "INTERACTION_TIME",Unit: "SECONDS",Statistic: "AVG"},  // エージェントの平均対応時間
          {Name: "OCCUPANCY",Unit: "PERCENT",Statistic: "AVG"},  // 利用率
          {Name: "QUEUE_ANSWER_TIME",Unit: "SECONDS",Statistic: "AVG"},  // 平均キュー応答時間
          {Name: "QUEUED_TIME",Unit: "SECONDS",Statistic: "MAX"},  // キューに入っている最大時間
          {Name: "SERVICE_LEVEL",Unit: "PERCENT",Statistic: "AVG", Threshold: { Comparison: "LT",ThresholdValue: 60}},  // サービスレベルX
        ];

        let starttime = new Date(2022,9-1,20,9,00,0);
        let endtime = new Date(2022,9-1,20,22,30,0);

        let params = {
          EndTime: endtime,
          Filters: { /* required */
            Queues: [
              "QueueId",
              /* more items */
            ]
          },
          HistoricalMetrics: historicalMetrics, /* required */
          InstanceId: "InstanceId", /* required */
          StartTime: starttime,
          Groupings: [  // Queue or Channel
            "QUEUE"
          ],
        };

        connect.getMetricData(params, function(err, data){
          if(err){
            alert(JSON.stringify(err));
          }else{
            alert(JSON.stringify(data.MetricResults[0]));
          }
        });
      }

      // getSpreadsheetData FailureHandler
      function onFailure(e){
        console.log(e, 'getSpreadsheetData()');
      } 
      // End of onFailure()
      google.script.run.withSuccessHandler(onSuccess).withFailureHandler(onFailure).getSpreadsheetData();
    });
  });
  </script>
</html>
```

```js
const sheet_name = "GetMetricData";

// スプレッドシートデータの取得
const app = SpreadsheetApp;
const ss = app.getActiveSpreadsheet();
const sheet = ss.getSheetByName(sheet_name);
const setting_sheet = ss.getSheetByName(settings_name);

// 実行ボタン表示
function GetMetricData(e){
  const html = HtmlService.createHtmlOutputFromFile('btn').setWidth(240).setHeight(200);
  app.getActiveSpreadsheet().show(html);
}

function getSpreadsheetData(){
  return true;
}
```

しかし、GetMetricData APIを用いると主に2つの課題がある。

1. 履歴メトリクスの種類としてキューとチャネルしか選べない。つまり、エージェントや電話番号単位で出力が出来ない。
2. 24時間以内の履歴しか取得できない。これを超えると、```InvalidParameterException```のエラーとなってしまう。

以上の理由により、冒頭で示した目的には沿わなく、使いものにならないという事でこちらの方法は没となった。

:::message

上記2点についてはAWS Supportにも事実確認をしておりますので、2022/12時点では確実な情報かと思います。
ちなみに、エージェント別の履歴メトリクスに関しては、[イベントストリーム]([Amazon Connect エージェントのイベントストリーム - Amazon Connect](https://docs.aws.amazon.com/ja_jp/connect/latest/adminguide/agent-event-streams.html))をうまく使うと良いみたいです。

:::

### ② 履歴メトリクスのスケジューリング出力を使う

これから紹介する方法は、過去分（構築時点から換算して）の履歴メトリクスを出力する方法ではないですが、未来に向けてシステムを構築して履歴メトリクスの取得を簡単にしよう！というコンセプトの場合の方法になってます。



## **実装方法**

### ① S3ライブラリを使用する

### ② S3ライブラリを使用する
