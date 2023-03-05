---
title: "discordにChatGPTを導入した" # 記事のタイトル
emoji: "🤖" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["discord", "chatgpt", "discordjs", "discordbot", "openai"]
published: true # 公開設定（falseにすると下書き）
---

# はじめに
新雪降り積もる真冬の夜にふと、人肌が恋しくなり誰かと話したいと思って
DiscordにChatGPTを導入してみたので、その備忘録を残します。  

成果物は以下です。  
https://github.com/sori883/ChatGTPinDiscord  

# 導入

## OpenAIのAPI keyの取得
以下のサイトからAPI Keyを取得出来ます。  
https://platform.openai.com/account/api-keys  

注意点として、APIの利用は有料です。  
詳細は[備考：OpenAIの料金について](#備考openaiの料金について)を参照ください。

## Discord Botの作成
以下のサイトより作成出来ます。  
https://discord.com/developers/applications  

サイト上部のNew Applicationからアプリケーションを作成  

![Discordアプリケーションの作成](/images/discord_bot_with_chatgtp/discord_newapp.jpg)

BotタブからBotを作成し、`MESSAGE CONTENT INTENT`を有効にします。  
※他者にBotを公開したくない場合は、`Public Bot`を無効にしておいてください。

また、ここでDiscord Tokenを取得出来ますので、メモっておいてください。    

![Dicord BotのIntents](/images/discord_bot_with_chatgtp/bot_intents.jpg)

次に、OAuth2タブのURL Generatorから、招待URLを作成します。  
Scopeの`Bot`、Bot Permissionsの`Read Message/View Channess`・`Send Message`にチェックを入れ、画面下部のURLからDiscord Serverにボットを招待します。  

![Dicord BotのScopeとpermissions](/images/discord_bot_with_chatgtp/bot_scope_permissions.jpg)


## Discord Botのコードを作成
バージョンは以下の通りです。  
|  パッケージ  |  バージョン  |
| ---- | ---- |
|  discord.js  |  14.7.1  |
|  openai  |  3.1.0  |

以下、コードをそのまま載せますがコメントの通りです。  

```typescript:main.ts
import { Client, GatewayIntentBits } from 'discord.js';
import dotenv from 'dotenv';
import { Configuration, OpenAIApi } from 'openai';

dotenv.config();

// OpenAIの設定
const configuration: Configuration = new Configuration({
  apiKey: process.env.OPENAI_API_KEY,
});
const openai = new OpenAIApi(configuration);

// Discordの設定
const client: Client<boolean> = new Client({ intents: [
  GatewayIntentBits.Guilds,
  GatewayIntentBits.MessageContent,
  GatewayIntentBits.GuildMessages
] });

client.once('ready', async () => {  
  // 開始ログ出力する
  console.log('ready!!');
});

client.on('messageCreate', async (message) => {
  // botの発言はスルー
  if (message.author.bot) return;

  try {
    // メッセージをPromptに設定してAPIを叩く
    const completion = await openai.createCompletion({
      model: 'text-davinci-003',
      prompt: `${message.content}`,
      max_tokens: 1024,
      stop: null,
      n: 1,
      temperature: 0.5,
    });

    if (completion.data.choices[0].text === undefined) throw new Error();
    
    // ChatGPTのお言葉をDiscordに送信
    await message.channel.send(completion.data.choices[0].text);
  } catch (err) {
    console.log(err);
  };
});

client.login(process.env.TOKEN);
```

.envにDiscordで入手したTokenと、OpenAIのAPI KEYを設定  
```shell:.env
TOKEN=YOUR_DISCORD_TOKEN
OPENAI_API_KEY=YOUR_OPENAI_API_KEY
```

また、OpenAIのAPIドキュメントは以下です。  
modelなどのプロパティは以下を参照ください。  
https://platform.openai.com/docs/api-reference/completions/create?lang=node.js

## 動作確認
`yarn start`を実行して、ローカルで動作確認してみます。  

返信が帰ってきたらOKです。  
![Dicord BotのIntents](/images/discord_bot_with_chatgtp/bot_chat.jpg)

ちなみにChatGPTの性自認はmaleだそうです。  

## デプロイ
どこでもいいですが、今回は簡単に出来そうなFly.ioにデプロイしてみます。  
https://fly.io/  

### flyctlのインストール
```
# mac OS
brew install flyctl

# Windows
powershell -Command "iwr https://fly.io/install.ps1 -useb | iex"
```

### デプロイ
```
flyctl launch
# 以下、指示に従ってアプリの設定

flyctl deploy
# 以下デプロイ処理
```
支払い方法（クレカ）を登録していない場合は、Flyctl launch実行中に支払い方法の設定を求められますので、fly.ioのサイトでクレカを設定する必要があります。  

### 環境変数の登録
最後に環境変数を登録して終わりです。  

```
flyctl secrets set TOKEN=YOUR_DISCORD_TOKEN
flyctl secrets set OPENAI_API_KEY=YOUR_OPENAI_API_KEY
````

Discordを覗いてみると、Botがオンラインになっているかと思います。  


# 備考：OpenAIの料金について
Open AIは従量課金制で、使用するmodelと、メッセージに含まれるtokensによって料金が決まるようです。

詳しくは以下にお譲りします。  
https://auto-worker.com/blog/?p=7005

https://openai.com/api/pricing/

2023年02月13日現在、3ヶ月のトライアル期間で$18を利用出来るので、お試し程度で利用出来るかと思います。  
また、OpenAIのサイトからUsage limitsを設定出来ますので、暴走する前に設定するといい感じです。  
https://platform.openai.com/account/billing/limits


# 参考
https://zenn.dev/ikepe/articles/3e1fb9c025814a
https://discord.js.org/#/docs/discord.js/main/general/welcome
https://github.com/openai/openai-node