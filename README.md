# dbt Core

## ■概要

- dbtはデータマート開発ツール
- dbtがやってくれることのイメージ
  - ローデータの加工
  - 加工したデータをテーブルやビューに反映する
- `dbt Cloud`と`dbt Core`
  - `dbt Cloud`は、WebベースのUIでデータマート開発ができる`SaaS`
  - `dbt Core`は、dbtコマンドを利用するためのCLI、コマンドを実行してデータマート開発ができる
    - このドキュメントは`dbt Core`の説明

## ■アーキテクチャ

![image](https://i.gyazo.com/cbf7106bcca23875f26e9e5b030adf8e.png)

### ●CLI

dbtコマンドを利用するためのCLI

### ●Aadapter

- データベースにアクセスするためのAdapter
  - BigQueryやPostgresなど多くのデータベースに対応している

### ●Package

- 公開されているパッケージ(Googleアドワーズの支出データの変換 etc...)をimportして利用することが可能

## ■概念

dbtが提供している機能は大きく次の3つに分けられる。

- **データ連携**
  - モデル
  - シードファイル
  - パッケージ
  - テスト
- **チーム開発**
  - ドキュメント
- **データのライフサイクル監視**
  - スナップショットデータ

以降に各機能の概要を記載する

### ●データ連携

#### ▶︎モデル

- ローデータからリレーションを実体化する処理のまとまり
  - SQLモデル
    - SQLのselect文と設定から、リレーションを実体化するためのSQLを生成
    - テンプレートエンジン(Jinja)の利用が可能
    - クエリ内にマクロ(if文やforループ)を埋め込むことができる
    - 生成したSQLでリレーションを実体化(マテリアライゼーション)する
    - 実体化の戦略に沿ってリレーションを最新の状態に保ち続ける
  - Pythonモデル
    - Pythonコード上でリレーションの実体化方法を定義しデータセットを返却する
    - データセットを実体化の戦略に沿ってリレーションを最新の状態に保ち続ける
- モデルの依存関係を辿ってモデルの実行順序が制御可能

#### ▶︎シードファイル

- 変更頻度の少ない静的なデータやマスタデータを、CSVファイルからリレーションに実体化する際に利用する
- :memo: シードファイルはとりあえずテーブル作りたいとか、お試しの時に使ったりもする

#### ▶︎パッケージ

- 公開されているパッケージ(Googleアドワーズの支出データの変換 etc...)をデータ連携時の処理として利用できる
- [dbt - Package hub](https://hub.getdbt.com/)にPackageが公開されている

#### ▶︎テスト

- モデルが期待通りに動作しているか検証する
- テストはSQLを書いて取得した結果が正しいかを検証する

### ●チーム開発

#### ▶︎ドキュメント

リレーションの内容や実体化の戦略等をドキュメントとして生成する

### ●データのライフサイクル監視

#### ▶︎スナップショットデータ

- 任意の生データをスナップショットデータとして保持する
- スナップショットデータは**時間と共に変化するデータの状態を把握するため**に利用する

📝 スナップショットの利用例

- `orders`テーブルの`status`フィールドは注文の進捗状況を反映するものとする

| `id` | status  | updated_at |
| :--- | :------ | :--------- |
| 1    | pending | 2019-01-01 |

- 注文が「保留」から「出荷」になった状態

| `id` | status  | updated_at |
| :--- | :------ | :--------- |
| 1    | shipped | 2019-01-02 |

- この注文は現在「出荷済み」状態でいつ「保留」状態になったのかという情報が失われている
- この情報を失わないようにするのがスナップショット機能
- スナップショット・テーブルの例を次に示す

| `id` | status  | updated_at | dbt_valid_from | dbt_valid_to |
| ---- | ------- | ---------- | -------------- | ------------ |
| 1    | pending | 2019-01-01 | 2019-01-01     | 2019-01-02   |
| 1    | shipped | 2019-01-02 | 2019-01-02     | `null`       |

## ■インストール

- `dbt Core`の[インストール方法は4通り](https://docs.getdbt.com/docs/get-started/installation)ある。
  - `Homebrew`
    - MacOSで`dbt Core`を使うなら`Homebrew`が一番手軽
      - 依存の数が多いので、インストールに凄く時間がかかる
  - `pip`
    - WindowsやLinuxで`dbt Core`を使うならこれ、MacOSで`pip install`したら何か足りないものがあるらしくスッと利用することはできなかった
      - 依存の数が多いので、インストールに凄く時間がかかる
  - `Docker`
    - インストールは早いし、ローカルの開発環境は汚れない
    - 処理時間は`Homebrew`や`pip`に比べると著しく遅い
    - Docker networkの設定に調整が必要な場合もある
  - `Source`
    - 開発中の最新版を使いたい場合はcloneしたプロジェクトからpip installする

## ■はじめ方

[Getting Started(dbt Core)](https://docs.getdbt.com/docs/get-started/getting-started-dbt-core)の手順に沿ってサンプルプロジェクトを作成する

### ▶︎1.準備

#### Gitプロバイダ(GitHub etc..)に新規リポジトリを作成する

プロジェクト名は`jaffle_shop`

#### Postgresのサービスを起動する

- docker-composeでサービスを起動する
  - 起動するサービスとその役割
    - postgres-jaffle_shop-dev
      - 開発DB
    - postgres-jaffle_shop-prod
      - 商用DB
    - pgadmin
      - PostgreSQL用のGUI管理ツール

<!-- markdownlint-disable MD033 -->
<details>
<summary>:spiral_notepad: docker-compose.yml(クリックすると展開されます)</summary>

```yml
version: "3"

services:
  postgres-jaffle_shop-dev:
    image: postgres:15.1
    ports:
      - 55432:5432
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: jaffle_shop
    volumes:
      - postgres-jaffle_shop-dev:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "user", "-d", "jaffle_shop"]
      interval: 5s
      retries: 20
    restart: always
  postgres-jaffle_shop-prod:
    image: postgres:15.1
    ports:
      - 55433:5432
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: jaffle_shop
    volumes:
      - postgres-jaffle_shop-prod:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "user", "-d", "jaffle_shop"]
      interval: 5s
      retries: 20
    restart: always
  pgadmin:
    image: dpage/pgadmin4:6.17
    depends_on:
      postgres-jaffle_shop-dev:
        condition: service_healthy
      postgres-jaffle_shop-prod:
        condition: service_healthy
    ports:
      - 9999:80
    volumes:
      - pgadmin:/var/lib/pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: root@pgadmin.com
      PGADMIN_DEFAULT_PASSWORD: password
    healthcheck:
      test:
        ["CMD-SHELL", "wget -q --spider http://localhost/misc/ping || exit 1"]
      interval: 20s
      retries: 20
    restart: always
volumes:
  pgadmin:
  postgres-jaffle_shop-dev:
  postgres-jaffle_shop-prod:
```

</details>
<!-- markdownlint-enable MD033 -->

### ▶︎2.プロジェクトの作成

1. `dbt --version`を実行してコマンドがインストールされていること/バージョンを確認する

   ```sh
   dbt --version
   ```

1. `jaffle_shop`プロジェクトを`init`コマンドを実行して開始する

   ```sh
   dbt init jaffle_shop
   ```

1. `init`コマンドで生成されたファイルを確認する

   ```sh
   cd jaffle_shop
   ls -1

   # README.md
   # analyses/
   # dbt_project.yml
   # macros/
   # models/
   # seeds/
   # snapshots/
   # tests/
   ```

### ▶︎3.Postgresへの接続

#### 1.profiles.ymlの編集

- ローカルで開発する場合、dbtは[~/.dbt/profiles.yml](https://docs.getdbt.com/docs/get-started/connection-profiles)に記載されたデータベースへの接続情報を利用する。
- [~/.dbt/profiles.yml](https://docs.getdbt.com/docs/get-started/connection-profiles)に準備で起動したDBへの接続情報を記載する

```yml
jaffle_shop:
  outputs:
    dev:
      type: postgres
      # dbtプロジェクトが実行されるスレッド数
      threads: 1
      host: localhost
      port: 55432
      user: user
      pass: password
      dbname: jaffle_shop
      schema: public
    prod:
      type: postgres
      threads: 1
      host: localhost
      port: 55433
      user: user
      pass: password
      dbname: jaffle_shop
      schema: public
  target: dev
```

#### 2.Postgresへの接続確認

`debug`コマンドを実行し設定に問題がないか確認する

```sh
dbt debug

# 05:43:30  Running with dbt=1.3.1
# ...
# All checks passed!
```

#### 3.dbtの初回実行

`init`コマンド実行時に作成されたサンプルモデルを適用する

```sh
dbt run

# 02:24:19  Running with dbt=1.3.1
# ...
# 02:24:19  Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

dbtを実行して作成されたテーブルの取得

```sh
docker-compose exec postgres-jaffle_shop-dev psql -U user jaffle_shop -c "SELECT * FROM public.my_first_dbt_model"

#  id
# ----
#   1
#
# (2 rows)
```

サンプルモデルが適用され、テーブルが実体化できたことがわかった

## ■概念構造のつかい方

作成したjaffel_shopプロジェクトを利用して、dbtの概念構造のつかい方を記載する  
`jaffel_shop`プロジェクトのパッケージ構成を次に記載する

```txt
/jaffel_shop
├── 📁 analyses
├── 📁 dbt_packages
├── 📁 logs
├── 📁 macros
├── 📁 models
├── 📁 seeds
├── 📁 snapshots
├── 📁 target
└── 📁 tests
```

### ●`📁 seeds`:シードファイル

:knife: つかい方

- `📁 seeds`ディレクトリに`.csv`ファイルを配置する
- ファイル名がモデル名として使用される

:point_up: テストデータを取り込む

1. `📁 seeds`ディレクトリに[raw_customers.csv](https://github.com/dbt-labs/dbt-core/blob/main/tests/fixtures/jaffle_shop_data/raw_customers.csv)を配置する
   - `raw_customers.csv`はdbt-coreリポジトリのテストデータ
2. `seed`コマンドを実行してシードファイルを読み込む

   -

   ```sh
   dbt seed --select raw_customers
   # 03:10:18  Running with dbt=1.3.1
   # ...
   # 03:10:19  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
   ```

3. シードファイルのデータが実テーブルに反映されているか確認する

   -

   ```sh
   docker-compose exec postgres-jaffle_shop-dev psql -U user jaffle_shop -c "SELECT * FROM raw_customers limit 5"
   #  id | first_name | last_name
   # ----+------------+-----------
   #   1 | Michael    | P.
   #   2 | Shawn      | M.
   #   3 | Kathleen   | P.
   #   4 | Jimmy      | C.
   #   5 | Katherine  | R.
   # (5 rows)
   ```

### ●`📁 models`:モデル

- ローデータからリレーションを実体化する方法をモデルと呼ぶ
- モデルには`SQLモデル`と`Pythonモデル`の2種類のモデルがある

#### ▶︎準備

1. `📁 seeds`フォルダに次のファイルを配置する
   - [raw_customers.csv](https://github.com/dbt-labs/dbt-core/blob/main/tests/fixtures/jaffle_shop_data/raw_customers.csv)
   - [raw_orders.csv](https://github.com/dbt-labs/dbt-core/blob/main/tests/fixtures/jaffle_shop_data/raw_orders.csv)
   - [raw_payments.csv](https://github.com/dbt-labs/dbt-core/blob/main/tests/fixtures/jaffle_shop_data/raw_payments.csv)
2. `seed`コマンドを実行してサンプルデータを実体化する

#### ▶︎SQLモデル

:knife: つかい方

- `.sql`ファイルは1つのモデルとして1つの`select` 文で構成される
- ファイル名がモデル名として使用される
- モデルは `models`ディレクトリ内のサブディレクトリに入れ子にすることができる

:point_up: 準備したデータから顧客の`初回注文日`,`直近の注文日`,`注文数`を確認するためのテーブルを実体化する

##### 1.`📁 models`ディレクトリに次のSQLを配置する

<!-- markdownlint-disable MD033 -->
<details><summary>:spiral_notepad: /models/cutomers.sql（クリックすると展開されます）</summary>

```sql
with customers as (
    select
        id as customer_id,
        first_name,
        last_name
    from raw_customers
),
orders as (
    select
        id as order_id,
        user_id as customer_id,
        order_date,
        status
    from raw_orders
),
customer_orders as (
    select
        customer_id,
        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders
    from orders
    group by customer_id
),
final as (
    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders
    from customers
    left join customer_orders using (customer_id)
)
select * from final
```

</details>
<!-- markdownlint-enable MD033 -->

##### 2. `run`コマンドで実体化する

実体化のオプションを何も設定しないと`VIEW`が作成される

```sh
dbt run --select customers

# Running with dbt=1.3.1
# Found 3 models, 4 tests, 0 snapshots, 0 analyses, 289 macros, 0 operations, 3 seed files, 0 sources, 0 exposures, 0 metrics
# Concurrency: 1 threads (target='dev')
# 1 of 1 START sql view model public.customers ................................... [RUN]
# ✅
# 1 of 1 OK created sql view model public.customers .............................. [CREATE VIEW in 0.30s]
# Finished running 1 view model in 0 hours 0 minutes and 0.63 seconds (0.63s).
# ✅
# Completed successfully
# Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

<!-- markdownlint-disable MD033 -->
<details>
<summary>:spiral_notepad: dbtのログ（クリックすると展開されます）</summary>

```log
[0m06:49:02.989399 [debug] [Thread-1  ]: On model.jaffle_shop.customers: /* {"app": "dbt", "dbt_version": "1.3.1", "profile_name": "jaffle_shop", "target_name": "dev", "node_id": "model.jaffle_shop.customers"} */

  create view "jaffle_shop"."public"."customers__dbt_tmp" as (
    with customers as (
    select
        id as customer_id,
        first_name,
        last_name
    from raw_customers
),
orders as (
    select
        id as order_id,
        user_id as customer_id,
        order_date,
        status
    from raw_orders
),
customer_orders as (
    select
        customer_id,
        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders
    from orders
    group by customer_id
),
final as (
    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders
    from customers
    left join customer_orders using (customer_id)
)
select * from final
  );
[0m06:49:03.030024 [debug] [Thread-1  ]: SQL status: CREATE VIEW in 0.04 seconds
[0m06:49:03.041935 [debug] [Thread-1  ]: Using postgres connection "model.jaffle_shop.customers"
[0m06:49:03.042278 [debug] [Thread-1  ]: On model.jaffle_shop.customers: /* {"app": "dbt", "dbt_version": "1.3.1", "profile_name": "jaffle_shop", "target_name": "dev", "node_id": "model.jaffle_shop.customers"} */
alter table "jaffle_shop"."public"."customers__dbt_tmp" rename to "customers"
```

</details>
<!-- markdownlint-enable MD033 -->

##### 3. 実体化したビューを確認する

#### ▶︎Pythonモデル

- SQLモデルでは解決できないユースケースを解決するために利用する
- `models`ディレクトリに`.py`ファイルを配置する
- ファイル名がモデル名として使用される
- `model(dbt, session)`関数を定義する

### ●シードファイル

### ●パッケージ

### ●テスト

### ●ドキュメント

### ●スナップショットデータ

## ■参考

- 読む順
  1. 概要把握
     - [データエンジニア界隈で話題のdbt（data build tool）のまとめ](https://qiita.com/manabian/items/67af7e4476d436aded77)
     - [データエンジニアリングの背景を踏まえてdbt（Data Build Tool）を少し深く理解してみる](https://qiita.com/manabian/items/ab6c2a5dde78c432ecf1)
  1. チュートリアル
     - [dbtとは？](https://zenn.dev/dbt_tokyo/books/537de43829f3a0/viewer/)
  1. その他
     - [dbt Cloudとdbt-core (CLI)の違いを整理してみた](https://dev.classmethod.jp/articles/20220223-dbt-cloud-cli-diff/)
  1. 公式
     - [公式のドキュメント](https://docs.getdbt.com/)
