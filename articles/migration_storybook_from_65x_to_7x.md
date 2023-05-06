---
title: "Storybookをバージョン6から7への移行" # 記事のタイトル
emoji: "📙" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["storybook", "nextjs"]
published: true # 公開設定（falseにすると下書き）
---

# はじめに
Storybookのメジャーバージョン7が最新版になりました。  
https://github.com/storybookjs/storybook/releases/tag/v7.0.6  

この記事では、Next.js環境での移行を備忘録として残します。  
移行手順は公式を参考にします。  
https://github.com/storybookjs/storybook/blob/v7.0.2/MIGRATION.md#from-version-65x-to-700  

# アップグレード
公式手順の通りコマンドをポチっていきます。  
```
npx storybook@latest upgrade
```

1. インストールするバージョンが問題ないか聞かれるので`y`を返します。  
```
 Need to install the following packages:
   storybook@7.0.6
 Ok to proceed?
```

2. Storybook7では、`start-storybook`や`build-storybook`が廃止され、`storybook dev`、`storybook build`になりました。    
以下に`y`を返すと`devDependencies`にstorybookというパッケージします。    
```
 Do you want to run the 'storybook-binary' migration on your project?
```

また、以下にも`y`を返すことで`package.json`のコマンドを自動で置き換えます。  
```
 Do you want to run the 'sb-scripts' migration on your project?
```

私の環境では、以下の通りpackage.jsonが変更されました。  
```json:package.json
  変更前
  "storybook": "start-storybook -p 6006",
  "build-storybook": "build-storybook"

  変更後
  "storybook": "storybook dev -p 6006",
  "build-storybook": "storybook build"
```

3. Next.js環境では、`framework`に`@storybook/react`、`builder`に`@storybook/builder-webpack5`を指定して、使用していましたが、`@storybook/nextjs`を使用するようになりました。  
下記に`y`と返すと不要なパッケージを削除して`@storybook/nextjs`をインストールします。  
```
Do you want to run the 'new-frameworks' migration on your project?

 ✅ Removing dependencies: @storybook/builder-webpack5, @storybook/manager-webpack5, 
 storybook-addon-next

 ✅ Installing new dependencies: @storybook/nextjs
```

# main.jsの修正
上記の処理で`Do you want to run the 'new-frameworks' migration on your project?`に`y`と答えると`main.js`も書き換わるらしいですが、手元の環境では変わらなかったので手動で修正しました。  

- frameworkを`@storybook/nextjs`に変更
- storybook-addon-nextを削除
- core(builder)を削除


書き換え前
```js:main.js
const path = require('path')

module.exports = {
  "stories": [
    "../src/**/*.stories.mdx",
    "../src/**/*.stories.@(js|jsx|ts|tsx)"
  ],
  "addons": [
    "@storybook/addon-links",
    "@storybook/addon-essentials",
    "@storybook/addon-interactions",
    '@storybook/addon-a11y',
    {
      name: 'storybook-addon-next',
      options: {
        nextConfigPath: path.resolve(__dirname, '../next.config.js')
      }
    }
  ],
  staticDirs: ['../public'],
  "framework": "@storybook/react",
  "core": {
    "builder": "@storybook/builder-webpack5"
  },
  babel: async (options) => ({
    ...options,
    presets: [...options.presets, '@emotion/babel-preset-css-prop'],
    plugins: ['react-require']
  }),
}
```

書き換え後
```js:main.js
module.exports = {
  "stories": [
    "../src/**/*.stories.mdx",
    "../src/**/*.stories.@(js|jsx|ts|tsx)"
  ],
  "addons": [
    "@storybook/addon-links",
    "@storybook/addon-essentials",
    "@storybook/addon-interactions",
    '@storybook/addon-a11y',
  ],
  staticDirs: ['../public'],
  // 変更部分
  framework: "@storybook/nextjs",
  babel: async (options) => ({
    ...options,
    presets: [...options.presets, '@emotion/babel-preset-css-prop'],
    plugins: ['react-require']
  }),
}
```

# Storyの修正
最後に、非推奨だらけになったStoryを修正しました。  

- ComponentMeta, ComponentStoryなどが非推奨の型になり、代わりに Meta, StoryObjを使用する  
- CSF2も非推奨なので、CSF3に書き換えます。

書き換え前
```tsx:stories.tsx
import { ComponentMeta, ComponentStory } from '@storybook/react';

import { TagChip } from './tagChip';

export default {
  title: 'Elements/TagChip',
  component: TagChip,
} as ComponentMeta<typeof TagChip>;

const Template: ComponentStory<typeof TagChip> = (args) => <TagChip {...args} />;

export const Single = Template.bind({});
Single.args = {
  tagsOnArticles: ['tag1']
};

export const Many = Template.bind({});
Many.args = {
  tagsOnArticles: ['tag1','tag2','tag3']
};
```

書き換え後
```tsx:stories.tsx
import type { Meta, StoryObj } from '@storybook/react';

import { TagChip } from './tagChip';

const meta: Meta<typeof TagChip> = {
  title: 'Elements/TagChip',
  component: TagChip,
};
export default meta;

export const Single: StoryObj<typeof TagChip> = {
  args: {
    tagsOnArticles: ['tag1']
  },
};

export const Many: StoryObj<typeof TagChip> = {
  args: {
    tagsOnArticles: ['tag1','tag2','tag3']
  },
};
```

私は手動で直しましたが、CSF2→CSF3は以下のコマンドで書き換えをしてくれます。  
```
npx storybook@latest migrate csf-2-to-3 --glob="src/**/*.stories.tsx"
```

# あとがき
以上がStorybook7への移行作業でした。    
Storyの数が多くなかったため移行自体は1時間程度で完了し、Github Action→Chromaticにデプロイも変更なしで動作しました(^^)  
