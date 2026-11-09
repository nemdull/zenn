---
title: "Mac mini で Claude Code を3並列の常駐ワーカーにする — Docker とファイルキュー、そして最初の3回の失敗"
emoji: "🤖"
type: "tech"
topics: ["claude", "claudecode", "docker", "ai", "shell"]
published: false
---

## モチベーション

「Claude Code に頼んだ作業が終わるまで、ターミナルの前で待っているのが勿体ない」

そう思って、常時起動している Mac mini に **Claude Code を 3 つの Docker コンテナで常駐させ、ファイルを置くだけでタスクを拾って処理する仕組み**を作りました。この記事は、その構成と、動くまでに踏んだ 3 回の失敗の記録です。

先に正直に書いておくと、この仕組みは「作って、動くことを確認した」段階です。2026 年 4 月にテストタスクを 4 件流し、3 回失敗して 1 回成功しました。その失敗ログがそのまま残っているので、同じ構成を作る人の時間を節約できると思い、記事にしています。

設計で一番悩んだのは「**複数のワーカーが同じタスクを取り合う競合をどう防ぐか**」でした。Redis もキューサービスも使わずに済む方法が、UNIX の古い仕組みにあります。

## 全体像

```
submit-task.sh "やってほしいこと"
        │
        ▼
tasks/pending/20260421_145237_task.txt     ← ファイルを置くだけがタスク投入
        │
   ┌────┼────┐
worker-1 worker-2 worker-3                 ← 同じ tasks/ を見ている 3 コンテナ
        │
        ▼
mv pending/X.txt running/worker-2_X.txt    ← mv に成功した 1 台だけが処理する
        │
        ▼
claude --print --max-turns 20 --output-format text "$(cat X.txt)"
        │
        ├─ exit 0 → workspace/output_worker-2_<時刻>.md ＋ done/<時刻>_worker-2_X.txt
        └─ exit≠0 → 同じ出力ファイル（エラー内容）＋ failed/<時刻>_worker-2_X.txt
```

ディレクトリは次の形です（ホスト側）。

```
claude-agent/
├── docker-compose.yml
├── Dockerfile
├── docker-entrypoint.sh
├── .claude/settings.json      # Claude Code の権限（allow / ask / deny）とフック登録
├── hooks/
│   ├── pre-tool.sh            # ツール実行前：危険パターンをブロック
│   └── post-tool.sh           # ツール実行後：監査ログ
├── scripts/
│   ├── worker.sh              # ワーカー本体
│   ├── submit-task.sh         # タスク投入
│   └── queue-status.sh        # キューの状況表示
├── tasks/{pending,running,done,failed}/
└── volumes/
    ├── secrets.env            # API キー（git 管理外、read-only でマウント）
    ├── workspace/             # 出力先（3 ワーカーで共有）
    └── logs/
```

**設計の核心は「ファイルシステムをキューにする」こと**です。`pending/` にテキストを置くのが投入、`mv` が状態遷移、`ls` が監視。外部サービスへの依存はゼロです。

## docker-compose — 3 ワーカーは同じ定義の複製

YAML のアンカーで共通部分を 1 回だけ書き、ワーカーごとに ID を変えるだけにしています。

```yaml
# docker-compose.yml
x-worker-base: &worker-base
  build:
    context: .
    dockerfile: Dockerfile
  image: claude-agent:latest
  env_file:
    - ./volumes/secrets.env
  environment:
    - TZ=Asia/Tokyo
    - LANG=ja_JP.UTF-8
  volumes:
    - ./volumes/secrets.env:/home/agent/.config/agent/secrets.env:ro
    - ./volumes/workspace:/home/agent/workspace
    - ./volumes/logs:/home/agent/logs
    - ./tasks:/home/agent/tasks
    - ./scripts:/home/agent/scripts
  restart: unless-stopped

services:
  claude-agent:                 # 手動操作用のシェル
    <<: *worker-base
    container_name: claude-agent
    stdin_open: true
    tty: true
    command: ["bash"]

  worker-1:
    <<: *worker-base
    container_name: claude-worker-1
    command: ["/home/agent/scripts/worker.sh", "worker-1"]

  worker-2:
    <<: *worker-base
    container_name: claude-worker-2
    command: ["/home/agent/scripts/worker.sh", "worker-2"]

  worker-3:
    <<: *worker-base
    container_name: claude-worker-3
    command: ["/home/agent/scripts/worker.sh", "worker-3"]
```

注意点が一つ。`tasks/`・`logs/` だけでなく **`workspace/` も 3 ワーカーで共有**しています。分離したほうが安全ですが、出力を 1 か所で見たかったので共有にし、代わりに出力ファイル名に `worker-ID` と時刻を入れて衝突を避けています。ワーカーが同じリポジトリを同時に編集するような使い方をするなら、ここは専用ボリュームに分けるべきです（後述）。

## worker.sh — mv のアトミック性で競合を防ぐ

```bash
#!/bin/bash
# scripts/worker.sh（抜粋。ログ出力などは省略）
WORKER_ID="${1:-worker-unknown}"
TASK_DIR="/home/agent/tasks"
PENDING_DIR="$TASK_DIR/pending"
RUNNING_DIR="$TASK_DIR/running"
DONE_DIR="$TASK_DIR/done"
FAILED_DIR="$TASK_DIR/failed"
WORKSPACE_DIR="/home/agent/workspace"
POLL_INTERVAL=5

while true; do
    for TASK_FILE in "$PENDING_DIR"/*.txt "$PENDING_DIR"/*.md; do
        [ -f "$TASK_FILE" ] || continue        # glob が展開されなかったときの保険

        TASK_NAME="$(basename "$TASK_FILE")"
        RUNNING_FILE="$RUNNING_DIR/${WORKER_ID}_${TASK_NAME}"

        # ===== 競合防止のキモ =====
        # 同一ファイルシステム上の mv はアトミック。3 台が同時に試みても成功は 1 台だけ。
        # 負けたワーカーは mv が失敗するので、黙って次のファイルへ。
        if ! mv "$TASK_FILE" "$RUNNING_FILE" 2>/dev/null; then
            continue
        fi

        TASK_CONTENT="$(cat "$RUNNING_FILE")"
        TIMESTAMP="$(date '+%Y%m%d_%H%M%S')"
        OUTPUT_FILE="$WORKSPACE_DIR/output_${WORKER_ID}_${TIMESTAMP}.md"

        # --print          : 対話なしで実行し、結果をテキストで標準出力へ
        # --max-turns 20   : ツール呼び出しの往復を 20 回で打ち切る（暴走防止）
        # --output-format  : text（json にすると後処理しやすい）
        if claude \
            --print \
            --max-turns 20 \
            --output-format text \
            "$TASK_CONTENT" \
            > "$OUTPUT_FILE" 2>&1; then
            mv "$RUNNING_FILE" "$DONE_DIR/${TIMESTAMP}_${WORKER_ID}_${TASK_NAME}"
        else
            mv "$RUNNING_FILE" "$FAILED_DIR/${TIMESTAMP}_${WORKER_ID}_${TASK_NAME}"
        fi

        sleep 2   # 連続実行の間隔（レート制限対策）
    done
    sleep "$POLL_INTERVAL"
done
```

POSIX は同一ファイルシステム上の `rename(2)` をアトミックと規定しています。`mv` はその薄い皮なので、**「mv に成功した者がタスクの所有者」**という規則だけで排他が成立します。`flock` も Redis も要りません。

`running/` に移すときにファイル名へワーカー ID を付けているのは、あとで「誰が処理中か」を `ls` で見られるようにするためです。

## 権限の設計 — settings.json とフックの二重防衛

Claude Code は強力なので、コンテナの中とはいえ「できること」を明示的に絞っています。層は 2 つです。

**1 層目：`.claude/settings.json` の `permissions`。** 読み取り系の git／gh／cat／grep などは `allow`、`git push` や `gh pr create`、`npm install`、`curl` は `ask`、`rm -rf`／`sudo`／`git push --force`／`git reset --hard`／`psql` などは `deny`。`Write`／`Edit` は `~/workspace/**` の下だけ許可。秘密情報ファイルと `~/.ssh/*` は `Read` も `deny` です。

**2 層目：PreToolUse フック。** 設定の `deny` をすり抜ける表記（引用符やパスの書き方の違い）に備えて、ツール入力の JSON を文字列として見て危険パターンを拾います。

```bash
#!/bin/bash
# hooks/pre-tool.sh（抜粋）
INPUT=$(cat)                                        # stdin に JSON が来る
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // "unknown"')
TOOL_INPUT=$(echo "$INPUT" | jq -c '.tool_input // {}')

DANGEROUS_PATTERNS=('rm -rf' 'sudo ' 'DROP TABLE' 'FLUSHALL' '--force'
  '/etc/shadow' '/etc/passwd' 'secrets.env' '.ssh/' 'chmod 777'
  'shutdown' 'reboot' 'mkfs' 'dd if=')

INPUT_STR=$(echo "$TOOL_INPUT" | tr '[:upper:]' '[:lower:]')
for pattern in "${DANGEROUS_PATTERNS[@]}"; do
  if echo "$INPUT_STR" | grep -qF -- "$(echo "$pattern" | tr '[:upper:]' '[:lower:]')"; then
    echo "{\"decision\": \"block\", \"reason\": \"Dangerous pattern detected: '$pattern'.\"}"
    exit 0
  fi
done

echo '{"decision": "approve"}'
exit 0
```

フックは **判定を JSON で標準出力に返し、終了コードは 0** にしています。`PostToolUse` 側のフックは全ツール呼び出しを `logs/hooks.log` に記録する監査用で、成功したタスクのログはこうなります。

```
[2026-04-21 14:52:41] [PRE ] tool=Bash input={"command":"echo \"Hello from Claude Agent!\"","description":"Print hello message"}
[2026-04-21 14:52:41] [POST] tool=Bash exit_code=unknown output_len=0chars
[2026-04-21 14:52:43] [PRE ] tool=Bash input={"command":"mkdir -p /home/agent/results && echo \"テスト成功！...\" > /home/agent/results/test-result.txt", ...}
[2026-04-21 14:52:45] [PRE ] tool=Read input={"file_path":"/home/agent/results/test-result.txt"}
```

ツール単位で「何をしたか」が残るので、無人で動かすときの安心材料になります。

**Dockerfile 側の守り**は、`agent` ユーザー（sudo なし）で動かすこと、`secrets.env` を `chmod 600` かつ read-only マウントにすること、の 2 点です。ベースは `ubuntu:22.04`、Node.js 20 で `npm install -g @anthropic-ai/claude-code`、`gh` CLI も入れています。

## 最初の 3 回の失敗（ログそのまま）

ここが一番役に立つ部分だと思います。`logs/worker-worker-2.log` から抜粋します。

**1 回目（14:40）— 存在しないフラグ**

```
[2026-04-21 14:40:23] [worker-2] ▶ タスク取得: test-001.txt
[2026-04-21 14:40:23] [worker-2] 🤖 Claude Code 実行中...
[2026-04-21 14:40:23] [worker-2] ❌ 失敗 (exit=1): test-001.txt
[2026-04-21 14:40:23] [worker-2] エラー出力: error: unknown option '--no-interactive'
```

最初の `worker.sh` には `claude --print --no-interactive ...` と書いていました。**`--no-interactive` というフラグは Claude Code に存在しません。** `--print`（`-p`）がそもそも非対話モードです。worker-3 も同じタスク種別で同時に同じエラーを出していて、「3 台が同時に同じ間違いをする」のが並列化の怖さだと実感しました。

**2 回目（14:48）— API キーがワーカーに届いていない**

```
[2026-04-21 14:48:06] [worker-2] ❌ 失敗 (exit=1): test-003.txt
[2026-04-21 14:48:06] [worker-2] エラー出力: Not logged in · Please run /login
```

`env_file` で `ANTHROPIC_API_KEY` をコンテナに渡していたのに、ログインしていないと言われる。原因は `docker-entrypoint.sh` で `su - agent -c "worker.sh"` と**ログインシェルに切り替えていた**ことでした。`su -` は環境を作り直すので、root 側で読み込んだ環境変数が `agent` ユーザーに引き継がれません。

直し方は、entrypoint で `secrets.env` を読んだあと、`su` の内側で明示的に `export` してから `worker.sh` を起動するようにしました。

```bash
# docker-entrypoint.sh（抜粋）
set -a; source "$SECRETS_FILE"; set +a
# ...
elif [[ "$1" == *"worker.sh"* ]]; then
    exec su - agent -c "export ANTHROPIC_API_KEY=... bash /home/agent/scripts/worker.sh $2"
```

**3 回目（14:52）— 成功、13 秒**

```
[2026-04-21 14:52:37] [worker-2] ▶ タスク取得: test-004.txt
[2026-04-21 14:52:37] [worker-2] 🤖 Claude Code 実行中...
[2026-04-21 14:52:50] [worker-2] ✅ 完了: test-004.txt → /home/agent/workspace/output_worker-2_20260421_145237.md
```

出力ファイルには、Claude がコマンドを実行して結果を表にまとめたものが入っていました。

```
| ステップ | 結果 |
|---|---|
| `echo "Hello from Claude Agent!"` | `Hello from Claude Agent!` |
| `date` | `Tue Apr 21 14:52:41 JST 2026` |
| ファイル作成 | `/home/agent/results/test-result.txt` に書き込み成功 |
```

このとき worker-3 は何も拾っていません。4 件のタスクは「置いた瞬間に空いていたワーカー」が 1 台ずつ `mv` で取っていて、**取り合いの競合は一度も起きませんでした**（起きていれば `mv` の失敗がログに出ます）。

## 使い方

```bash
# 起動
docker compose up -d

# タスク投入（コンテナ内、または ./tasks/pending に直接ファイルを置く）
submit-task.sh "sf-meter-api の README を、コードブロックを変えずに英語化して"
submit-task.sh -f my_task.txt
submit-task.sh --interactive        # エディタで書く

# 状況
queue-status.sh                      # pending/running/done/failed の件数
tail -f volumes/logs/worker-*.log    # ワーカーのログ
```

タスクはプレーンテキストです。JSON にしなかったのは、`echo "..." > tasks/pending/$(date +%s).txt` でも投入できる気軽さを優先したからです。

## いまの限界と、次にやること

正直な現状を書きます。

- **実稼働はテストタスク 4 件まで。** 「夜間に回したら朝にはドキュメントが英語化されていた」は、まだ体験していません。この記事はその手前の記録です
- **ワークスペースが共有。** リポジトリを編集するタスクを複数同時に流すなら、ワーカーごとの named volume に分ける必要があります
- **クラッシュ時の回収がない。** ワーカーが途中で落ちると `running/` にファイルが残ります。起動時に一定時間以上古い `running/*` を `pending/` に戻す処理を足す予定です（`mv` で戻すだけなので数行）
- **`ask` の権限に当たると止まる可能性。** `--print` モードでは確認を返せないので、`settings.json` の `allow` を読み取り系について広めに取っています。`--dangerously-skip-permissions` は使っていません
- **ポーリングは 5 秒間隔。** `inotifywait` でイベント駆動にすれば待ち時間とログのノイズが減ります
- **API キーの扱い。** コンテナ内の環境変数と read-only マウントに留めていますが、`ps` で見える位置に露出する瞬間はあります。次は Docker secrets に移す予定です

リポジトリは機密設定ファイルを同じディレクトリに含むため公開していません。主要なファイルは本文に載せたとおりです。

## まとめ

- タスクキューは **ディレクトリ 4 つと `mv`** で足ります。`rename(2)` のアトミック性が排他を保証してくれるので、ミドルウェアは要りません
- Claude Code を無人で動かすときの守りは、**`settings.json` の allow/ask/deny ＋ PreToolUse フックの文字列検査 ＋ sudo なしユーザー**の 3 段にすると落ち着きます
- つまずきは「存在しないフラグ」と「`su -` で環境変数が消える」の 2 つ。どちらもログに一行で出るので、**ワーカーの stderr を必ずファイルに残す**設計にしておくと 10 分で直せます

「Claude Code にタスクを置いて、別のことをしに行く」という体験は、たしかに作業のリズムを変えます。ただし今回の 4 件はあくまで動作確認で、本番の価値は、これから流すタスクで測るつもりです。
