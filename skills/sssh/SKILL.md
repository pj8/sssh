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

## 前提条件

- AWS CLI / Session Manager plugin / jq / peco がインストール済みであること
- 使用する AWS CLI プロファイルの output 形式が `json` であること
- 有効期限内の AWS 認証情報があること

## シンプルな使い方

```bash
# プロファイル、クラスタ、サービス、タスク、コンテナを対話的に選んで /bin/sh を起動
./sssh

# プロファイルとリージョンを指定
./sssh --profile my-profile --region ap-northeast-1

# MFA（多要素認証）が必要なプロファイルでは OTP（ワンタイムパスワード）を渡す
./sssh --profile my-profile --otp 123456
```

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
パイプ・リダイレクト・変数展開はそのまま書けて、リモートコマンドの終了コードが
`sssh` の終了コードになる（v5.0.0以降）。成否判定が必要な自動処理にも使える。

```bash
# コンテナ上で一回きりのコマンドを実行
./sssh --profile my-profile --cluster my-cluster --service my-service \
       --task "$TASK_ARN" --container app --command "php -v"

# パイプもそのまま書ける（sh -c で包む必要はない）
./sssh --profile my-profile --cluster my-cluster --service my-service \
       --task "$TASK_ARN" --container app --command "php -v | head -n 1"

# リモートコマンドの終了コードが伝搬される
./sssh --profile my-profile --cluster my-cluster --service my-service \
       --task "$TASK_ARN" --container app --command 'exit 7' ; echo $?  # => 7
```

情報メッセージ（日時、選択結果、実行コマンドライン）はすべて stderr に出力され、
stdout にはリモートコマンドの出力だけが流れる。そのため
`./sssh --command 'php -v' | grep PHP` のようなパイプ処理がそのまま動く。

## ポートフォワード

`--remote-host` `--remote-port` `--local-port` を指定すると、選択したコンテナ経由で
リモートホストへのポートフォワードを開始する。

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
| `-o`, `--otp` | MFA 用のワンタイムパスワード |
| `--cluster` | クラスタ名 |
| `--service` | サービス名 |
| `--task` | タスク名（ARN） |
| `--container` | コンテナ名 |
| `--remote-host` | ポートフォワード先のリモートホスト名 |
| `--remote-port` | ポートフォワード先のリモートポート番号 |
| `--local-port` | ポートフォワードで使うローカルポート番号 |
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
