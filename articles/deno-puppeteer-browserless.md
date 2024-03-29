---
title: "browserlessで最速スクレイピングをする with Deno Deploy"
emoji: "🧑🏼‍💻"
type: "tech"
topics: ["deno", "puppeteer", "browserless"]
published: true
publication_name: "ispec_inc"
---

# スクレイピングはまだまだめんどくさい

以前にこんな記事を書きましたが、
https://zenn.dev/ispec_inc/articles/lambda-puppeteer

これでも手順が多すぎてしんどいです、、

サーバレスにブラウザを操作したいだけなのに！！

そこで[ browserless ](https://www.browserless.io) というSaasを使ってchromiumの設定不要でスクレイピングをする方法をご紹介します


# browserless

## なんぞ？
[![Image from Gyazo](https://i.gyazo.com/243ec95fed8073a86379373d672a712f.png)](https://gyazo.com/243ec95fed8073a86379373d672a712f)

browserlessはヘッドレスchromiumのAPIを提供してくれるサービスです

つまりchromiumをインストールしなくてもpuppeteerを起動できちゃうんです！

LPのPlayGroundで好きに実行できるので試してみてください

↓ みたいな感じでブラウザ起動時の`browserWSEndpoint` オプションでウェブソケットのURLを指定するだけで利用できます(コードを動かすサーバーは自分で用意する必要があります)

```js
const browser = await puppeteer.connect({
  browserWSEndpoint: `wss://chrome.browserless.io?token=${Deno.env.get("BROWSERLESS_TOKEN")}`,
});
```
## 料金

[Introducing Unit-based Pricing](https://www.browserless.io/blog/2022/12/08/unit-based-pricing/)

Unitベースの課金でpuppeteerなら28秒が1unitとなり、1000unitまでは無料なので、個人の利用なら無料で十分に活用できると思います



# 実用編

## 外形監視

Deno Deployでやってみます

弊社のホームページのスクショを定期的にslackに投稿する例です

Deno DeployにCronが乗ったのでまじで以下のファイルをpushするだけで動きます(もちろんbrowserlessとslack webhookの設定は必要です)

Deno Deployはファイル書き込みができないのですが、@slack/web-apiの画像アップロードAPIがBufferも受け付けてくれるのでありがたく使います


参考

- Announcing Deno Cron(https://deno.com/blog/cron)
- node: specifiers (https://docs.deno.com/runtime/manual/node/node_specifiers)

```js
// 環境変数に BROWSERLESS_TOKEN, SLACK_TOKENをセットしています
import puppeteer from "https://deno.land/x/puppeteer@16.2.0/mod.ts";
import { WebClient } from "npm:@slack/web-api";
import { Buffer } from 'node:buffer';

const main = async () => {
    const browserlessToken = Deno.env.get("BROWSERLESS_TOKEN");
    const slackToken = Deno.env.get("SLACK_TOKEN");
    if (!browserlessToken || !slackToken) {
        throw "[ERROR] `BROWSERLESS_TOKEN` and `SLACK_TOKEN` is required"
    }
    const browser = await puppeteer.connect({
      browserWSEndpoint: `wss://chrome.browserless.io?token=${browserlessToken}`,
    });

    const page = await browser.newPage();

    await page.goto('https://ispec.tech');
    const arr = await page.screenshot();

    // https://slack.dev/node-slack-sdk/web-api
    // slack/web-apiのfiles.uploadV2ではfile意外にfileBufferとReadStreamを渡せる
    const buf = Buffer.from(arr);

    await browser.close();

    const client = new WebClient(slackToken);
    await client.files.uploadV2({
        channel_id: "your-channel-id",
        file: buf,
        filename: "ispec.png",
    });
}

// 毎日朝9時(JST)に起動
Deno.cron("Daily Cron morning", "0 0 * * *", async () => {
    await main();
});

```

結果はこんな感じできちんと投稿されました！✌️ (アニメーションのせいかなんか白いですが、まぁよし)
[![Image from Gyazo](https://i.gyazo.com/8c4ce7039786c831a84423b53f8b5104.png)](https://gyazo.com/8c4ce7039786c831a84423b53f8b5104)


# 終わりに

Deno + browserlessの破壊力が伝わったら嬉しいです

人間がやるべきではない仕事はゴンゴン自動化していきましょう!
