---
title: dotenvxとhuskyで環境変数を安全にGit管理する
slug: dotenvx-husky
description: チーム開発で環境変数の運用に課題感があったので、dotenvxとhuskyで解決しました。
date: '2026-03-29'
---

## はじめに

チーム開発で`.env`ファイルの共有に困ったことはないでしょうか。弊社ではSlackのDMで送り合ったり、Notionに書いて共有したりしていました。しかし、変数を追加したときに更新や共有を忘れたり、ローカルの値が古いまま開発してしまったり、問題が起きがちです。

そこで、[dotenvx](https://dotenvx.com/)で`.env`ファイルを暗号化してGit管理するようにしました。さらに、[husky](https://typicode.github.io/husky/)のpre-commitフックで平文の環境変数のcommitを防止する仕組みを導入しました。この記事ではセットアップから実際の運用フローまでを紹介します。

## 課題感

`.env.*`ファイルは`.gitignore`しないと安心できませんが、同時に、Gitで管理できたらいいのに...！というフラストレーションも溜まります。

特に次のような場面で課題を感じていました。

- 新メンバーが参加したとき、すべての環境変数を手動で渡す必要がある
- 誰かが新しいAPIキーを追加したのに共有が漏れて、他のメンバーの開発環境で動作しない
- 開発環境と本番環境で値が乖離していることに気づかない

暗号化して安全にGit管理できれば、バージョン管理・コードレビュー・`git pull`での自動共有を実現できます。

この対策として、 [環境変数を暗号化してGitで管理する](https://zenn.dev/andmohiko/articles/3ebf7036294c38) にて紹介した方法で運用していましたが、こちらも手動で暗号化・復号する必要があり、更新漏れや手順の煩雑さを感じていました。

## dotenvxとは

dotenvxは、`.env`ファイルを暗号化して安全にGit管理できるCLIツールです。dotenvの作者が開発しています。

### 暗号化の仕組み

dotenvxはAES-256暗号化を使用しています。

- **DOTENV_PUBLIC_KEY**: 暗号化に使用する公開鍵。各`.env`ファイル内に記載され、commitして問題ない
- **DOTENV_PRIVATE_KEY**: 復号に使用する秘密鍵。`.env.keys`に保存され、絶対にcommitしてはいけない

暗号化前後の`.env`ファイルはこのようになります。

```bash
# 暗号化前
DATABASE_URL="postgresql://user:pass@host/db"
API_KEY="sk-1234567890"
```

```bash
# 暗号化後
#/-------------------[DOTENV_PUBLIC_KEY]--------------------/
#/            public-key encryption for .env files          /
#/       [how it works](https://dotenvx.com/encryption)     /
#/----------------------------------------------------------/
DOTENV_PUBLIC_KEY="04a9..."
DATABASE_URL="encrypted:BFH/jShhZMhn..."
API_KEY="encrypted:A1B2C3..."
```

`dotenvx run`コマンドがアプリ起動前に環境変数を復号し、`process.env`に注入する仕組みです。

## huskyとは

huskyはGit hooksを簡単に管理できるツールです。今回はpre-commitフックを使って、平文の`.env`ファイルがcommitされるのを防止する安全装置として導入しました。

### pre-commitフックの役割

dotenvxで暗号化する運用にしていても、復号して編集した後に再暗号化を忘れてcommitしてしまうリスクがあります。pre-commitフックでは、ステージングされた`.env`ファイルの値が`encrypted:`で始まるかどうかをチェックし、平文が検出されたらcommitをブロックします。

人間はミスをするものなので、仕組みで防止できるのは大きな安心感があります。

## セットアップ

ここからはNext.jsを使ったモノレポ（pnpm）を前提にセットアップ手順を紹介します。

### dotenvxとhuskyのインストール

dotenvxはアプリケーションのディレクトリに、huskyはプロジェクトルートにインストールします。

```bash
# dotenvx（アプリケーションディレクトリに追加）
pnpm add -D @dotenvx/dotenvx --filter <app-package-name>

# husky（プロジェクトルートに追加）
pnpm add -D husky -w
```

### huskyの初期化

huskyを初期化し、`.husky`ディレクトリを作成します。

```bash
npx husky init
```

これでルートの`package.json`に`prepare`スクリプトが追加され、`.husky/pre-commit`が自動生成されます。

### pre-commitフックの作成

`.husky/pre-commit`を以下の内容で書き換えます。

```bash
# dotenvx: 暗号化されていない.envファイルのcommitを防止
# commit対象の.envファイルに平文の値が含まれていないかチェックする

# ステージされているファイルの中で、環境変数のファイルがあれば取得する。ただし、.env.exampleは除外する
staged_env_files=$(git diff --cached --name-only | grep -E '(^|/)\.env(\.[^/]+)?$' | grep -v '\.env\.keys' | grep -v '\.env\.example' || true)

# 環境変数ファイルがなければ終了する
if [ -z "$staged_env_files" ]; then
  exit 0
fi

has_error=false

for file in $staged_env_files; do
  staged_content=$(git show ":$file" 2>/dev/null)
  if [ $? -ne 0 ]; then
    echo "⚠️  ステージされたファイルの内容を取得できませんでした: $file"
    has_error=true
    continue
  fi

  # 環境変数の中身で平文のままになっているものがないか確認する
  plaintext_lines=$(echo "$staged_content" \
    | grep -E '^[A-Za-z_][A-Za-z0-9_]*=' \
    | grep -v '^DOTENV_PUBLIC_KEY' \
    | grep -v '^#' \
    | grep -v '="encrypted:' \
    | grep -v "='encrypted:" \
    | grep -v '=encrypted:' || true)

  if [ -n "$plaintext_lines" ]; then
    echo "❌ 暗号化されていない環境変数が検出されました: $file"
    echo "$plaintext_lines" | while read -r line; do
      key=$(echo "$line" | cut -d'=' -f1)
      echo "   - $key"
    done
    has_error=true
  fi
done

if [ "$has_error" = true ]; then
  echo ""
  echo "💡 暗号化してからcommitしてください:"
  echo "   cd apps/frontend && pnpm run env:encrypt"
  exit 1
fi
```

ポイントは、`git show`でステージされた内容を取得しているところです。ワーキングディレクトリのファイルではなく、実際にcommitされる内容をチェックするため、正確な検出ができます。

### .gitignoreの設定

`.env.keys`は秘密鍵を含むので、`.gitignore`に追加します。

```gitignore
# dotenvx - 秘密鍵（絶対にcommitしない）
.env.keys
```

### 環境変数ファイルの作成

Next.jsプロジェクトでは、以下のファイル構成が推奨されます。

```
apps/frontend/
├── .env                      # サーバー側の環境変数・開発環境
├── .env.local                # クライアント側の環境変数・開発環境
├── .env.production           # サーバー側の環境変数・本番環境
├── .env.production.local     # クライアント側の環境変数・本番環境
├── .env.keys                 # 秘密鍵（⚠️ git管理しない）
└── .env.example              # 環境変数のテンプレート（git管理）
```

`.env.example`には変数名だけを記載し、値はダミーにしておきます。新メンバーがどの変数を設定すればいいか把握するためのドキュメントとして機能します。

### package.jsonにスクリプトを追加

アプリケーションの`package.json`にスクリプトを追加します。

```json
{
  "scripts": {
    "dev": "dotenvx run --convention=nextjs -- next dev",
    "env:encrypt": "dotenvx encrypt && dotenvx encrypt -f .env.local && dotenvx encrypt -f .env.production && dotenvx encrypt -f .env.production.local",
    "env:decrypt": "dotenvx decrypt && dotenvx decrypt -f .env.local && dotenvx decrypt -f .env.production && dotenvx decrypt -f .env.production.local"
  }
}
```

`--convention=nextjs`フラグは、[Next.jsの標準的な読み込み順序で環境変数を注入するオプション](https://dotenvx.com/docs/advanced/run-convention-nextjs)です。Next.js以外のプロジェクトでは不要です。

### 暗号化の実行と確認

環境変数を記入したら暗号化を実行します。

```bash
pnpm run env:encrypt
```

これにより、各`.env`ファイルの値が`encrypted:...`形式に暗号化され、`.env.keys`ファイルが生成されます。

暗号化後、pre-commitフックの動作も確認しておきましょう。暗号化済みのファイルをcommitできること、平文のファイルがブロックされることの両方を試すと安心です。

## 実際の運用フロー

### 環境変数を追加・変更する

環境変数を追加・変更する際は、以下の流れで作業します。

```bash
# 復号
pnpm run env:decrypt

# .envファイルを編集

# 再暗号化
pnpm run env:encrypt

# commit
git add .env .env.local .env.production .env.production.local
git commit -m "chore: update env vars"
```

万が一再暗号化を忘れても、pre-commitフックがブロックしてくれるので安心です。

### 新メンバーのオンボーディング

dotenvx導入後、新メンバーのオンボーディングはとてもシンプルになりました。

1. リポジトリをclone
2. チームメンバーから`.env.keys`を受け取り、アプリケーションディレクトリに配置
3. `pnpm install`
4. `pnpm dev`

これだけで開発を始められます。以前は環境変数を一つ一つ共有していましたが、`.env.keys`を1ファイル渡すだけで済むようになりました。

### Vercelへのデプロイ

dotenvxはCLIラッパーとして動作するため、ビルド時のみ環境変数を復号できます。Vercelのサーバーレスランタイム（SSR/Server Actions）ではdotenvxが動作しないため、Vercelの環境変数にすべての本番変数を直接登録する必要があります。

本番環境の環境変数を変更した場合は、`.env`ファイルの更新とVercelダッシュボードの更新を両方行う必要がある点に注意してください。

## t3-envとの併用

dotenvxは暗号化・復号の仕組みを提供するツールであり、環境変数の型バリデーションは守備範囲外です。[t3-env](https://env.t3.gg/)を併用すると、dotenvxが`process.env`に値を注入した後にt3-envがZodスキーマでバリデーションを行う流れになり、暗号化管理と型安全なバリデーションを両立できます。

変数を追加した際は`.env`ファイルとt3-envのスキーマの両方を更新する必要がありますが、スキーマの更新を忘れるとビルドエラーになるので、漏れに気づきやすいのも良いところです。

## この運用で出てくる不満

### `.env.keys`の再共有

新しい`.env`ファイルを追加すると、新しい公開鍵・秘密鍵のペアが生成されます。`.env.keys`が更新されるため、チームメンバーへの再共有が必要になります。`.env`ファイルが増えるケースはそんなに多くないので目を瞑ります。

### `.env.example`は手動メンテナンス

暗号化の対象外なので、変数を追加・削除したときに手動で更新する必要があります。現在は甘んじて受け入れています。

## さいごに

dotenvxで暗号化、huskyで平文commitの防止という組み合わせで、`.env`ファイルを安全にGit管理できるようになりました。

導入してみて特に良かったのは、新メンバーのオンボーディングが楽になったことと、環境変数の変更がPRで追跡できるようになったことです。暗号化されているので差分から値を読み取ることはできませんが、「いつ誰がどの変数を変更したか」はGitの履歴からわかります。
また、平文の環境変数をリモートにpushしてしまうことも防げるようになり、安心感があります。

環境変数の管理に課題を感じている方はぜひ試してみてください。

## 参考

- [dotenvx](https://dotenvx.com/)
- [husky](https://typicode.github.io/husky/)
- [t3-env](https://env.t3.gg/)
