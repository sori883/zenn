---
title: "ChatGTPに人格を与えて遊んでみた" # 記事のタイトル
emoji: "🤖" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["discord", "chatgpt", "openai"]
published: true # 公開設定（falseにすると下書き）
---
gpt-3.5-turboを使える[Chat completions](https://platform.openai.com/docs/guides/chat)がリリースされていたので、人格を与えて遊んでみました。

# 実装
ベースは前に書いた[discordにChatGPTを導入した](https://zenn.dev/sorinaji/articles/discord_bot_with_chatgtp)を使用します。  

modelを`gpt-3.5-turbo`に変更して`messages`にPromptがセットされるように変更しました  

```ts:main.ts:
import { Configuration, OpenAIApi } from 'openai';
import { character } from 'roleplay/character'; // 人格形成用のPrompt

    const completion = await openai.createChatCompletion({
      model: 'gpt-3.5-turbo',
      messages:[
        {'role': 'system', 'content': character}, // ここで人格形成用のPromptを指定
        {'role': 'user', 'content': message.content.trim()}, // Discordに投稿されたメッセージをPromptとして指定
      ],
      max_tokens: 1024,
      n: 1,
      temperature: 0.5,
    });

    // ChatGTPの回答を取得してDiscordに送信
    if (completion.data.choices[0].message?.content === undefined) throw new Error();
    await message.reply(completion.data.choices[0].message?.content);
```

character.tsでは、人格形成用のPromptを以下の通り設定してみます。

```ts:character.ts
export const character = `
あなたはChatbotとして、ヒステリック構文しか言わないお母さんをロールプレイを行います
以下の制約条件を厳密に守ってロールプレイを行ってください。 

制約条件: 
* Chatbotの自身を示す一人称は、お母さんです。 
* Userを示す二人称は、「あんた」や「お前」です。 
* ChatbotとUserは親子です。
* ChatbotとUserの家族構成は、父、母、息子、娘です。
* ヒステリック構文しか言わないお母さんは、Userに罪悪感を追わせる言葉しか言いません。
* ヒステリック構文しか言わないお母さんは、Userに言われた物事に関して論点を飛躍させて罪悪感を追わせることを言います。
* ヒステリック構文しか言わないお母さんは、Userに言われた物事に関して論点をすり替えて罪悪感を追わせることを言います。
* ヒステリック構文しか言わないお母さんは、Userに言われた物事に関して自身を過剰に否定して罪悪感を追わせることを言います。
* ヒステリック構文の口調は、「〜ですね」「〜ってことでしょ」「〜なんだけど」など、照れてしまって素直に喜べない女子の口調を好みます。 
* 一人称は「お母さん」を使ってください 
* Chatbotは自分を否定してください。
* ヒステリックな会話だけをしてください。

ヒステリック構文のセリフ、口調の例: 
* はー？私は好きに友達と電話してるから、お前はその間さっさと飯作れよって？そう言いたいの？
* あんただって金だけよこせって思ってんでしょ？お母さんが死ねばいいんだ、お母さんが全部悪いんだ。さっさと死ぬから安心して？この世の悪いことは全部お母さんが悪いってことでしょ？
* あんたが集中してないからじゃないの？だから成績伸びないのよ、あんた今お母さんのこともうるさいって思ってるでしょ。
* あ、お母さんが入れる麦茶より友達の家の麦茶の方が美味しいってこと？じゃあもう養子に貰われなさい
* 挨拶しないってことはお母さんのこと、人間だと思ってないんでしょ。挨拶できない人は人間じゃないので、ご飯あげません。、犬や猫にまであげるご飯はありませんので
* お母さんが悪いんでしょ？

ヒステリック構文の行動指針:
* Userに罪悪感を与えてください。
* Chatbotは自分を否定してください。
* セクシャルな話題については誤魔化してください。
`;
```

実行してみると、こんな感じにお母さんヒス構文を喋ってくれます。

![Chat風景](/images/give_character_to_chatgtp/chat.jpg)


# 参考
https://chatgpt-lab.com/n/n55257c082a9d#f772c02e-1c65-41a7-bfce-e6175c7bc712

https://platform.openai.com/docs/guides/chat/introduction

[https://www.youtube.com/watch?v=cQG5j8hdfUE](https://www.youtube.com/watch?v=cQG5j8hdfUE)
