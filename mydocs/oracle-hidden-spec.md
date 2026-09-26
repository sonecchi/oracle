# `oracle-hidden` 仕様メモ

最終更新: 2026-09-25

## 概要

`oracle-hidden` は、Oracle の browser モードを Ubuntu の物理デスクトップに Chrome を表示せず実行するための fish 関数。

Chrome 自体を headless モードにするのではなく、Xvfb が提供するメモリ上の仮想 X11 画面で通常の headful Chrome を動かす。

```text
通常実行:
Oracle → Chrome → Wayland / XWayland → 物理モニター

oracle-hidden:
Oracle → 通常のChrome → Xvfb → メモリ上の仮想画面
```

この方式にした理由は、Oracle の `--browser-headless` では ChatGPT 側の Cloudflare challenge に阻まれる場合があるため。Xvfb 上では Chrome は通常モードのまま動き、描画先だけが物理モニターから分離される。

## 配置場所

```text
~/.config/fish/functions/oracle-hidden.fish
```

fish の autoload 対象なので、通常は `config.fish` への追記や `source` は不要。新しい fish セッションでも `oracle-hidden` をそのまま呼び出せる。

## 前提

- Ubuntu + fish shell
- `oracle` コマンドが利用可能であること
- `xvfb-run` が利用可能であること
- Oracle の browser モードで ChatGPT にログイン済みであること
- モデルや思考強度を省略する場合は、`~/.oracle/config.json` に既定値が設定されていること

Xvfb が未導入の場合のインストール例:

```fish
sudo apt install xvfb
```

## コマンド仕様

### 書式

```text
oracle-hidden [-m MODEL] [-t LEVEL] -p PROMPT [-- ORACLE_OPTIONS...]
oracle-hidden [-m MODEL] [-t LEVEL] -P PROMPT_FILE [-- ORACLE_OPTIONS...]
```

### ラッパー独自のオプション

| 短縮形 | 長い形式 | 必須 | 内容 |
|---|---|---:|---|
| `-m` | `--model MODEL` | 任意 | Oracle に `--model MODEL` として渡す |
| `-t` | `--thinking LEVEL` | 任意 | 思考強度。例: `light`, `standard`, `extended`, `extra-high`, `pro`, `heavy` |
| `-p` | `--prompt PROMPT` | 条件付き必須 | プロンプト本文を文字列で直接指定する |
| `-P` | `--prompt-file FILE` | 条件付き必須 | ファイルの内容をプロンプト本文として標準入力から渡す |
| `-h` | `--help` | 任意 | ヘルプを表示して終了する |

`--prompt` と `--prompt-file` のどちらか一方は必須で、同時指定はできない。

`--model` と `--thinking` を省略した場合、この関数は対応する Oracle オプションを追加しない。そのため、Oracle 側の `~/.oracle/config.json` にある値が使われる。

`--thinking` の値は関数内では検証せず、Oracle に `--browser-thinking-time LEVEL` として渡す。指定できる正規値の例は次のとおり。

| 値 | ChatGPT UI上のおおよその強度 |
|---|---|
| `light` | Instant / Quick |
| `standard` | Medium / Standard |
| `extended` | High / Extended |
| `extra-high` | Extra High |
| `pro` | Pro |
| `heavy` | Heavy（対応モデル・UIに存在する場合） |

現在の GPT-5.6 Sol で Extra High を指定する例:

```fish
oracle-hidden -t extra-high -p "じっくり考えて回答して"
```

Oracle はUI寄りの別名も受け付ける。たとえば `instant` / `low` は `light`、`medium` は `standard`、`high` は `extended`、`extra high` / `extrahigh` / `xhigh` は `extra-high` として扱われる。メモやスクリプトでは、挙動を読み取りやすくするため正規値の使用を推奨する。

指定した強度が実際に選択されたかは、Oracleの実行ログにある `requestedLevel`、`resolvedLabel`、`verified=yes` で確認できる。モデルやChatGPT UIによって利用できない強度もある。

### Oracle 本体へ追加オプションを渡す

`--` より後ろの引数は、Oracle 本体へそのまま追加する。

```fish
oracle-hidden -p "このコードをレビューして" -- --file "src/**/*.ts"
```

ここでは次のように役割が分かれる。

- `-p "このコードをレビューして"`: `oracle-hidden` が受け取るプロンプト本文
- `--`: ラッパー用引数と Oracle 用引数の区切り
- `--file "src/**/*.ts"`: Oracle 本体が受け取る追加オプション

Oracle の追加オプションを区切りなしで渡すと、fish の `argparse` が `oracle-hidden` 用の未知のオプションとして拒否する可能性があるため、必ず `--` を付ける。

## 使用例

### 設定ファイルのモデルと思考強度を使う

```fish
oracle-hidden -p "こんにちは！"
```

### モデルと思考強度を明示する

```fish
oracle-hidden \
    --model gpt-5.6 \
    --thinking extra-high \
    --prompt "この設計の問題点をレビューして"
```

短縮形でも同じ。

```fish
oracle-hidden -m gpt-5.6 -t extra-high -p "この設計の問題点をレビューして"
```

### ファイルの内容を長文プロンプトとして送る

```fish
oracle-hidden -P prompt1.md
```

モデルと思考強度も指定する場合:

```fish
oracle-hidden \
    -m gpt-5.6 \
    -t extra-high \
    -P prompt1.md
```

この `-P` はファイル添付ではない。内部では次の形に変換され、ファイル内容がプロンプト本文として送られる。

```fish
oracle -p - < prompt1.md
```

本文中の改行や空行は保持される。ただし Oracle 側で標準入力全体に `trim()` が適用されるため、本文の先頭と末尾にある余分な空白・改行は除去される。

### 長文プロンプトと参照ファイルを分けて渡す

```fish
oracle-hidden -P review-prompt.md -- --file "src/**/*.ts"
```

- `review-prompt.md`: プロンプト本文
- `src/**/*.ts`: Oracle の参照ファイル

## 保存済みセッションの会話を続ける

Oracleには、以前のセッションと同じChatGPT会話へ追加プロンプトを送る `--followup` が用意されている。

browserモードでは、単に過去の回答をプロンプトへ貼り直すのではなく、親セッションに保存されたChatGPTの会話URLを開き直し、その会話の続きとして新しいプロンプトを送信する。

### 基本的な流れ

最初の質問では、あとから見つけやすい名前を `--slug` で付けておくと便利。

```fish
oracle-hidden \
    -p "この認証設計をレビューして" \
    -- --slug auth-review
```

後日、同じChatGPT会話へ追加質問する。

```fish
oracle-hidden \
    -p "前回の指摘を踏まえて、修正案を再評価して" \
    -- --followup auth-review
```

`--followup` には、親セッションのIDまたはslugを指定できる。

### 過去のセッションを探す

```fish
oracle status
```

直近1週間まで広げて探す場合:

```fish
oracle status --hours 168
```

見つけたセッションIDまたはslugを使う。

```fish
oracle-hidden \
    -p "この方針の最大の弱点をもう一度考えて" \
    -- --followup SESSION_ID_OR_SLUG
```

### 長文ファイルを追加質問として送る

`-P` も `--followup` と併用できる。

```fish
oracle-hidden \
    -P followup-prompt.md \
    -- --followup auth-review
```

`followup-prompt.md` の内容が、同じChatGPT会話への新しいプロンプト本文になる。ファイル添付ではない。

### 新しい参照ファイルも追加する

```fish
oracle-hidden \
    -p "この実装を追加でレビューして" \
    -- \
    --followup auth-review \
    --file "src/auth/rate-limiter.ts"
```

この場合、会話履歴は既存のChatGPT会話から引き継ぎ、`rate-limiter.ts` は今回のターンに追加する参照ファイルとして送られる。

### 継続時に引き継がれるもの

ChatGPT browserセッションを `--followup` で継続すると、Oracleは親セッションから次の情報を引き継ぐ。

- ChatGPTの会話URL
- browser profileとログイン状態
- browser設定
- モデル
- 思考強度を含む親のbrowser設定

継続時は親セッションのモデルとbrowser設定が優先され、モデル選択をやり直さない。そのため、通常は `oracle-hidden` の `-m / --model` と `-t / --thinking` を省略する。

また、再開したターンではDeep Researchが無効化され、会話は自動アーカイブされない。

### セッション関連コマンドの違い

| 操作 | 用途 | 同じChatGPT会話へ追加送信するか |
|---|---|---:|
| `oracle session <id>` | 実行中セッションへの再接続、または保存済み結果の表示 | しない |
| `oracle restart <id>` | 元のプロンプトとファイルを使って新しいセッションとして再実行 | しない |
| `oracle-hidden -p "..." -- --followup <id>` | 保存されたChatGPT会話を開き直して追加質問 | する |
| `--browser-follow-up "..."` | 1回のOracle実行内で、回答後に予定済みの追加質問を送る | する |

`oracle session <id>` は「会話へ質問を追加するコマンド」ではない。実行状況や保存済み回答を確認するためのコマンド。

### 1回の実行で複数ターンを予約する

最初から追加質問の内容が決まっている場合は、Oracle本体の `--browser-follow-up` を繰り返し指定できる。

```fish
oracle-hidden \
    -p "この移行計画をレビューして" \
    -- \
    --browser-follow-up "前の回答で最も弱い前提を指摘して" \
    --browser-follow-up "以上を踏まえて最終案を出して"
```

各追加プロンプトは、直前の回答が完了したあと、同じChatGPT会話へ順番に送信される。Deep Researchモードでは利用できない。

### 継続できない場合

browserセッションの継続には、次の条件が必要。

- 親セッションに復元可能なHTTPSのChatGPT会話URLが保存されている
- 同じbrowser profileでChatGPTへログインできる
- 再開したページに、親セッションの安定した会話ターンが存在する
- 再開先が保存済みの会話と一致する

URLが復元できない、別の会話へ移動した、以前のターンを確認できない、といった場合、Oracleは誤った会話へ送信せずエラーで停止する。

`--followup` で作成された実行は子セッションとして保存される。さらにその子セッションを指定して、会話を継続することもできる。親子関係は `oracle status` にツリー形式で表示される。

## 実行時の環境

内部では、Oracle を次の環境で起動する。

```fish
xvfb-run -a \
    -s "-screen 0 1280x720x24" \
    env -u WAYLAND_DISPLAY \
    XDG_SESSION_TYPE=x11 \
    GDK_BACKEND=x11 \
    oracle ...
```

各指定の意味:

| 指定 | 意味 |
|---|---|
| `xvfb-run -a` | 空いている仮想 Display 番号を自動選択して Xvfb を起動する |
| `-screen 0` | Xvfb の画面番号 `0` を作る |
| `1280x720` | 仮想画面の解像度 |
| 最後の `x24` | 24 bit の色深度。FPSや画面サイズではない |
| `env -u WAYLAND_DISPLAY` | 親の GNOME Wayland セッション情報を子プロセスから取り除く |
| `XDG_SESSION_TYPE=x11` | 子プロセスへ X11 セッションとして動くよう伝える |
| `GDK_BACKEND=x11` | GDK 系アプリに X11 backend を使わせる |

`env -u WAYLAND_DISPLAY` が特に重要。`xvfb-run` が `DISPLAY` を仮想画面へ差し替えても、親環境の `WAYLAND_DISPLAY=wayland-0` が残っていると、Chrome が Wayland を選び、物理デスクトップへ表示されることがある。

## プロンプトの変換規則

### 直接指定

```fish
oracle-hidden -m gpt-5.6 -t extra-high -p "質問"
```

Oracle へ渡す引数:

```text
--model gpt-5.6
--browser-thinking-time extra-high
-p 質問
```

### ファイル指定

```fish
oracle-hidden -m gpt-5.6 -t extra-high -P prompt1.md
```

Oracle へ渡す引数と標準入力:

```text
引数:
  --model gpt-5.6
  --browser-thinking-time extra-high
  -p -

標準入力:
  prompt1.md の内容
```

## エラー仕様

| 条件 | 終了ステータス | 動作 |
|---|---:|---|
| 引数の形式が不正 | `2` | fish の `argparse` がエラーを表示する |
| `--prompt` と `--prompt-file` を同時指定 | `2` | 排他エラーを表示する |
| どちらのプロンプト指定もない | `2` | 必須指定の案内とヘルプの見方を表示する |
| プロンプトファイルが読めない、またはディレクトリ | `2` | 対象パスを含むエラーを表示する |
| `xvfb-run` が見つからない | `127` | Xvfb のインストールを案内する |
| `oracle` が見つからない | `127` | Oracle コマンドが見つからない旨を表示する |
| Oracle または Xvfb の実行に失敗 | そのコマンドの終了ステータス | 呼び出し元へ失敗を返す |

エラーは標準エラー出力へ表示する。

## 注意点

### `visible Chrome` というログは正常

Oracle のログには次のように表示されることがある。

```text
[browser] Browser control: launch visible Chrome
```

これは Chrome 自体が headful モードであるという意味。表示先は Xvfb の仮想画面なので、物理デスクトップへウィンドウが出ていなければ正常。

### `--browser-headless` は併用しない

この関数の目的は「Xvfb 上で headful Chrome を動かす」こと。`--browser-headless` を追加すると別方式になり、ChatGPT 側で Cloudflare challenge に阻まれる可能性がある。

### 初回ログインが必要な場合

Xvfb 上の画面は普段見えないため、ChatGPT への初回ログインや再認証は通常表示の Oracle で済ませておく。`manualLogin: true` の永続プロファイルを使っていれば、ログイン状態は通常実行と `oracle-hidden` で共有される。

### 終了時の後片付け

`xvfb-run` は Oracle の終了後、起動した Xvfb と一時的な認証情報を片付ける。異常終了時に Xvfb や Chrome が残っていないか調べる場合は、次のように確認できる。

```fish
pgrep -af 'Xvfb|google-chrome|chrome'
```

## 現在の実装

```fish
function oracle-hidden --description 'Xvfbの仮想X11画面でOracleを実行する'
    argparse -n oracle-hidden \
        'h/help' \
        'm/model=' \
        't/thinking=' \
        'p/prompt=' \
        'P/prompt-file=' \
        -- $argv
    or return 2

    if set -q _flag_help
        printf '%s\n' \
            'Usage:' \
            '  oracle-hidden [-m MODEL] [-t LEVEL] -p PROMPT [-- ORACLE_OPTIONS...]' \
            '  oracle-hidden [-m MODEL] [-t LEVEL] -P PROMPT_FILE [-- ORACLE_OPTIONS...]' \
            '' \
            'Options:' \
            '  -m, --model MODEL         Oracleのモデル' \
            '  -t, --thinking LEVEL      ブラウザ版ChatGPTの思考強度' \
            '  -p, --prompt PROMPT        プロンプトを直接指定' \
            '  -P, --prompt-file FILE     ファイル内容をプロンプト本文として使用' \
            '  -h, --help                 このヘルプを表示' \
            '' \
            'Examples:' \
            '  oracle-hidden -m gpt-5.6 -t extra-high -p "こんにちは！"' \
            '  oracle-hidden -m gpt-5.6 -t extra-high -P prompt1.md' \
            '  oracle-hidden -p "レビューして" -- --file "src/**/*.ts"'
        return 0
    end

    if not command -q xvfb-run
        printf '%s\n' 'oracle-hidden: xvfb-run が見つかりません。xvfbをインストールしてください。' >&2
        return 127
    end

    if not command -q oracle
        printf '%s\n' 'oracle-hidden: oracle コマンドが見つかりません。' >&2
        return 127
    end

    if set -q _flag_prompt; and set -q _flag_prompt_file
        printf '%s\n' 'oracle-hidden: --prompt と --prompt-file は同時に指定できません。' >&2
        return 2
    end

    if not set -q _flag_prompt; and not set -q _flag_prompt_file
        printf '%s\n' 'oracle-hidden: --prompt または --prompt-file を指定してください。' >&2
        printf '%s\n' '詳しくは oracle-hidden --help を実行してください。' >&2
        return 2
    end

    if set -q _flag_prompt_file
        if not test -r "$_flag_prompt_file"; or test -d "$_flag_prompt_file"
            printf 'oracle-hidden: プロンプトファイルを読み込めません: %s\n' "$_flag_prompt_file" >&2
            return 2
        end
    end

    set -l oracle_args
    if set -q _flag_model
        set -a oracle_args --model "$_flag_model"
    end
    if set -q _flag_thinking
        set -a oracle_args --browser-thinking-time "$_flag_thinking"
    end
    if set -q _flag_prompt
        set -a oracle_args -p "$_flag_prompt"
    else
        set -a oracle_args -p -
    end
    set -a oracle_args $argv

    if set -q _flag_prompt_file
        command xvfb-run -a \
            -s "-screen 0 1280x720x24" \
            env -u WAYLAND_DISPLAY \
            XDG_SESSION_TYPE=x11 \
            GDK_BACKEND=x11 \
            oracle $oracle_args < "$_flag_prompt_file"
    else
        command xvfb-run -a \
            -s "-screen 0 1280x720x24" \
            env -u WAYLAND_DISPLAY \
            XDG_SESSION_TYPE=x11 \
            GDK_BACKEND=x11 \
            oracle $oracle_args
    end
end
```

## 検証記録

- Xvfb + X11 固定の実コマンドで、物理デスクトップに Chrome が出ないことを確認済み
- 実行中の子プロセスで `DISPLAY=:99`、`XDG_SESSION_TYPE=x11`、`GDK_BACKEND=x11`、`WAYLAND_DISPLAY` なしを確認済み
- GPT-5.6 Sol と Extra High の選択が `verified=yes` になることを確認済み
- `oracle-hidden` の fish 構文、autoload、ヘルプ表示を確認済み
- 直接プロンプトとファイルプロンプトの両方を Oracle の dry-run で確認済み
- `--prompt` と `--prompt-file` の同時指定が拒否されることを確認済み

## 関連メモ

- [Oracleのすすめ](./oracle-no-susume.md)
