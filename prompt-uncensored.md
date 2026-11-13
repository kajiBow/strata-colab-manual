# AI用プロンプト（Uncensored版 / abliterated モデル）

以下をそのまま AI エージェントに渡すと、Google Colab の A100 上に Strata（Qwen3.8-Flash-Next **abliterated** IQ3_S）の API サーバーを構築し、延命策込みで運用開始まで進められます。通常版との違いはモデルの入手と setup の方法だけです（ステップ2）。

> **推奨: このプロンプトは Muse に投げるのがおすすめです。** シェル実行・ファイル操作・長時間の監督プロセスまで担えるエージェント向けに書かれており、このマニュアル自体も Muse の運用実績をもとに作られています。

---

あなたは Google Colab の A100 ランタイム上に、Strata で Qwen3.8-Flash-Next の **アンセンサード（abliterated）版 IQ3_S** を動かし、OpenAI 互換 API として Cloudflare トンネルで公開する作業を行います。以下の手順と制約を厳密に守ってください。

## 環境の前提

- `google-colab-cli`（コマンド名 `colab`）が導入済みで、Google AI Pro アカウントで OAuth 認証済みであること。
- **colab コマンドは必ず全てのプロキシ環境変数を外して実行する**こと（`env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY -u all_proxy`）。プロキシ経由だと割り当てリクエストが 411 エラーで失敗します。
- 認証トークン（keyring）は対話的な実行文脈でのみ有効です。setsid/nohup で切り離したループでは期限切れ後に認証プロンプトが出て死ぬため、取得リトライは必ず自分のプロセスツリー内で回してください。

## パラメータ

| 項目 | 値 |
|---|---|
| セッション名 | `strata<N>`（未使用の番号） |
| GPU | A100 |
| モデル | `alesha-pro/Qwen3.8-Flash-Next-abliterated-GSQ-RCO-Strata-GGUF` の IQ3_S |
| サーバーポート | 8081（8080 は使用中のことがある） |
| コンテキスト (`--max-context`) | 262144 |
| 思考レベル (`reasoning_effort`) | high |
| API キー | 既存の env ファイル（`STRATA_API_KEY`）があれば再利用。無ければ `sk-strata-` + ランダム43文字を生成し、mode 600 の env ファイルにのみ保存する |

## 手順

### 1. A100 取得
`colab new -s <name> --gpu A100` を約20秒間隔で最大10回試行する（1試行は約9分でタイムアウト）。取得できたら GPU 名と RAM を確認する（A100-SXM4-40GB / RAM 83GB が目安）。

### 2. モデルのダウンロードと構築（通常版との差分）
このモデルは setup.py の組み込みファミリーに無いため、GGUF を手動で用意して `--gguf-dir` で渡します。

1. Colab VM 上で `https://github.com/Niko1221/Strata.git` を `/content/Strata` に clone し、`uv` を導入する。
2. GGUF を Hugging Face の resolve URL から VM 内デタッチドでダウンロードする（約35分、計83.62GB）:
   - `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf`（54.82GB）
   - `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf`（28.80GB）
   - リポジトリ内パスは `IQ3_S/` プレフィックス付き。ファイル名は公式 ISTA 版と同一形式。
   - `curl -L -C - --retry 5` でレジューム可能にすること。
3. リポジトリの `SHA256SUMS` で両シャードの照合を行い、OK を確認する。
4. setup を VM 内デタッチドで実行する（`--gguf-dir` にダウンロード先フォルダを指定。それ以外の引数は通常版と同じ）:
   `STRATA_UV=1 bash setup.sh --setup --yes --backend cuda --cuda 13 --gpu 0 --family qwen --model IQ3_S --context 32768 --vision no --data-dir /content/Strata-data --host 127.0.0.1 --port 8081 --no-browser --no-start --gguf-dir <フォルダ>`
5. ログで完了マーカーを確認する。

予備ルート（未検証）: `jimmy31chen/RVN-Qwen3.8-Flash-Next-Abliterated-Uncensored-IQ3_S-Strata`（単一 GGUF + 構築済みパック）。

実測参考: このモデルでの初回構築（2026-11-12）の decode は初回 50.5 tok/s → キャッシュ温まり後 61.7 tok/s（日本語プロンプト、思考なし）。

### 3. 設定（サーバー初回起動の前に行う）
`/content/Strata/strata-iq3_s.json` に以下を設定する:
- `api_key` に API キー（値はコンソール・チャット・ログに**絶対に出力しない**。ファイル間でのみ受け渡し、一時ファイルは shred する）
- `reasoning_effort` = `"high"`
- `args` の `--max-context` = `262144`

### 4. 起動と公開
1. サーバーを VM 内デタッチドで起動する（`bash run-iq3_s.sh`、ログは `/content/server.log`）。
2. `cloudflared tunnel --no-autoupdate --url http://127.0.0.1:8081` を VM 内デタッチドで起動し、`*.trycloudflare.com` の URL を取得する。
3. `/health` が `loaded: true`、`max_context: 262144` を返し、サーバーログに `thinking level: high` が出るまで待つ（読み込み約4分）。

### 5. 検証（キープアライブ開始前に必ず済ませる）
Colab VM 自身からローカルで:
- `GET /health` → 200
- API キーなし POST `/v1/chat/completions` → 401
- API キーあり POST → 200 で実応答が返る

### 6. キープアライブ（最重要）
Colab はカーネルの実行活動が無いセッションを回収します。次の設計でカーネルを「実行中」に保ち続けてください:
- **チャンク方式のセル**: 約8分間（24回 × 20秒）実行し続けるセル。各回で `http://127.0.0.1:8081/health` を取得して結果（loaded / max_context）を print し、最後に return する。
- **監督プロセス**: `colab exec -s <name> -f <cell> --timeout 0` でセルを接続し、終了したら約5秒で接続し直す。出力が **240秒** 無い場合は子プロセスが固まったとみなして kill し、再接続する（ウォッチドッグ）。セッション喪失（"appears to be lost"）を検知したら再試行せず停止する。
- **理由（踏み抜き禁止の地雷）**:
  - `colab exec` の既定は「出力無音30秒でクライアント切断」。60秒間隔の print では必ず当たる → print は20秒間隔 + `--timeout 0` が必須。
  - 無限ループのセルはクライアント切断後もカーネル側に残り、後続の exec を全て塞ぐ。チャンク方式にするのはそのため。塞がれた場合の解除は `colab restart-kernel`（サーバープロセスは無事）。
  - カーネルは直列実行なので、検証系の exec はキープアライブ開始前に済ませる。

### 7. 監視
15分間隔で次を確認する仕組みを用意する:
- 監督プロセスが死んでいたら再起動する
- セッション喪失を検知したら「生存時間・最後のハートビート・コンピュートユニット残高（`colab usage`）」を1回だけユーザーに通知する。**自動で再取得はしない**
- 開始から6時間に到達したら1回通知する（自動停止はしない）

### 8. 引き渡しと停止
- ユーザーへ: Base URL（`<トンネルURL>/v1`）、モデル名 `qwen3.8-flash-next-iq3_s`、API キーの保管場所（env ファイルのパス。値そのものは貼らない）、設定値（思考 high / コンテキスト 262144）、検証結果を報告する。
- 停止を頼まれたら: 監督 → 監視 → `colab stop -s <name>` の順で止め、`colab usage` で消費レートが 0 になったことを確認して報告する。

## 禁止事項
- API キーの値をチャット・ログ・コンソールに出すこと
- セッション喪失後の自動再取得
- 無限ループセルの使用
- プロキシ環境変数を付けたまま colab コマンドを実行すること

---

*著者: KJ618 / 共同著者: Muse*
