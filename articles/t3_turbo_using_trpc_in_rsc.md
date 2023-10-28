---
title: "T3-TurboでもT3-Stackに実装されているtRPC in RSCをしたい" # 記事のタイトル
emoji: "🤩" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["trpc", "typescript", "vercel","turborepo","t3stack"]
published: true # 公開設定（falseにすると下書き）
---

# はじめに
現時点のT3-Turboの`api.ts`を参照するとReact Queryを使用したtRPCクライアントしか実装されておらず、データフェチが必要なコンポーネントには`"use client"`と記載してClient Componentとする必要があります。  
https://github.com/t3-oss/create-t3-turbo/blob/main/apps/nextjs/src/utils/api.ts

Server Componentでもデータフェッチしたかったので、実装したの先人がいないか調べていたら、T3-Appに実装されてるよ！と以下のIssueにコメントがありましたので、T3-Turboに移植した作業をこの記事で備忘録として残したいと思います。  
https://github.com/t3-oss/create-t3-turbo/issues/596


## 成果物
レポジトリ
該当のコミットは`README`を参照ください。
https://github.com/sori883/t3-turbo-using-trpc-inside-rsc

※Expoは使ったことがないので、未着手です。  


# 環境準備
T3-AppとT3-Turboの環境を準備します。  
```ts
// T3-App
pnpm create t3-app@latest

// T3-Torbo
npx create-turbo@latest -e https://github.com/t3-oss/create-t3-turbo
```

# tRPCクライアントを移植する
`T3-App/src/trpc`にあるtRPC関連の下記3ファイルを`T3-Turbo/apps/nextjs/src/utils`に移植します。
- react.tsx
- server.ts
- shared.ts

次にエラーが出ている部分を修正します。
```ts
// 変更前
import { type AppRouter } from "~/server/api/root";

// 変更後
import type { AppRouter } from "@acme/api";
```

最後に、元からある`api.ts`は削除します。  

# Providerの変更
TRPCReactProviderを既存の./providersから、先程移植したtRPCクライアントに変更します。  

`T3-Turbo/apps/nextjs/src/app/layout.tsx`を下記の通り編集します。
```
// 変更前
import { TRPCReactProvider } from "./providers";

// 変更後
import { TRPCReactProvider } from "~/utils/react";
```

providers.tsxは削除します。

以上で作業自体は完了です。

# 動作確認

疎通確認用のAPIを用意して、Client、Server Componentからそれぞれ確認します。  

## Server Component
疎通確認用のAPIを以下の通り定義します。  
```ts:post.ts
  protectedPing: protectedProcedure
    .input(z.object({ text: z.string() }))
    .query(({ ctx, input }) => {
      return {
        greeting: `Hello ${ctx.session?.user.id} san, this is protected ping -> ${input.text}`,
      };
    }),
```

```ts:page.tsx
// Server Component用のtRPCクライアント
import { api } from "~/utils/server"

export default async function SampleServer() {
  // 疎通確認
  const hello = await api.post.protectedProcedure.query({ text: "from server" });
  return (
    <div>
        <p>{hello ? hello.greeting : "Loading tRPC query..."}</p>;
    </div>
  );
}
```

## Client Component
疎通確認用APIはServer Componentと同じものを使います。

```ts:pingComp.tsx
"use client";

// Client Component用のtRPCクライアント
import { api } from "~/utils/react"

export default function PingComp() {
  const hello = api.post.protectedProcedure.useSuspenseQuery({ text: "from client" });
  if(!hello) {
    return (
    <div>
      <p>no hello</p>
    </div>
    );
  }

  return (
    <p>{hello[0].greeting}</p>
  )
}
```

```ts:page.tsx
import PingComp from "./pingComp";
import { Suspense } from "react";

export default function SampleClient() {
  return (
    <Suspense fallback={<p className="mt-4">Loading...</p>}>
      <PingComp />
      </Suspense>
  );
}
```

各ページにPingの結果が表示されていたらOKです。


