---
title: Firebase Functionsのデプロイで裏側で何が起きているのか
slug: how-firebase-deploy-works
description: firebase deployコマンドではなぜローカルでビルドするのに、クラウドでnpm installするのかについて調べました。
date: '2025-07-16'
---

## はじめに

[Cloud Functions for Firebase](https://firebase.google.com/docs/functions?hl=ja)（以下Firebase Functions）を使うときの、「`firebase deploy`コマンドを実行するだけで関数がデプロイできる」という開発体験の良さが好きです。

エンジニアを始めた頃はnpmを使っていましたが、pnpmを使い始めたある日、**pnpm workspaceを使った開発環境でのデプロイエラー**にぶつかりました。ローカルでは問題なくビルドできるのに、`firebase deploy`を実行するとデプロイに失敗します。調べてみると、クラウド側でもう一度`npm install`が実行されているということを知りました。

「なぜローカルでビルドするのに、クラウドでnpm installするのか？」という疑問が浮かびました。
そこで、Firebase CLIを使った`firebase deploy`の裏側で何が起きているのか、そしてなぜこのような仕組みになっているのかを調べました。

## Firebase Functionsのデプロイ設定を見て思うこと

`firebase.json`ファイルでは次のようにpredeployコマンドを設定することができます。

```json
{
  "functions": [
    {
      "source": "functions",
      "codebase": "default",
      "ignore": [
        "node_modules",
        ".git",
        "firebase-debug.log",
        "firebase-debug.*.log"
      ],
      "predeploy": [
        "cd functions && pnpm run lint",
        "cd functions && pnpm run build"
      ]
    }
  ]
}
```

この設定を見ると、実際のデプロイ処理が走る前にlintとbuildのコマンドが実行されることがわかります。
predeployコマンドはローカルで実行され、pnpmの機能を使ってビルドすることもできます。そのため、ローカルでのビルド成果物をそのままデプロイできるのではないかと思ってしまいます。

しかし、実際にはpnpm特有の機能を使うとデプロイエラーが出ます。この誤解を整理してみます。

## Firebase Functionsのデプロイでよくある誤解

Firebase Functionsのデプロイについて、特に誤解されやすいポイントが2つあります。

### 誤解1: ローカルの依存関係がそのまま使われる

「ローカルでビルドするなら、`node_modules`もそのまま使われる」と考える人もいるでしょう。しかし、実際には**クラウド側で依存関係が再インストール**されます。

### 誤解2: デプロイは単純なファイル転送

「`firebase deploy`はビルド済みファイルを単純にアップロードするだけ」と思われがちですが、実際には**複雑な処理が段階的に実行**されています。

## `firebase deploy`の内部で起きていること

では、実際に`firebase deploy`を実行したとき、裏側で何が起きているのかを詳しく見ていきましょう。

### Step 1: 前処理とバリデーション

```bash
# firebase.jsonの読み込み
# プロジェクト設定の確認
# 認証情報の確認
```

Firebase CLIは、まず設定ファイルを読み込み、プロジェクトの設定やユーザーの認証情報を確認します。

### Step 2: TypeScriptのビルド

```bash
# tsconfig.jsonの読み込み
# TypeScriptファイルのコンパイル
# 出力ディレクトリ（通常はlib/）への出力
```

`functions/`ディレクトリ内のTypeScriptファイルがJavaScriptにコンパイルされます。この時点で、型チェックも実行されます。

### Step 3: package.jsonの処理

```json
{
  "dependencies": {
    "firebase-functions": "^4.0.0",
    "axios": "^1.0.0"
  }
}
```

`package.json`から`dependencies`のみが抽出され、`devDependencies`は除外されます。

### Step 4: アーカイブの作成

以下のファイルがZIP形式でアーカイブされます：

```
archive.zip
├── index.js (ビルド済み)
├── package.json (dependenciesのみ)
├── firebase.json
└── 他の必要なファイル
```

### Step 5: Google Cloud Platformへのアップロード

作成されたアーカイブが、Firebase経由でGCPにアップロードされます。この際、以下の情報も送信されます：

- 関数名
- トリガー設定
- 環境変数
- リソース設定（メモリ、タイムアウトなど）

### Step 6: クラウド側での依存関係インストール

GCP上でアーカイブが展開され、以下のコマンドが実行されます：

```bash
# アーカイブの展開
unzip archive.zip

# 依存関係のインストール
npm install --production

# 最終的な実行環境の準備
```

### Step 7: Cloud Functionsへのデプロイ

最後に、準備が完了した関数がCloud Functionsにデプロイされ、利用可能になります。

## なぜビルドと依存インストールが分かれているのか

ここまで整理すると、**ビルドはローカルで行い、ビルド成果物をGCPにアップロードした上で、依存関係はGCP側で改めてインストールされる**ということがわかります。
このような複雑な仕組みになっている理由はいくつかあります。

### 理由1: 責務の分離

**ビルドに必要なものと実行に必要なものは根本的に異なる**からです。

```json
{
  "devDependencies": {
    "typescript": "^5.0.0",
    "tsc": "^2.0.0",
    "@types/node": "^20.0.0"
  },
  "dependencies": {
    "firebase-functions": "^4.0.0",
    "axios": "^1.0.0"
  }
}
```

- ビルド時に必要: TypeScriptコンパイラ（`tsc`）、型定義ファイル（`@types/*`）など
- 実行時に必要: 実際のライブラリ（`firebase-functions`、`axios`など）

ビルドが完了すると、TypeScriptファイルは純粋なJavaScriptになり、`devDependencies`は不要になります。

### 理由2: 実行環境の最適化

クラウドの実行環境では、**必要最小限の依存関係のみをインストール**することで、以下のメリットがあります：

- 起動時間の短縮: 不要なパッケージがないため、cold start時間が短縮される
- メモリ使用量の削減: 実行時のメモリ使用量が最適化される
- セキュリティの向上: 攻撃対象となりうる不要なパッケージが除外される

### 理由3: 環境の違いへの対応

開発環境とクラウド環境では、以下のような違いがあります：

- OS: macOS/Windows vs Linux
- Node.js バージョン: 開発環境とクラウド環境で異なる可能性
- ネイティブモジュール: OS依存のバイナリが含まれる場合

これらの違いに対応するため、**クラウド環境で依存関係を再構築**する必要があります。

## なぜクラウド側では npm が使われるのか

ここで疑問になるのが、「なぜクラウド側では常にnpmが使われるのか？」という点です。

### Cloud Functionsの実行環境

Cloud Functions for Firebaseは、実際にはGoogle Cloud Functions上で動作しています。**GCFの実行環境はnpm前提で構成されている**（2025年現在）ため、pnpmやyarnなどの他のパッケージマネージャーは使用できません。

### ネイティブモジュールへの対応

多くのNode.jsパッケージには、OS依存のネイティブバイナリが含まれています。例えば：

- `bcrypt`: パスワードハッシュ化ライブラリ
- `canvas`: 画像処理ライブラリ
- `sqlite3`: SQLiteデータベースライブラリ

これらのパッケージは、**実行環境でビルドされる必要がある**ため、クラウド側でのインストールが必要になります。

### セキュリティと再現性

クラウドでの`npm install`は、以下の理由から標準化されています：

- セキュリティ: 信頼できる環境での依存関係インストール
- 再現性: 同じ環境で毎回同じ結果を得られる
- 監査: インストールされるパッケージの追跡が可能

## pnpm workspaceでエラーが発生する理由

この仕組みを理解すると、なぜpnpm workspaceでエラーが発生するのかが明確になります。

### pnpm workspaceの特殊性

pnpm workspaceは、以下のような特殊な構造を持っています：

```json
{
  "dependencies": {
    "@workspace/shared": "workspace:*"
  }
}
```

この`workspace:*`という記法は、**pnpm特有の機能**です。

### エラーの原因

クラウド側では標準的な`npm install`が実行されるため、pnpm特有の機能は理解できません。結果として、依存関係の解決に失敗し、デプロイエラーが発生します。

## まとめ

Firebase Functionsのデプロイは、表面的にはシンプルに見えますが、**ローカルとクラウドで処理を分担する合理的な設計**になっています。

本記事で解説した主なポイントは以下の通りです：

- ビルドはローカル、依存関係インストールはクラウドで実行される
- devDependenciesは実行環境には含まれない
- クラウド側では常にnpmが使用される
- 環境の違いに対応するため、依存関係は再構築される

この仕組みを理解することで、pnpm workspaceなどでのデプロイエラーの原因が分かり、適切な対策を取ることができます。また、パッケージマネージャーの選択や依存関係の管理においても、より良い判断ができるようになるでしょう。

Firebase Functionsの裏側の仕組みを知ることで、より効果的な開発と運用ができることを願っています。
