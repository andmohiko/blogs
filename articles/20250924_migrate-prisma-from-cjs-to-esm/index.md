---
title: Prisma ClientをCJSからESMに移行する
slug: migrate-prisma-from-cjs-to-esm
description: Prisma Client v6.6.0からESMに対応したため、CJSからESMへの移行方法と、その背景にある技術的な理由を解説します。
date: '2025-09-24'
---

## はじめに

Prismaは[2025年4月のアップデート](https://www.prisma.io/blog/prisma-orm-6-6-0-esm-support-d1-migrations-and-prisma-mcp-server)でv6.6.0からPrisma ClientがESMに対応しました。
今回はPrisma Clientを従来のCJSの書き方からESMの書き方に移行する方法を書きました。

今回の移行作業は下記のプルリクエストにまとまっています。
https://github.com/andmohiko/next-hono-monorepo/pull/1

## Prismaとは

[Prisma](https://www.prisma.io/)は、TypeScriptとJavaScript向けの次世代ORM（Object-Relational Mapping）です。データベースとのやり取りを型安全で直感的に行うことができるツールとして、多くの開発者に愛用されています。

Prismaの主な特徴：
- 型安全性: TypeScriptの型システムを活用し、コンパイル時にデータベースクエリの型チェックが可能
- 直感的なAPI: SQLを直接書かずに、JavaScript/TypeScriptのオブジェクト指向的な書き方でデータベース操作が可能
- マイグレーション管理: データベーススキーマの変更をバージョン管理できる
- Prisma Studio: データベースの内容を視覚的に確認・編集できるGUIツール

## 移行のメリット

ESMに移行することで、以下のメリットが得られます。

- 標準準拠: JavaScriptの標準的なモジュールシステムを使用
- パフォーマンス向上: 静的な解析による最適化
- ツリーシェイキング: 未使用コードの削除によるバンドルサイズの削減
- 将来性: モダンなJavaScriptエコシステムとの互換性

筆者も今後はなるべくESMに寄せたいと思っていたため、今回の移行をすることにしました。

## 実際の移行作業

Prisma ClientをCJSからESMに移行する手順を説明します。移行作業は[公式に記載されている通り](https://www.prisma.io/blog/prisma-orm-6-6-0-esm-support-d1-migrations-and-prisma-mcp-server)です。主に設定ファイルの変更とimport文の書き換えが中心です。

本記事では、ESMとしてビルドしているプロジェクトでの移行を想定します。使用する環境は問いませんが、Node.js 14以降での実行を推奨します。

### 1. Prisma Clientのバージョンアップ

まず、Prisma Clientをv6.6.0以上にアップデートします。

```bash
$ pnpm install prisma@latest @prisma/client@latest
```

### 2. schema.prismaの書き換え

`schema.prisma` でESMとして動作するように指定します。

```
// schema.prisma
generator client {
  provider     = "prisma-client" // 新しいジェネレーター
  output       = "../src/generated/prisma" // 必須：出力パス
  moduleFormat = "esm" // ESM形式を指定
}
```

`output`のパスは好みです。アプリケーション内で参照するため、`src/`内に生成するのがおすすめです。

### 3. import文の書き換え

従来のCJS形式のimport文をESM形式に書き換えます：

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '~/generated/prisma/client'
const prisma = new PrismaClient()
```

### 4. Prisma Clientの再生成

設定変更後は、Prisma Clientを再生成してみましょう

```bash
pnpm prisma generate
```

`output`で指定したパスにPrisma Clientが生成されていれば移行完了です。
`/generated`はgitignoreしておいてもよいかもしれません。

## この書き換えで内部的に起こっていること

ESMに移行する際に書き換えが必要な理由について説明します。これは単純にPrisma Clientのimport文を変更するだけでなく、JavaScriptのモジュールシステムの根本的な違いに関わってきます。

### CommonJSとES Modulesの違い

#### CommonJS（CJS）

CJSはNode.jsの従来のモジュールシステムで、`require()`と`module.exports`を使用します。モジュールは同期的に読み込まれ、実行時にモジュールが解決されます。

```
// 実行時に決まる動的インポート
const moduleName = someCondition ? 'moduleA' : 'moduleB'
const myModule = require(moduleName) // 実行時に評価

// 同期的読み込み - ファイルを読み終わるまで待つ
const fs = require('fs') // この行で完全に読み込み完了
console.log('loaded!') // require完了後に実行
```

#### ES Modules（ESM）

ESMはJavaScriptの標準的なモジュールシステムで、`import`と`export`を使用します。モジュールは静的な解析が可能で、コンパイル時にモジュールが解決されます。

```
// コンパイル時に決まる静的インポート
import { someFunction } from 'myModule' // 文字列は固定値のみ

// 非同期的読み込み - Promise的な仕組み
import fs from 'fs' // 実際の読み込みは後で行われる
console.log('this runs first!') // import文の解決を待たない
```

### なぜ書き換えが必要なのか

今までのPrisma Clientは`pnpm prisma generate`をすると`node_modules/`配下に生成され、アプリケーション側からはそちらをimportして使用することができました。しかし、ESMではPrisma Clientの生成先を指定する必要があります。
こちらを指定しないと、

```
q.default.join(dirname, "../query-engine-darwin");
ReferenceError: dirname is not defined in ES module scope
```

というエラーにぶつかります。これはESMではモジュールスコープであることが理由です。

#### `__dirname`の問題

CJSでは自分が書いたファイルが実際には関数の中にラップされて実行されるため、`__dirname`は自動的に提供されます。

```
// 自分が書いたファイル（example.js）
const fs = require('fs')
console.log('Hello')

// ↓ Node.jsが内部的に変換（簡略化）
function moduleWrapper(exports, require, module, __filename, __dirname) {
  // ↑ これらの変数が自動的に渡される
  
  const fs = require('fs')  // ← あなたのコード
  console.log('Hello')      // ← あなたのコード
  
  // __dirnameが使える理由：関数の引数として渡されているから
  console.log(__dirname) // '/path/to/current/directory'
}

// 実際の実行時
moduleWrapper(
  {}, // exports
  require, // require関数
  {}, // module
  '/path/to/file.js', // __filename
  '/path/to' // __dirname ← ここで値が渡される
)
```

しかし、ESMではファイルが関数でラップされず、その代わりにモジュール自体がスコープになります。そのため、`__dirname`は提供されません。

```
// 自分が書いたファイル（example.mjs）
import fs from 'fs'
console.log('Hello')

// ↓ Node.jsでの扱い（関数ラップなし）
// グローバルスコープでもなく、関数スコープでもない
// 「モジュールスコープ」という独立したスコープ

console.log(__dirname) // ReferenceError: __dirname is not defined
// ↑ 関数の引数として渡されていないので存在しない
```

#### バンドリング時の違い

また、バンドリング時の違いもあります。
CJSバンドリングの場合、すべてが同期的なので、順次処理されます。
```
const moduleA = require('./moduleA') // 1. これを完全に読み込み
const moduleB = require('./moduleB') // 2. 次にこれを読み込み
```

依存関係が明確で、バンドラーが理解しやすい形になっています。
これに対し、ESMでは静的に解析されるため、実行時まで何をインポートするかわからないものはバンドラーが解釈できなくなります。
```
// 静的解析は得意
import { funcA } from './moduleA' // OK: 静的

// 動的インポートは可能だが、静的解析は困難
const moduleName = './module' + suffix
import(moduleName) // OK: 動的インポート（Promiseを返す）
```

#### Prisma内部で起きていること

この違いがPrisma内でも起きます。
CJSでは次のようなパスの解決が走っています。

```
// Prismaの内部的な処理
// 1. プラットフォーム検出
const platform = process.platform // 'darwin', 'linux', etc.

// 2. 動的パス構築
const enginePath = path.join(__dirname, `query-engine-${platform}`)

// 3. 動的require（CJSでは問題なし）
const queryEngine = require(enginePath)
```

これをESMバンドル環境で行うと、
- __dirnameが存在しない
- 動的require()ができない
- バンドラーがファイルパスを事前に解決できない

という問題が起きます。
そこで、そのため、`schema.prisma`で`output`としてPrisma Clientの生成先を指定し、新しいジェネレーターでは静的なTypeScriptとして出力することで、バンドラーが他のコードと同じように処理できるようにしているということでした。

## さいごに

Prisma ClientのCJSからESMへの移行は、設定ファイルの変更とimport文の書き換えが中心となる比較的簡単な作業でした。しかし、移行の背景にはJavaScriptのモジュールシステムの進化と、モダンな開発環境への対応という重要な意味があります。プロジェクト全体の依存関係やビルド設定を確認しながら進めることになるため、CJSとESMの勉強にもなりました。

みなさんもぜひPrismaをESM形式に書き換えてみてください。
