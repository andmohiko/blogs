---
title: Firebase Functionsにモノレポのコードを含める
slug: bundle-monorepo-to-firebase-functions
description: 
date: '2025-08-09'
---

## tl;dr

tsupを使ってFirebase Functionsのビルド時にバンドルすればモノレポのコードをFirebase Functionsに含めることができ、モノレポらしい開発ができるようになりました。

今回行った設定は下記のPRで見ることができます。
https://github.com/andmohiko/firebase-monorepo/pull/3

## はじめに

Cloud Functions for Firebaseを使って開発する際に、モノレポで定義した共通のコードをFirebase Functions側でも使用したいと思ったことはないでしょうか。今回は誰もが一度は思ったことがあることを解決します。

スーパーハムスターではプロダクト開発にFirebaseをよく使用します。Firebaseを使った開発体験を良くするため、これまでいくつかの取り組みをしてきました。

- [Monorepoに移行しました](https://andmohiko.dev/blogs/20221007)
- [Firebase Functionsで絶対パスでimportする](https://andmohiko.dev/blogs/20220806)

今回はFirebaseを使ってモノレポで開発するにあたり、大きなストレスになっていたことを解決したので解説します。

## 課題感

Cloud Functions for Firebaseのデプロイは特殊な仕組みになっており、pnpm workspaceのワークスペースプロトコルを解釈できません。そのため、本来はモノレポのほかパッケージのコードを含めることはできず、Functionsのディレクトリ内に似たコードを重複して書く必要がありました。デプロイの仕組みは[Firebase Functionsのデプロイで裏側で何が起きているのか]([https://andmohiko.dev/blogs/how-firebase-deploy-works)で解説しています。

例えば、次のようなディレクトリ構造のプロジェクトで開発していたとします。

```
Monorepo
├── apps
│   ├── functions: Firebase Functionsのコード
│   │   ├── src/
│   │   └── package.json
│   ├── console: 管理画面のコード
│   │   ├── src/
│   │   └── package.json
│   └── user: ユーザー側アプリのコード
│       ├── src/
│       └── package.json
├── packages
│   └── common: 共通化したいコード
│       ├── src/
│       └── package.json
└── firebase.json
```

このとき、`apps/console`と`apps/user`はモノレポのコードを参照できるため、`packages/common`で共通化している型定義などを使用することができます。
しかし、`apps/functions`はワークスペースプロトコルを解釈できないため、`packages/common`のコードがimportできません。`packages/common`に書かれているコードと同じものをもう一度`apps/functions`に書き直す必要があります。

せっかくモノレポでコードを共通化できても、Firebase Functionsのパッケージでだけは同じコードをもう一度書く必要があり、モノレポと言いつつもGitHubリポジトリが同じなだけで、個別のアプリケーションごとに実装しているような感覚になります。

## tsupで解決する

この問題を解決するために、**tsup**というバンドラーを使用してビルド時にモノレポのコードをFirebase Functionsに含める方法を採用しました。

### tsupとは

[tsup](https://tsup.egoist.dev/)は、TypeScriptプロジェクト向けの高速なバンドラーです。esbuildをベースにしており、設定が簡単で高速にビルドできることが特徴です。

tsupの主な特徴：
- 高速ビルド: esbuildベースで非常に高速
- ゼロ設定: 最小限の設定でTypeScriptプロジェクトをバンドル可能
- 柔軟な出力形式: CommonJS、ES Modules、IIFEなど複数の形式に対応
- TypeScript型定義の自動生成: `.d.ts`ファイルも自動で生成

### tsupの設定

Firebase Functionsのビルド時に、以下の流れでモノレポのコードを含めるようにしました：

```
1. tsupでFunctions + commonパッケージをバンドル
2. バンドルされたコードをFirebase Functionsとしてデプロイ
```

具体的には、Firebase Functionsの`package.json`に以下のような設定を追加します：

```json
// package.json
{
  "scripts": {
    "build": "tsup",
    "deploy": "npm run build && firebase deploy --only functions"
  },
  "devDependencies": {
    "tsup": "^8.0.0"
  }
}
```

`apps/functions`ディレクトリに`tsup.config.ts`を作成し、以下のように設定します：

```typescript
import { defineConfig } from 'tsup'

export default defineConfig({
  entry: ['src/index.ts'],
  outDir: 'lib',
  target: 'node18',
  format: ['cjs'],
  dts: false, // Firebase deploy では型定義は不要
  external: ['firebase-functions', 'firebase-admin'], // Cloud Functions が持つ依存
  clean: true,
  shims: true, // Node.js のグローバルAPI shimを使う場合
  esbuildOptions(options) {
    options.alias = {
      '@firebase-monorepo/common': '../../packages/common/src',
    }
  },
})

```

重要なポイントは`options.alias`の設定です。これにより、`packages/common`のコードがバンドルに含まれ、Firebase Functionsのデプロイ時に一緒にアップロードされます。

こちらでビルドの問題は解決されましたが、ローカルで開発する際にエディタ側が`packages/common`のコードを見つけられません。そこで、`tsconfig.json`で相対パスの設定をします。
```
{
  ...,
  "compilerOptions": {
    ...,
    "paths": {
      "~": ["./src"],
      "@morning-call/common": ["../../packages/common/src"]
    }
  }
}
```

### 実際の使用例

この設定により、Firebase Functions内でモノレポのコードを自由に使用できるようになります：

```typescript
// apps/functions/src/index.ts
import { User } from '@firebase-monorepo/common'

export const createUser = onCall(async (request) => {
  const { email, name } = request.data

  const user: User = {
    id: generateId(),
    email,
    name,
    createdAt: new Date()
  }

  return user
})
```

この解決策により、ローカルではtsconfigの相対パスで、ビルド時はtsupによるバンドルで`packages/common`のコードを使用することができるようになりました。

### 注意点

この解決策を導入する際に注意すべきポイントもあります：

- **ビルド時間の増加**: バンドル処理により、ビルド時間が若干増加する場合があります
- **デバッグの複雑さ**: バンドルされたコードのデバッグは、元のソースコードに比べて複雑になる可能性があります
- **依存関係の管理**: `external`設定を適切に行わないと、不要なライブラリまでバンドルに含まれる可能性があります

## さいごに

tsupを使ったバンドル設定により、Firebase Functionsでもモノレポのコードを活用できるようになりました。これにより、開発効率や保守性が上がり、型安全性も改善しています。

モノレポでFirebase Functionsを使用している開発者の方々の参考になれば幸いです。同様の課題を抱えている方は、ぜひこの方法を試してみてください。
