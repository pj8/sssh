---
name: sssh
description: Amazon ECS Fargate のコンテナへ ECS Exec で接続するとき、コンテナ上でシェル起動や一回きりのコマンド実行をするとき、またはコンテナ経由でリモートホスト（RDSなど）へポートフォワードするときに、このリポジトリの ./sssh を使うために参照する。
---

# sssh

## 概要

`sssh` は Amazon ECS Fargate のコンテナに ECS Exec で接続するための Bash スクリプト。
プロファイル・クラスタ・サービス・タスク・コンテナを選び、シェル起動・コマンド実行・ポートフォワードを行う。

引数で指定しなかった項目は peco による対話選択になる（候補が1つなら自動選択される）。
Coding Agent が非対話で実行する場合は、必要な項目をすべて引数で指定すること。
MFA が必要な場合は、先に対話端末の AWS CLI 等で認証を済ませ、有効な認証情報を用意する。

## 前提条件

- AWS CLI / Session Manager plugin / jq / peco がインストール済みであること
- `--otp` でクラスタ一覧を取得する場合は `/usr/bin/expect` が必要
- 使用する AWS CLI プロファイルの output 形式が `json` であること
- 有効期限内の AWS 認証情報があること

## シンプルな使い方

```bash
# プロファイル、クラスタ、サービス、タスク、コンテナを対話的に選んで /bin/sh を起動
./sssh

# プロファイルとリージョンを指定
./sssh --profile my-profile --region ap-northeast-1

# クラスタ一覧取得時の MFA 入力に OTP を渡す（--cluster は省略）
./sssh --profile my-profile --otp 123456
```

`--otp` は `--cluster` 省略時のクラスタ一覧取得でのみ使われ、
`--cluster` を指定すると無視される。上の例は認証後に対話選択・シェル起動へ進むため、
非対話実行の認証準備には対話端末の AWS CLI 等を使う。

## クラスタ・サービス・コンテナを指定した接続

```bash
./sssh --profile my-profile \
       --cluster my-cluster \
       --service my-service \
       --container app
```

- `--opt value` と `--opt=value` のどちらの形式も使える。
- `--task` を省略した場合、RUNNING のタスクが複数あると対話選択になる。
  完全に非対話で実行するには `--task` も指定する。タスクARNは次のように取得して
  変数に入れておくと、以降の例でそのまま使える。

```bash
TASK_ARN=$(aws ecs list-tasks --profile my-profile --cluster my-cluster \
    --service-name my-service --desired-status RUNNING --output json | jq -r '.taskArns[0]')
```

## コマンド実行

`--command` を指定すると、コマンドは `/bin/sh -c ...` でラップされて実行される。
パイプ・リダイレクトはそのまま書ける。リモート側で変数展開する場合は、
コマンド全体を単一引用符で囲むか、`$` をエスケープする。
`--command "echo $HOSTNAME"` ではローカルシェルが先に変数を展開してしまう。
リモートコマンドの終了コードが
`sssh` の終了コードになる（v5.0.0以降）。成否判定が必要な自動処理にも使える。

```bash
# コンテナ上で一回きりのコマンドを実行
./sssh --profile my-profile --cluster my-cluster --service my-service \
       --task "$TASK_ARN" --container app --command "php -v"

# パイプもそのまま書ける（sh -c で包む必要はない）
./sssh --profile my-profile --cluster my-cluster --service my-service \
       --task "$TASK_ARN" --container app --command "php -v | head -n 1"

# リモート側の環境変数を展開する（ローカルでは展開しない）
./sssh --profile my-profile --cluster my-cluster --service my-service \
       --task "$TASK_ARN" --container app --command 'echo "$HOSTNAME"'

# リモートコマンドの終了コードが伝搬される
./sssh --profile my-profile --cluster my-cluster --service my-service \
       --task "$TASK_ARN" --container app --command 'exit 7' ; echo $?  # => 7
```

情報メッセージ（日時、選択結果、実行コマンドライン）はすべて stderr に出力され、
stdout にはリモートコマンドの出力と AWS CLI / Session Manager 由来のメッセージが流れる。
上の全項目指定のコマンドに `| grep PHP` などを付けて、ローカルで出力を絞り込める。

## ポートフォワード

`--remote-host` `--remote-port` `--local-port` のいずれかを指定すると、
ポートフォワードの処理に入る。`--local-port` は必須で、未指定だと失敗する。
`--remote-host` の省略時は `127.0.0.1`（コンテナ側）、`--remote-port` の省略時は `3306` を使う。

```bash
# localhost:13306 -> rds.example.com:3306
./sssh --profile my-profile --cluster my-cluster --service my-service \
       --task "$TASK_ARN" --container app \
       --remote-host rds.example.com --remote-port 3306 --local-port 13306
```

セッションは前面で動き続けるため、別プロセスから接続確認する場合はバックグラウンド実行を検討する。

## オプション一覧

| オプション | 説明 |
|---|---|
| `-p`, `--profile` | AWS プロファイル |
| `-r`, `--region` | AWS リージョン（省略時はプロファイルの設定を使用） |
| `-c`, `--command` | 実行するコマンド（デフォルト: `/bin/sh`） |
| `-o`, `--otp` | クラスタ一覧取得時の MFA 用 OTP（`--cluster` 指定時は無視） |
| `--cluster` | クラスタ名 |
| `--service` | サービス名 |
| `--task` | タスク名（ARN） |
| `--container` | コンテナ名 |
| `--remote-host` | ポートフォワード先のホスト名（省略時: `127.0.0.1`） |
| `--remote-port` | ポートフォワード先のポート番号（省略時: `3306`） |
| `--local-port` | ポートフォワードで使うローカルポート番号（ポートフォワード時は必須） |
| `-h`, `--help` | ヘルプ表示 |
| `-v`, `--version` | バージョン表示 |

## 注意事項

- 対話シェル（`--command` なし）の場合、シェルの終了コードは `sssh` に反映されない
  （ECS Exec の制限。終了コード伝搬は `--command` 指定時のみ）。
- ECS Exec は SSM エージェント内の PTY（擬似端末）でコマンドを実行するため、
  stdout はテキスト前提（`\n` が `\r\n` に変換されることがある）。tar や gzip などの
  バイナリ出力は壊れるので、ファイル転送は S3 などを経由する。
- `--command` にパスワードやトークンを含めない。コマンドラインは画面表示・シェル履歴・
  CloudTrail に残り、ECS Exec のログ設定によっては CloudWatch Logs / S3 にも記録される。
- プロファイルの output 形式が `json` 以外だとスクリプトがクラッシュする。
