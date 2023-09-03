---
title: "Next.js製ブログでunifiedを使ってMarkdownを変換する" # 記事のタイトル
emoji: "📝" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["nextjs", "typescript", "markdown","unified"]
published: true # 公開設定（falseにすると下書き）
---

## やったこと
Markdownで書いた文章をそのままブログに載せたかったので、unifiedを使ってMarkdownからReactElementに変換出来るようにしたので、その備忘録を記載します。  

成果物は以下です。
[自分のブログ](https://sori883.dev/posts/this_blog_architecture)

なお、独自記法の変換は行っておりません。  

## Markdown変換の概要
unifiledはMarkdownやHTMLを操作しやすいように、文字列を一度astと呼ばれる木構造に変換して処理を行い、再度木構造からHTML等に変換を行います。  

実際に、MarkdownをReactElement(HTML)に変換する基本的なステップは以下の通りです。  
1. Markdownをmdastにパース
1. mdastをhastに変換
1. hastをReactElementに変換

※mdastはMarkdownのast、hastはHTMLのast  

上記の変換はunifiedに属しているremark、rehypeを使用して変換処理を作っていきます。  
- remarkはMarkdownを扱います（Markdownをmdastにパースなど）
- rehypeはHTMLを扱います（hastをReactElementに変換など）

詳細は以下がとても参考になります。  絶対見たほうがいいです。  
https://qiita.com/sankentou/items/f8eadb5722f3b39bbbf8


## MarkdownからReactElementに変換する
まず単純にMarkdownからReactElementへの変換だけ行います。  
以下の必要なライブラリをインストールします。  
```
yarn add unified remark-parse remark-rehype rehype-react
```

インストールしたライブラリを使って以下の通り変換出来ます。  
上述の通り、markodownからmdast、 mdastからhast、hastからReactElementに変換しています。  

```typescript
import rehypeReact from 'rehype-react';
import remarkParse from 'remark-parse';
import remarkRehype from 'remark-rehype';
import { unified } from 'unified';

export const markdownToReact = (markdown: string): ReactElement => {
  return unified()
    .use(remarkParse) // markdown -> mdast の変換
    .use(remarkRehype) // mdast -> hast の変換
    .use(rehypeReact, {
      Fragment, // 不要なdivで囲まれないようにする
      createElement,
    })
    .processSync(markdown).result; // 変換実行
};
```

変換だけであれば、これだけで可能です。  

## プラグインを使ってMarkdownを拡張する
unifiedは、プラグイン制で一連の変換の中でastを操作する処理が出来るようになっています。  
プラグインは自作も出来ますが、本記事では有名どころの下記を導入してみます。  

- [remark-gfm](https://github.com/remarkjs/remark-gfm)
  - GithubのReadme形式でMarkodownを書けるプラグイン
- [remark-breaks](https://github.com/remarkjs/remark-breaks)
  - 改行をbrにするプラグイン
- [rehype-raw](https://github.com/rehypejs/rehype-raw)
  - Markdownの中でHTMLを使えるようにするプラグイン

### 適用時の注意
プラグインには大抵、`remark-〇〇`、`rehype-〇〇`と名前がついています。
remarkとつくものはmdastに対して、rehypeはhastに対して処理を行うため適用順番に気をつける必要があり、処理の流れとしては以下のイメージです。  

1. remarkParseでMarkdownをmdastに変換
1. `remark-〇〇`系のプラグインを適用する
1. remarkRehypeでmdastをhastに変換する
1. `rehype-〇〇`系のプラグインを適用する
1. hastをReactElementに変換

実際に、先程の変換処理にプラグインを導入すると以下の通り書けます。  

```typescript
// 導入するプラグインをインポート
import rehypeRaw from 'rehype-raw'; 
import remarkBreaks from 'remark-breaks';
import remarkGfm from 'remark-gfm';

import rehypeReact from 'rehype-react';
import remarkParse from 'remark-parse';
import remarkRehype from 'remark-rehype';
import { unified } from 'unified';

export const markdownToReact = (markdown: string): ReactElement => {
  return unified()
    .use(remarkParse)
    .use(remarkGfm) // mdastに対してプラグインの処理を行う
    .use(remarkBreaks) // mdastに対してプラグインの処理を行う
    .use(remarkRehype, {
      allowDangerousHtml: true, // rehype-rawのために直接記載されたタグを許可する
    })
    .use(rehypeRaw) // hastに対してプラグインの処理を行う
    .use(rehypeReact, {
      Fragment,
      createElement,
    })
    .processSync(markdown).result;
};
```

## コンポーネントを置き換える
最後に、`rehypeReact`のコンポーネントを置き換える機能で、H2などの要素を自分の作成したコンポーネントに置き換えてみます。  

本記事ではHTMLのCodeタグをシンタックスハイライト付きのCodeに置き換えます。  

変換処理で、`rehypeReact`の`components`で置き換えたい要素と自作したコンポーネントを指定します。

```typescript
import rehypeRaw from 'rehype-raw'; 
import remarkBreaks from 'remark-breaks';
import remarkGfm from 'remark-gfm';

import rehypeReact from 'rehype-react';
import remarkParse from 'remark-parse';
import remarkRehype from 'remark-rehype';
import { unified } from 'unified';

// 自作したCodeコンポーネント
import { mdCode } from 'components/mdCode';

export const markdownToReact = (markdown: string): ReactElement => {
  return unified()
    .use(remarkParse)
    .use(remarkGfm)
    .use(remarkBreaks)
    .use(remarkRehype, {
      allowDangerousHtml: true,
    })
    .use(rehypeRaw)
    .use(rehypeReact, {
      Fragment,
      components: {
        code: mdCode, // Codeを自作コンポーネントに置き換える
      },
      createElement,
    })
    .processSync(markdown).result;
};
```

自作コンポーネントでは、`@mantine/prism`を使用してシンタックスハイライトが使えるCodeコンポーネントを返しています。  

```typescript
import { ComponentPropsWithoutRef, FC } from 'react';

import { Prism } from '@mantine/prism';
import duotoneDark from 'prism-react-renderer/themes/duotoneDark';

import { isLanguage } from 'types/isLanguage';


export const mdCode: FC<ComponentPropsWithoutRef<'code'>> = ({ className, children }) => {

  const match = /language-(\w+)/.exec(className || '');
  const lang = match && isLanguage(match[1]) ? match[1] : 'bash';

  return (
    <Prism
      language={lang}
      withLineNumbers
      getPrismTheme={(_theme, ) => duotoneDark}
    >
      {String(children).replace(/\n$/, '')}
    </Prism>
  );
};
```

その他にAタグはIMGタグをNextのLinkやImageに置き換えるといい感じになりそうです。  

## あとがき
変換だけであれば意外と簡単にできました。  
そのうちMarkdownの独自記法も頑張って作って記事にしてみます。（そのうち）  


## 参考
https://qiita.com/sankentou/items/f8eadb5722f3b39bbbf8

https://zenn.dev/yoshiishunichi/articles/667120b3d0c9d2