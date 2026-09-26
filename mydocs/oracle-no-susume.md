# Oracleのすすめ

## ブラウザ版ChatGPT をCLIから使い、Codexを実装に集中させる

作成日: 2026-09-24  

---

## まず30秒で

**Oracleは、ブラウザ版ChatGPTをCLIから使えるようにするOSSツールです。**

ターミナルから質問すると、Oracleがログイン済みのChatGPTをブラウザ経由で操作し、回答をターミナルへ返してくれます。

```fish
oracle -p "この設計の問題点をレビューして"
```

ブラウザを手で開いて、質問を貼り付け、回答をコピーする作業をCLIから行えるイメージです。

> OracleはOpenAI公式製品ではなく、第三者が開発しているオープンソースソフトウェアです。

---

## Oracleとは何か

Oracleは、プロンプトと必要なファイルをまとめてAIへ送り、回答をセッションとして保存するCLI／MCPサーバーです。

主に2つの実行方法があります。

- **Browserモード**: ログイン済みのChatGPTをChrome経由で操作する
- **APIモード**: OpenAIなどのAPIを直接利用する

今回紹介するのは、APIキーを使わない **Browserモード** です。

```text
ターミナル
   ↓ 質問
Oracle
   ↓ Chromeを自動操作
ChatGPT Web
   ↓ 回答
ターミナル + ローカルセッション + ChatGPTの履歴
```

---

## Oracleを使うと嬉しいこと

### 1. Chatの大きな利用枠を活用し、Codexの利用枠を温存できる

ブラウザ版ChatGPTのヘッダーでは、**Chat** と **Work** を切り替えられます。

- **Chat**: 質問、壁打ち、要約、比較、文章作成などの通常の会話
- **Work**: 調査や資料作成など、成果物まで進めるエージェント作業
- **Codex**: コード調査、実装、テストなどの開発作業

OpenAIの公式ドキュメントでは、**WorkとCodexは利用枠を共有する**と明記されています。一方、通常のChatはその共有対象とは別の会話モードです。

Oracleはプロンプト送信前にChat／Workの状態を確認し、**通常のChat側へ切り替えてから送信**します。Workの利用枠ではなく、Chat側の会話として処理されるのが重要なポイントです。

```text
Chat  ← Oracleが使う
  └─ 通常のChat枠

Work ─┐
      ├─ 利用枠を共有
Codex ┘
```

Proプランでは通常のChatを非常に多く利用でき、日常的なテキスト相談なら**実質ほぼ無制限に近い感覚**で使えます。

そのため、次のような役割分担ができます。

| Oracleに任せる | Codexに任せる |
|---|---|
| 要件や設計の壁打ち | リポジトリの調査 |
| 実装方針の比較 | ファイル編集 |
| 長文の要約・分析 | テスト・ビルド |
| コードや計画のセカンドレビュー | 修正と動作確認 |

**「考える・相談する・レビューする」はOracle、  
「実際に変更して検証する」はCodex** と分けることで、Codexの利用枠を実装作業へ集中させられます。

つまりOracleは、**Chatの大きな利用枠を、開発時の壁打ちやセカンドレビューへ持ち込む橋渡し役**です。

### 2. CodexのWeekly limitを使い切ったときの避難先になる

CodexのWeekly limitに到達しても、通常のChat枠を使うOracleなら、設計相談・原因分析・コードレビュー・次の実装方針づくりを継続できます。

```text
CodexのWeekly limitに到達
  ↓
Oracleで設計・分析・レビューを進めておく
  ↓
利用枠のリセット後、整理済みの方針をCodexへ渡す
  ↓
Codexはすぐ実装・テストから再開できる
```

OracleだけでCodexの実装作業を完全に置き換えるものではありませんが、**利用枠が戻るまで開発を止めないための一時的なバックアップ**として使えます。

### 3. ターミナルからすぐ相談できる

ブラウザへのコピー＆ペーストが不要です。シェル履歴にコマンドが残るため、同じ質問を少し変えて再実行するのも簡単です。

### 4. モデルと思考の強さを指定できる

普段は軽め、難しい設計判断だけ深く考えさせる、といった使い分けができます。

### 5. 回答と会話履歴が残る

- Oracle側: `~/.oracle/sessions/` にプロンプト・回答・実行ログを保存
- ChatGPT側: 設定によりサイドバーの会話履歴にも残せる

---

## 初回ログイン

初回だけ、Oracle専用のChromeプロファイルでChatGPTへログインします。

```fish
oracle --engine browser --browser-manual-login --browser-keep-browser --browser-input-timeout 120000 -p "こんにちは！"
```

1. OracleがChromeを起動する
2. 表示された画面でChatGPTへログインする
3. ログイン状態が `~/.oracle/browser-profile` に保存される
4. 2回目以降はログイン状態を再利用する

---

## `~/.oracle/config.json` の設定例

毎回長いオプションを書かなくて済むように、デフォルト設定を保存します。

```json5
{
  // APIではなく、ログイン済みChatGPTを使うBrowserモードで実行する
  engine: "browser",

  // ChatGPT上で使用するモデル
  model: "gpt-5.6",

  browser: {
    // Oracle専用のChromeプロファイルを使い、初回ログイン状態を保存・再利用する
    manualLogin: true,

    // 思考の強さをExtra Highにする（xhighのブラウザ版設定名）
    thinkingTime: "extra-high",

    // プロンプト入力欄が操作可能になるまで最大120秒待つ
    inputTimeoutMs: 120000,

    // ページ遷移などで接続が切れた場合、5秒後から自動再接続を始める
    autoReattachDelayMs: 5000,

    // 自動再接続を3秒間隔で試す
    autoReattachIntervalMs: 3000,

    // 1回の自動再接続処理を最大60秒待つ
    autoReattachTimeoutMs: 60000,

    // ChatGPTの通常サイドバーに会話を残す
    archiveConversations: "never",
  },
}
```

設定後は、これだけで利用できます。

```fish
oracle -p "こんにちは！"
```

内部ではChromeが自動操作されますが、普段はブラウザを手で操作する必要がありません。

---

## 通常のコマンド

### シンプルな質問

```fish
oracle -p "この設計のメリットとリスクを整理して"
```

### コードを渡してレビュー

```fish
oracle -p "この実装の問題点を重要度順にレビューして" --file "src/**/*.ts" --file "!**/*.test.ts"
```

### 送信前に対象とトークン量を確認

```fish
oracle --dry-run summary --files-report -p "この実装をレビューして" --file "src/**/*.ts"
```

### 過去の実行を確認

```fish
oracle status --hours 72
oracle session <セッションID>
```

---

## モデルと思考の強さを指定する

一時的に指定する場合は、コマンドへ追加します。

```fish
oracle --model gpt-5.6 --browser-thinking-time extra-high -p "複数案を比較し、最も安全な設計を提案して"
```

Browserモードでよく使う指定は次のとおりです。

| Oracleの指定 | ChatGPT上の目安 | 用途 |
|---|---|---|
| `standard` | Medium / Standard | 日常的な質問 |
| `extended` | High | 設計・レビュー |
| `extra-high` | Extra High | 難しい比較・深い分析 |
| `pro` | Pro | 特に難しい問題。時間・利用枠に注意 |

補足:

- Browserモードでは `extra-high` が正式な設定値
- `xhigh` も別名として使え、内部で `extra-high` に変換される
- APIモードの推論強度では `--reasoning-effort xhigh` を使う
- 強くするほど、一般に回答までの時間と消費量が増える

---

## 標準入力からプロンプト本文を読む

改行を含む長文をファイルに書き、その内容をプロンプト本文として送れます。

```fish
oracle -p - < prompt.md
```

これはファイル添付ではありません。`prompt.md` の内容が、改行を保ったままChatGPTの入力欄へ入ります。

送信前に確認する場合:

```fish
oracle --dry-run full --render-plain -p - < prompt.md
```

`oracle -p prompt.md` ではファイルを読み込まず、`prompt.md` という文字列を質問してしまうので注意してください。

---

## おすすめの使い分け

```text
1. Oracleに要件・設計・懸念点を相談する
2. 方針を人間が確認する
3. Codexに実装とテストを依頼する
4. 必要ならOracleにセカンドレビューを頼む
5. 最終判断は人間が行う
```

Oracleは「もう一人の実装者」というより、**CLIから呼べる壁打ち相手・レビュアー**として使うと分かりやすいです。

---

## 注意点

- OracleはOpenAI公式ツールではなく、ChatGPTの画面を自動操作する第三者製OSS
- ChatGPTのUI変更によって、一時的に動かなくなる可能性がある
- プロンプトや `--file` で渡した内容はChatGPTへ送信される
- 機密情報、個人情報、秘密鍵、`.env` などを送らない
- 業務利用では、社内規定と契約中のChatGPTワークスペース設定を確認する

---

## まとめ

> **相談・設計・レビューはOracle、実装・検証はCodex。**  
> AIの利用枠と役割を分けて、Codexを「コードを完成させる仕事」に集中させる。

最初の一歩はこれだけです。

```fish
oracle -p "今抱えている技術課題を、選択肢とリスクに分けて整理して"
```

---

## デモで見せるなら

1. ターミナルで次を実行する

   ```fish
   oracle -p "こんにちは！"
   ```

2. 回答がターミナルへ戻るところを見せる
3. ChatGPTのサイドバーにも同じ会話が残っていることを見せる
4. 時間があれば `oracle -p - < prompt.md` も見せる

---

## 参考リンク

- [Oracle GitHub](https://github.com/steipete/oracle)
- [Oracle documentation](https://askoracle.sh/)
- [OpenAI Docs: Use ChatGPT — Chat / Work / Codexの違い](https://learn.chatgpt.com/docs/use-chatgpt)
- [OpenAI Docs: Pricing — WorkとCodexの共有利用枠](https://learn.chatgpt.com/docs/pricing)
- [OpenAI Docs: ChatGPT on the web](https://learn.chatgpt.com/docs/web)
- [OpenAI Docs: Models and reasoning](https://learn.chatgpt.com/docs/models)
- [OpenAI Docs: Prompting](https://learn.chatgpt.com/docs/prompting)
- [ChatGPT pricing](https://chatgpt.com/pricing)
