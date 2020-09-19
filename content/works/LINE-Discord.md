---
title: "LINEが嫌いすぎてLINE-Discord間の接続botを作った話"
date: 2020-09-13T11:51:00+09:00
description: "LINE嫌いたちの強い味方"
image: "https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/6b292cfa-cc48-0ded-d123-6e560f74f02a.png"
draft: false
weight: 2
---
<div align="right">78th KEN_RP</div>

## 注意
**この記事は技術情報共有サービス"Qiita"にて私が投稿した技術記事を部誌掲載に当たり一部改変したものです。ご了承ください。**

**元記事はこちら→[【ゼロから解説】LINEとDiscordのグループをbotで接続する【無料･高速･鯖いらず】](https://qiita.com/KENRP/items/6cd8d9ce0a93df249937)**

## 前置き

(だらだらとした文章を書くのが好きなので冗長になってしまいますがご了承を。適当に読み飛ばしてください。)

私はメッセージングアプリとして主にDiscordを使っているんですが、これがメチャクチャ使いよいんですよね。一度この「1つのサーバーに小部屋がたくさんある」形式に慣れてしまうともうLINEなんて使いにくくて戻れません。

けれど世間で一般に使われているメッセージングアプリはLINEであり、その普及率は95%を超えています[^1]。かく言う私もスマホにLINEはインストールしてあります。また、LINEをメインに使っている人たちの中には「Discord？なにそれ？慣れるのめんどそうだね笑」なんて人も多いのではないでしょうか。

そこで私が考えついた答えがこれです。
**「DiscordとLINEのトークルーム接続したらええやん」**
けれど困ったことに、私の家にはサーバーがありません。電子レンジも冷蔵庫もあるのにサーバーはありません。そこで、botをサーバーレスで構築することを目的とします。
はい。というわけで実装していきましょう。

## 実装方法
### 材料
##### Google Apps Script(GAS)
Google Drive上に置けるサーバーサイドスクリプト環境。無料なのに常時起動してpost監視とかできるので、おうちにサーバーがない人類の強い味方。言語はJavaScriptと互換性あり。

##### Glitch
Node.jsでアプリ開発ができるサービス。結構なんでもできる有能くん。これは常時起動は出来ないけどWebアプリ簡単に作れるのでよい。後述のUptime Robotで常時起動できる。

##### Discord.js
Discord botのいろいろを詰め込んだJSライブラリ。詳しくは[公式リファレンス](https://discord.js.org/#/docs/main/stable/general/welcome)見てね。

##### LINE Messaging API
LINE botのいろいろを詰め込んだAPI。公式リファレンスは[こちら](https://developers.line.biz/ja/reference/messaging-api/)。

##### Uptime Robot
~~5分ごととかの間隔で指定したアドレス宛にHTTPリクエストを送ってくれる。Glitchは5分でスリープするので、これを使って常時起動させる(ちょっと怪しいけど一応公式推奨の方法)。有料だが無料プランあり。~~
**追加情報いただきました。どうやら現在は使えない方法のようです。
役割としては「定期的にpostを投げる」だけなので、GASのトリガー機能を使って何とかできそうです。
時間ができたら書き足しますが、参考になりそうなリンクを張っておきます。
[Google Apps Script: 分単位で実行できる機能があった！ - 非IT企業に勤める中年サラリーマンのIT日記](http://pineplanter.moo.jp/non-it-salaryman/2016/05/20/google-apps-script-minute-timer/)**

### 設計
基本的にはLINE→Discordのメッセージ伝達をするDiscord上のbot(以下D-bot)と、Discord→LINEのメッセージ伝達をするLINE上のbot(以下L-bot)の2機から成っています。
##### D-bot
こいつは比較的簡単で、
①Messaging APIを使ってGASでLINEメッセージを取得する。
②DiscordのWebhookにメッセージを投げる。
これで終わります。
まあちょっと難しいところというか嵌まりやすいポイントはありますがそこまで苦戦はしないはずです。
![D-bot diagram-Page-1 (1).png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/3b5e9ac9-57cd-19c0-4e53-339a505bd5b4.png)


##### L-bot
これが結構しんどかったです。最初はGASだけで完結させる予定だったんですが、Discordの仕様のせいでよく知らないGlitchを使うハメになりました。
動作は、
(⓪GlitchをUptime Robotで常時稼働させる)
①Glitchで実装したDiscord bot(D-botとは別物)でDiscordのメッセージを取得。
②GlitchからGASへメッセージの内容をpostする。
③GASでGlitchからのpostを受け取り、Messaging APIを使ってLINEに投げる。
という感じです。D-botよりちょっと複雑ですね。
![D-bot diagram (2).png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/6b292cfa-cc48-0ded-d123-6e560f74f02a.png)


それでは次項からbot作成に入りましょう。



## LINEのbot(公式アカウント)を作る&各種設定
(以下、画像は2020/03/06現在のものです)
#### 1. アカウントを開設する
まずは[LINE Developersのページ](https://developers.line.biz/ja/)で開発者登録行います。リンクから飛んで右上のボタンからログインをしましょう。
初回ログイン時には開発者登録を求められるので、必要事項を記入して登録しましょう。
![06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/e6d2c4c8-df7b-1b05-8d0f-d5d45b41946b.png)
開発者登録が済んだら、次はプロバイダーを作成します。[開発者コンソール](https://developers.line.biz/console/)から「作成」を押して新規プロバイダーを作成します(画像は私のものなので既に1つプロバイダーが作成されています。また言語設定は右下から変更できます)。
![07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/f6208893-447c-f6a8-6fee-940fa8228b86.png)
プロバイダー名を入力してプロバイダーを作成すると、「登録されているチャネルはありません。」のメッセージと共にチャネル設定画面が出るはずです。ここで、「Messaging API」を選択して新規チャネルを作成します。
![08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/2267be29-9419-cc3b-2e8a-72f3f1b1d8c1.png)
チャネルの情報を入力する画面になるので、情報を入力していきましょう。アカウントの名前、説明は適当でいいです。業種は「個人→その他」、メアドはLINEの登録アドレスでよいでしょう。任意項目は埋めなくとも問題ありません。
![09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/5da6a007-08ca-31db-7efa-72b54160e494.png)
これでアカウント開設が完了しました！
#### 2. Tokenの取得とかMessaging APIの設定とかその他
アカウント開設が完了した後の画面の上部メニューから「Messaging API」タブを選択します。なおこのページには[LINE Developersコンソール](https://developers.line.biz/console/)からもアクセス出来ます。
![10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/e23d8021-821a-ea31-e655-2064b00cefe9.png)
Messaging APIタブの最下部にある「チャネルアクセストークン（ロングターム）」から、その右にある「発行」ボタンを押してTokenを発行します。このTokenはあとで必要になるのでどこかに控えておきましょう。
![11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/94ca51ff-aab5-49c0-55bc-a867c9462a05.png)
その上部の「グループ・複数人チャットへの参加を許可する」を「有効」、「応答メッセージ」「挨拶メッセージ」を「無効」に設定しておきましょう。Webhookは後で設定します。
これでアカウントの設定はWebhookの設定を残して終わりです！次はDiscordのWebhookを取得しに行きましょう。



## DiscordのWebhook URLを取得する
まずDiscordにアクセスします。デスクトップアプリ版でもブラウザ版でも構いません。
トーク共有をしたいサーバーで「サーバー設定→ウェブフック」に行きます(そのサーバーに対して管理者権限を持っていることを前提としています)。
![12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/a6acde0c-f272-d34b-2f5b-e9eabdceb203.png)
青色の「ウェブフックを作成」ボタンを押して共有したいチャンネルを選択し、ウェブフックURLをコピーした後に「保存」を押して終了します。なお、名前やアイコンなどの各種設定は後から変更可能です。![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/c7f3cc0a-e349-c150-b2d2-85f38d62fb8d.png)
このURLも後で使用するのでそれと分かるように控えておきましょう。
ではいよいよbotの実装に入っていきましょう。



## Google Apps Script(GAS)の利用
### GASでbot(の一部)を実装する
[Google Drive](https://drive.google.com/drive/u/0/my-drive)にアクセスして右クリックし、その他→Google Apps Scriptと進んでGASプロジェクトを作成しましょう。
コード作成画面になるので、botのコードを打ち込んでいきます。ついでに分かりやすくファイルのリネームも済ませておきましょう。

```Javascript:LineMessageSender.gs
var channel_access_token = "Channel Access Token"
var group_ID = "グループID(後で設定します)";

function doPost(e) {
  //ここで場合分けする(GlitchからもDiscordのメッセをPostするので)
  var events = JSON.parse(e.postData.contents).events;
  events.forEach(function(event) {
    if(event.type == "message") {
      sendToDiscord(event); // D-bot
    } else if(event.type == "follow") {
      follow(event);
    } else if(event.type == "unfollow") {
      unFollow(event);
    }else if(event.type == "discord") {
      sendToLine(event); // L-bot
    }
 });
}

// D-bot
function sendToDiscord(e) {
  // LINEからユーザ名を取得するためのリクエストヘッダー
  var requestHeader = {
    "headers" : {
      "Authorization" : "Bearer " + channel_access_token
    }
  };
  var userID = e.source.userId;
  var groupid_tmp = e.source.groupId;
  // LINEにユーザープロフィールリクエストを送信(返り値はJSON形式)
  var response = UrlFetchApp.fetch("https://api.line.me/v2/bot/group/"+groupid_tmp+"/member/"+userID, requestHeader);
  
  var message = e.message.text;
  // レスポンスからユーザーのディスプレイネームを抽出
  var name = JSON.parse(response.getContentText()).displayName;
  sendDiscordMessage(name, message);
  // LINEにステータスコード200を返す(これがないと動かない)
  return response.getResponseCode();
}

// L-bot
function sendToLine(e) {
  // メッセージの内容(送信先と内容)
  var message = {
    "to" : group_ID,
    "messages" : [
      {
        "type" : "text",
        "text" : e.name + "「"+e.message+"」"
      }
    ]
  };
  // LINEにpostするメッセージデータ
  var replyData = {
    "method" : "post",
    "headers" : {
      "Content-Type" : "application/json",
      "Authorization" : "Bearer " + channel_access_token
    },
    "payload" : JSON.stringify(message)
  };
  // LINEにデータを投げる
  var response = UrlFetchApp.fetch("https://api.line.me/v2/bot/message/push", replyData);
  // LINEにステータスコード200を返す
  return response.getResponseCode();
}

/* フォローされた時の処理 */
function follow(e) {
  // 今のところ空白だけど自由に実装してみると楽しい。
  // よくある「友だち追加ありがとうございます」はここで実装する。
  // 不特定多数にフォローされるような運用はしないのでスルー。
}

/* アンフォローされた時の処理 */
function unFollow(e){
  // ここでは普通こちらからもフォローを切る処理を入れる。
  // ここも不特定多数にフォローされるような運用はしないのでスルー。
}
```

```Javascript:DiscordWebhook.gs
function sendDiscordMessage(name, message) {
  var webhookURL = "Discord Webhook URL";
  // Discord webhookに投げるメッセージの内容
  var options = {
    "content" : name+" 「"+message+"」"
  };
  // データを作って投げる
  var response = UrlFetchApp.fetch(
    webhookURL,
    {
      method: "POST",
      contentType: "application/json",
      payload: JSON.stringify(options),
      muteHttpExceptions: true,
    }
  );
  // こちらはステータスコードを返す必要はない
}
```
`LineMessageSender.gs`の1行目`channel_access_token`には先ほど控えておいたLINEのチャネルアクセストークンを、`DiscordWebhook.gs`の2行目`webhookURL`には先ほど控えたDiscordのwebhook URLを入れておきましょう。
簡単に解説をすると、`LineMessageSender.gs`では`doPost()`関数でLINE/Glitch(後述)からのpostを監視します。メッセージが来たらその内容のpostが来るようになっているので、そこでLINEからのメッセージか、Discordからのメッセージか等を場合分けし、下の各種メソッドに流し込んでいます。
ちなみに`message`,`follow`,`unfollow`がLINE Messaging APIで定義されたLINEのイベントで、`discord`がこれからGlitchで定義するDiscordのメッセージイベントです。



### LINEとGASを紐付けてD-botを起動する
まずは先ほどのGASプロジェクトページにある「公開」から「ウェブアプリケーションとして公開」を選択します。
![13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/06b29779-94f9-3508-d252-51e333f272e4.png)
すると公開設定の画面が開くので、「Project version」を「New」、「Execute the app as」を「Me(メールアドレス)」、「Who has access to the app」を「Anyone, even anonymous」に設定して下の青いボタンを押します。![14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/0cfd9cbf-2983-83d3-d15d-2880b9047efb.png)
ここで嵌まりやすいポイントですが、GASコードを少しでも変更したら再度この手順を踏まないと更新が反映されません。さらに、**Project versionを毎度Newに設定しないと更新が適用されません。**そこは注意してください。

次に、先ほどのLINE DeveloperのMessaging API設定のページを開きます。[こちら](https://developers.line.biz/console/)からアクセスできます。
Webhook settingsの「Webhook URL」に先ほどのGASアプリケーションのURLを入力します。すると下部に「Use webhook」の項目が出現するので、そこをオンにしておきましょう。
![15.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/f3a21957-2e68-adfa-79b4-3697fd6242f2.png)
これでD-botの実装は完了です！お疲れ様でした！


## L-botの構築と実装
### L-botの実装、とその前に…
ここからはL-botを実装していきます。
当初はL-botもGAS上で実装する予定でした。しかし、なんとDiscordにはLINEのような「メッセージが送られたときに外部にpostする」ような機能が存在しません。嘆かわしいことです。
そこで、GASとWebhookでの実装は諦め、サーバーにbotを潜り込ませてメッセージをとってくるようにしようと思います。
(そのため、この仕組みではDiscordサーバー上に2機のbotがいます。ただしWebhookのbotはメンバー欄には表示されません)
ここで利用するのが**Glitch**です。恐らく[Discord bot サーバーなし][検索]とかやるとそこそこ出てくると思います。
では、GlitchでDiscord botを作っていきましょう。まずは…

### botのDiscordアカウントを用意する
まずは[Discord developer portal](https://discordapp.com/developers/applications/me/)にアクセスします。
左上の「New Application」を押してBotを作ってみましょう！
(画像は私のものです。ふざけたbotがいますがお気になさらず…)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/90129920-046b-635a-719f-99838906bcf0.png)
botの名前を入力してbotを作成しましょう。
したら設定画面に飛ぶので、そこの左メニューから「Bot」を選んで、
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/d068d286-3cad-f7b2-771e-c360dde411d7.png)
Add Bot!!
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/d1c44fcc-ba96-6129-4ebb-76d403203498.png)
Botを作ったら取り消せないよ～みたいなことが書いてありますが(多分)、今はそれが目的なので、Yes, do it!
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/d363db63-1e5c-f9d6-db26-8e20b4f69e20.png)
やせいの　ぼっと　が　あらわれた！！
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/6d603c37-1919-6087-ff0f-6f6a14d5cc98.png)
このTOKENというのをコピーしてどこかに控えておいてください。後ほど使います。

これでDiscord botが用意できました！
次にGlitchでbotの中身を書いていきます。



### GlitchでL-bot(Discord側)を構築する
まずはGlitchにユーザー登録します。[こちらのリンク](https://glitch.com/)からGlitchのページに飛び、右上のSign inから好きな登録方法を選び、ユーザー登録を済ませてください。
一から実装しても良いのですが、ここでは面倒なので既に作成済みのアプリケーションからコピーします。[このリンク](https://glitch.com/~pumped-chopper)にアクセスして少し下に行ったところに「Remix Your Own」をクリックしてコピーしましょう。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/5eeb7aa7-eb9f-4693-d014-d2df53475a59.png)
そうすると自動で新しいプロジェクトが作成されるので、早速コードを編集していきます。
すでに開いている`README.md`を華麗にスルーし、左のメニューから`main.js`を開いて編集していきます。ここでは、botが参加しているサーバーで発言があった場合に、その内容とユーザーの名前をGASに送信するような構成に魔改造していきます。

完成したものがこちらです。

```Javascript:main.js
// Response for Uptime Robot
const http = require("http");
http
  .createServer(function(request, response) {
    response.writeHead(200, { "Content-Type": "text/plain" });
    response.end("Discord bot is active now \n");
  })
  .listen(3000);

// Discord bot implements
const discord = require("discord.js");
const client = new discord.Client();

client.on("ready", message => {
  // botのステータス表示
  client.user.setPresence({ game: { name: "with discord.js" } });
  console.log("bot is ready!");
});

client.on("message", message => {
  // bot(自分)のメッセージには反応しない(これをしないと兵庫県警に捕まる)
  if (message.author.bot) {
    return;
  }
  // DMには応答しない
  if (message.channel.type == "dm") {
    return;
  }

  var msg = message;

  // botへのリプライは無視
  if (msg.isMemberMentioned(client.user)) {
    return;
  } else {
    //GASにメッセージを送信
    sendGAS(msg);
    return;
  }

  function sendGAS(msg) {
    // LINE Messaging API風の形式に仕立てる(GASでの場合分けが楽になるように)
    var jsonData = {
      events: [
        {
          type: "discord",
          name: message.author.username,
          message: message.content
        }
      ]
    };
    //GAS URLに送る
    post(process.env.GAS_URL, jsonData);
  }

  function post(url, data) {
    //requestモジュールを使う
    var request = require("request");
    var options = {
      uri: url,
      headers: { "Content-type": "application/json" },
      json: data,
      followAllRedirects: true
    };
    // postする
    request.post(options, function(error, response, body) {
      if (error != null) {
        msg.reply("更新に失敗しました");
        return;
      }

      var userid = response.body.userid;
      var channelid = response.body.channelid;
      var message = response.body.message;
      if (
        userid != undefined &&
        channelid != undefined &&
        message != undefined
      ) {
        var channel = client.channels.get(channelid);
        if (channel != null) {
          channel.send(message);
        }
      }
    });
  }
});

if (process.env.DISCORD_BOT_TOKEN == undefined) {
  console.log("please set ENV: DISCORD_BOT_TOKEN");
  process.exit(0);
}

client.login(process.env.DISCORD_BOT_TOKEN);
```
完成しましたがこれだけでは動きません。というかこれで動いたらちょっと怖いですね。
まだbotのtokenもGASのURLも指定していないので、これではGlitchはDiscordにもGASにもアクセス出来ません。
`.env`ファイルを編集していきます。

```shell:.env
# Environment Config

# store your secrets and config variables in here
# only invited collaborators will be able to see your .env values

# reference these in your code with process.env.SECRET

SECRET=
MADE_WITH=
DISCORD_BOT_TOKEN=[bot token]
GAS_URL=[GAS URL]

# note: .env is a shell file so there can't be spaces around =
 # Scrubbed by Glitch 2020-03-05T10:08:32+0000
```
上の`[bot token]`に先ほどコピーしたDiscord botのtokenを、`[GAS URL]`には前々項でLINE webhookに指定したGASのアプリケーションURLを指定しておきましょう。
これでコード部分は完成です。お疲れ様でした！

…しかしまだ動きません。`main.js`で使用している`request`モジュールをimportしましょう。
(私はこれが分からなくて3時間融かしました)
`package.json`を開きます。
すると上部に`Add package`メニューがあるので、そこをクリックして「request」と入力して検索をかけ、`request`モジュールをimportします。クリックするだけでimportできます。簡単ですね。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/88e7912f-c1a9-45a7-46ce-1c043aeef009.png)
ここまでできればもう一息です。ラストスパート！！



### Glitchを常時起動状態にする
これをしないとbotが5分でスリープしてしまいます。定期的にアクセスを投げるために**Uptime Robot**を使います。
まずは[Uptime Robotのトップページ](https://uptimerobot.com/)にアクセスして、右上のSign upからアカウントを作ります。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/8899ed29-83a1-3f7b-5322-5e6421e8649f.png)
プラン選択画面が出てきますが、右側のFree Planで十分です。下部のSign upをクリックしましょう。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/5dc37157-d023-6cef-a589-1f205bdd2a66.png)
あとは道なりに進めば登録は終了します。名前は偽名でOKです。
Uptime Robotにログインするとダッシュボードが開くので、そこにGlitchのURLにアクセスし続けるタイマーを仕掛けましょう。左上の「+ Add New Monitor」をクリックして、Monitor TypeのHTTP(s)を選択します。
![01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/ae9a0b1c-ae4f-e7d4-58b9-ce7d8f6cd345.png)
するといくつか項目が出現します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/210d29b4-a5fc-c5f5-3bb4-d568da8cf705.png)
「Friendly Name」にはわかりやすい名前をつけると良いでしょう。自由です。「URL」ですが、`https://{Glitchプロジェクト名}.glitch.me/`という形式のURLを入力します。
ここでいうGlitchプロジェクト名とは、自動で生成されたプロジェクト名のことです。
自分でコピーしたプロジェクトを見ると、`[英単語]-[英単語]`(もしくは`[英単語]-[英単語]-[英単語]`)という形式の名前がついていると思います。それがプロジェクト名です。
最後に、「Select "Alert Contacts To Notify"」のチェックを登録したメールアドレスにつけておきましょう。何か異常があれば、このメールアドレスに連絡が届きます(このメールアドレスは変更可能ですが少なくとも1つは必要です)。
他の部分は特に変更せずに、そのまま画面下の「Create Monitor」をクリックしましょう。これでUptime Robotの設定は終了です。お疲れ様でした！！



### L-botを起動する
あと1ステップです。頑張りましょう。
ここまできてもまだL-botは起動出来ません。LINEのgroup IDを設定していませんでしたね。
ここを設定していきましょう。
(かなりスマートじゃない手段を使って設定します。他に方法があればコメントなりTwitterなりで教えてください)
先ほどのGASコードを編集します。具体的には、`LineMessageSender.gs`の30行目を、

```Javascript:LineMessageSender.gs
var message = e.message.text;
```
を

```Javascript:LineMessageSender.gs
var message = e.message.text + groupid_tmp;
```
と一時的に変更してデバッグ出力みたいなことをします。
この状態にすればD-botはもう稼働しているので、LINE botに何か発言をすればLINEのgroup IDが含まれたメッセージがDiscordのWebhookチャンネルに送られてくるはずです。
![02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/ceebab67-092c-06f8-7597-aabde9bf5d88.png)
(画像では転送が成功した瞬間なので私が狂喜しています)
こうしてLINEのgroup IDが取得出来たら、それをGASの`LineMessageSender.gs`2行目の`group_ID`に設定しておきましょう。それとデバッグ出力を切るのも忘れずに。
これでプログラムは完成です！！



### L-bot(Discord側)をDiscordサーバーに配置する
最後のステップです。
[Discord developer portal](https://discordapp.com/developers/applications)に戻り、今度はSettingsの「OAuth2」を選択します。
すると設定画面に移行するので、その下の「SCOPES」で「bot」を選択します。
![03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/45dd5993-b9cb-dc69-121c-9b7e91480bb4.png)
次に権限を設定します。
画面下部に現れた「BOT PERMISSIONS」から「Administrator」を選択しておきましょう。
チェックを入れ終わったら表示されているURLをコピーし、そのURLにアクセスします。
するとサーバー選択画面になるので、追加したいサーバーを選択して「はい」を選択します。
![04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/e9646cdf-ddf9-068a-b955-c8a4629cb53a.png)
これでL-botの起動が完了しました！



## 完成
やった！！！完成！！！！！！！！！
こんな感じで動作します。
![ESZZercXYAcx7bW.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/bcbe3be8-ed33-ddb6-b4a1-d8972eb3a526.jpeg)
![ESZ4FHVUEAk2LHs.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/554644/2f8d396b-1408-d7cf-1d12-82eef4edbd81.jpeg)

## 最後に
お疲れ様でした。
結構細かいトラップがたくさん敷き詰めてあったイメージでした。暇があればトラップ集みたいなものも作って記事にしてみたいと思います。

それと、勘の良い方ならお気づきかもしれませんが、このままでは画像の送信はできません。画像の送信については「読者への演習問題とする」としておきます。ちょっとめんどくさいですができます。

また、このままではDiscordのメッセージは送信されたチャンネルに関係なく全てLINEへ転送されてしまいます。そういった動作仕様ですが、この動作が嫌いな方はぜひ自分で実装してみてください。ここまでの文章を理解できたあなたなら苦も無く実装できることでしょう。

## ちなみに
Slackだとwebhookからメッセージを取得出来るのでもうちょい楽らしいです。

## 参考文献
大変お世話になりましたm(_ _)m

[LINEのBot開発 超入門（前編） ゼロから応答ができるまで](https://qiita.com/nkjm/items/38808bbc97d6927837cd)

[Google Apps ScriptでPostする](https://qiita.com/\_y\_s\_k\_w/items/95b29a8a72bb97768256)

[DiscordにWebhookでいろいろ投稿する](https://qiita.com/Eai/items/1165d08dce9f183eac74)

[GASでDiscord botを作ろうと奮闘した話](http://sindouaika.blogspot.com/2018/02/gasdiscord-bot.html)

[簡単なDiscord Botの作り方（初心者向け）](https://note.com/bami55/n/ncc3a68652697)

[自動応答のDiscord Botを完全無料で構築する[gas][glitch]](https://qiita.com/yuzuki\_/items/23935e4a49035d313e5e)


[^1]: App Apeのデータに拠る。2020/03/06閲覧。
