---
title: インフラ費削減のためPlanet ScaleからSupabaseへ移行しました
slug: planetscale-supabase-migration
description: インフラ費用が高騰していたため、Planet Scale（MySQL）からSupabase（PostgreSQL）へデータベースを移行しました。移行手順と躓いたポイントを紹介します。
date: '2026-01-27'
---

## はじめに

社内ツールでPlanet Scaleをデータベースに使用していたのですが、インフラ費用に耐えきれず、Supabaseに移行しました。
今回はPlanet Scale（MySQL）からSupabase（PostgreSQL）への移行手順とつまづいたポイントについて書きました。

## モチベーション

ブルーメンヘラでは社内ツールのデータベースにPlanet Scaleを使用しています。以前はPlanet Scaleには無料枠があり、社内ツール程度であれば無料枠の範囲で動かせたため採用していました。しかし、[Planet Scaleは2024年に無料枠が廃止](https://planetscale.com/blog/planetscale-forever)され、毎月47ドルかかるようになりました。
[Supabaseは基本料金＋DBインスタンスごとに課金される料金体系](https://supabase.com/pricing)ですが、すでに他のアプリケーションでSupabaseのデータベースを使用していたため、今回移行すればインフラ費用が47ドル→10ドルに削減できることになります。

Supabaseを使うことでPostgreSQLを使用できるというメリットももちろんありますが、今回移行した社内ツールではMySQLでも問題なかったため、こちらのメリットはあまり感じていません。

## 移行の方針の比較検討

MySQLからPostgresへの移行手段として、

- pgLoaderを使う（MySQL→PostgreSQL移行ツール）
- CSVエクスポート→インポート
- カスタムスクリプトを書く

などがあります。

pgLoaderを使用したかったのですが、PlanetScaleはVitessを使用しており、`__vtschemaname`という内部変数が必要ですが、pgloaderがこちらに対応していませんでした。
そこで、今回はmysqldumpで一度全データを吸い出し、PostgreSQLにinsertするという方針を取りました。

## 使用ツール

今回の移行では、以下のツールを使用しました。

### [mysqldump](https://dev.mysql.com/doc/refman/8.0/ja/mysqldump.html)

MySQLからデータをエクスポートするための標準的なツールです。macOSでは`brew install mysql-client`でインストールできます。

### [Prisma](https://www.prisma.io/)

データベーススキーマの管理にPrismaを使用しています。PostgreSQL側のテーブル構造はPrismaスキーマから生成しました。

### [psql](https://www.postgresql.jp/docs/9.4/app-psql.html)

PostgreSQLにデータをインポートするために使用しました。

## 手順

### 1. MySQLからデータをエクスポート

まず、PlanetScaleからデータをエクスポートします。`mysqldump`コマンドを使用して、データのみ（テーブル定義を含まない）のダンプを作成しました。

```bash
/opt/homebrew/opt/mysql-client/bin/mysqldump \
  --single-transaction \
  --skip-add-locks \
  --skip-disable-keys \
  --complete-insert \
  --no-create-info \
  --no-tablespaces \
  --set-gtid-purged=OFF \
  --ssl-mode=REQUIRED \
  -h <MYSQL_HOST> \
  -u <MYSQL_USER> \
  -p'<MYSQL_PASSWORD>' \
  <DATABASE_NAME> > mysql_data.sql
```

重要なオプションについて説明します：

- `--no-create-info`: テーブル定義を含めない（Prismaで管理するため）
- `--no-tablespaces`: PlanetScale対応
- `--ssl-mode=REQUIRED`: SSL接続必須（PlanetScale）

### 2. PostgreSQLにテーブル構造を作成

Prismaスキーマを使用して、PostgreSQL側にテーブル構造を作成します。

```bash
npx prisma db push
```

### 3. MySQLダンプをPostgreSQL形式に変換

MySQLから吸い出したデータはMySQLの文法で記述されています。MySQLとPostgreSQLの構文の違いを吸収する必要がありため、変換スクリプトを自作しました。

主な処理内容は下記の通りです。

1. バッククォート(`)をダブルクォート(")に変換
2. Boolean値の変換（0→false, 1→true）
3. MySQLコメント（`/*!...*/`）の削除
4. MySQL特有の構文削除（`SET`文、`LOCK TABLE`など）

変換スクリプトの全体像はGitHubで公開しています。

https://github.com/andmohiko/planetscale-supabase-migration

### 4. PostgreSQLにデータをインポート

外部キー制約を一時的に無効化してからデータをインポートします。

```bash
psql "<POSTGRESQL_CONNECTION_STRING>" << 'EOF'
SET session_replication_role = 'replica';
\i postgres_data.sql
SET session_replication_role = 'origin';
EOF
```

`session_replication_role = 'replica'`を設定することで、外部キー制約やトリガーを一時的に無効化できます。これにより、テーブル間の依存関係を気にせずにデータをインポートできます。

### 5. データ確認

移行後、データが正しくインポートされたことを確認します。

```bash
psql "<POSTGRESQL_CONNECTION_STRING>" -c "
SELECT 'companies' as table_name, count(*) FROM companies
UNION ALL SELECT 'users', count(*) FROM users
ORDER BY table_name;"
```

## つまづいたこと

主につまづいたポイントは変換スクリプトの実装です。実際にはclaude codeが正規表現を書きまくりました。

### Boolean値の型変換

MySQLでは`TINYINT(1)`で表現されるBoolean値が、PostgreSQLでは`BOOLEAN`型になります。そのため、ダンプファイル内の`0`と`1`を`false`と`true`に変換する必要がありました。

```
ERROR: column "is_active" is of type boolean but expression is of type integer
```

このエラーが発生した場合、変換スクリプトでBoolean値を正しく変換する必要があります。単純な置換では、他の数値フィールドにも影響を与えてしまうため、カラムの位置を考慮した変換ロジックが必要でした。

### MySQLコメントの処理

MySQLダンプには`/*!...*/`形式の条件付きコメントが含まれています。これはMySQL独自の構文であり、PostgreSQLでは解釈できないためエラーになります。

```
ERROR: unterminated /* comment
```

変換スクリプトで、これらのコメントを適切に削除する処理を追加しました。

### 外部キー制約による挿入順序の問題

テーブル間に外部キー制約がある場合、データの挿入順序が重要になります。親テーブルのデータが存在しない状態で子テーブルにデータを挿入しようとすると、エラーになります。

この問題は、`session_replication_role = 'replica'`を設定して外部キー制約を一時的に無効化することで解決しました。

### SSL接続の要件

PlanetScaleはSSL接続が必須であり、`--ssl-mode=REQUIRED`オプションを指定しないとエラーになります。

```
server does not allow insecure connections, client must use SSL/TLS
```

## さいごに

最終的に、PlanetScaleからSupabaseへの移行は半日ほどで完了しました。
移行してから数日間様子を見て、問題なければPlanet Scaleを解約します。

移行のポイントをまとめると：

1. テーブル定義はPrismaで管理: データのみをエクスポートし、スキーマはPrismaで統一管理する
2. 変換スクリプトで差異を吸収: MySQLとPostgreSQLの構文の違いを変換スクリプトで処理する
3. 外部キー制約を一時的に無効化: インポート時の順序問題を回避する
4. 移行後のデータ確認: レコード数やリレーションの整合性を確認する

インフラ費用の削減やサービスの改善のために移行を検討している方の参考になれば幸いです。
