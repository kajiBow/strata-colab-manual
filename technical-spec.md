# 技術仕様: Strata × Colab A100

## 1. システム構成

| コンポーネント | 場所 | 役割 |
|---|---|---|
| Strata サーバー | Colab VM、port 8081 | Web UI と OpenAI 互換 API（`/v1/chat/completions`、`/v1/models`）と Anthropic 互換 API（`/v1/messages`）を提供。実体は `serve/server.py --engine strata` + エンジン子プロセス |
| cloudflared | Colab VM | クイックトンネルで 8081 を `https://*.trycloudflare.com` に公開。名前付きトンネルではないため URL は毎回変わる |
| google-colab-cli (v0.8.0) | 管理側 VM | セッション取得 (`colab new`)、コード実行 (`colab exec`)、カーネル再起動 (`colab restart-kernel`)、停止 (`colab stop`)、残高確認 (`colab usage`) |
| キープアライブ監督 | 管理側 VM | チャンクセルを `colab exec --timeout 0` で接続し直し続ける Python プロセス |
| 監視ジョブ | 管理側 VM (cron、15分) | 監督の再起動、セッション喪失の検知と通知 |

## 2. ハードウェアと容量

- Colab A100 ランタイム実測: **A100-SXM4-40GB / RAM 83GB / ディスク 113GB / 12 vCPU**
- IQ3_S の必要メモリ（公式見積もり式）: arena 50.3GB + ctx × 13 × 1056 bytes（int8 KV）+ 24GB
  - ctx 262144 → 約78GB … A100 の 83GB に収まる
  - ctx 1048576 (1M) → 約89GB … **RAM 不足で不可**。加えてモデルの学習済み上限は 262144 で、Strata の setup 選択肢も最大 524288
- L4 ランタイム（VRAM 24GB + RAM 52GiB = 計76GB）では IQ3_S は載らない

## 3. Strata サーバー設定

設定ファイル: `/content/Strata/strata-iq3_s.json`

| キー | 値 | 備考 |
|---|---|---|
| `api_key` | ランダムキー | 設定すると Django 風のホストチェックが無効化され、トンネル経由の 403 が解ける。キーなしリクエストは 401 |
| `reasoning_effort` | `high` | サーバー全体の既定。対応値: none / low / medium / high（high の内部名は xhigh、既定も xhigh）。リクエスト側の指定が優先 |
| `args` 内 `--max-context` | `262144` | Web UI の Model settings からは編集不可 |

生成パラメータの仕様:
- `max_tokens` は **思考と本文の合計枠**。思考 high のままで小さくすると、思考だけで尽きて本文が空（content=None）になる。推論不要なら `reasoning_effort: "none"` が確実。細かい制御は `reasoning_budget_tokens`（リクエスト単位/設定ファイル共通）
- 生成は同時1件が既定（`--parallel` は opt-in）
- Web UI の Chat settings に Max tokens 欄があり、**空欄 = 終わるまで無制限**（天井はコンテキスト）
- Web UI の API キー入力欄は About → Settings（ブラウザの localStorage `strata.apikey` に保存されるだけ。サーバー側キーの発行・変更はできない）

性能実測（IQ3_S、A100）: decode 初回 約50 tok/s、キャッシュ温まり後 約62 tok/s（公式版のローカル API では条件により 80 tok/s の回もあり）。

## 4. セッション消滅のメカニズム（調査結果）

主因は**アイドル回収**。根拠:
- 公式 CLI ドキュメント: 「VM の生存はカーネルの実行活動と対話で自動判定される。CLI にキープアライブデーモンは無い」。旧デーモンが維持していたのはカーネル exec ではなく **60秒ごとの TFE ping（LastActiveTime）という別信号**で、短い exec の繰り返しでは完全には代替できない（strata3 は約3.5分ごとの exec でも 111〜144分で消滅）。
- ランタイムプロキシトークンは1時間で失効（tokenExpiresInSeconds=3600）し、長命接続では再同期されない。401/404 を致命的とみなしてローカルでセッションを prune する CLI の挙動との相互作用も併発要因の可能性。
- トンネル検知説: FAQ 上は「対話的計算に関係しない Web サービス提供」「リモートプロキシへの接続」は全ランタイム禁止だが、有料アカウントが cloudflared の LLM サーバーを1〜2時間で切られた報告は横断調査で見つからなかった。通信量と終了の相関も実測では見られない（ほぼ無通信でも約48分で落ちた回がある）。
- 寿命上限説（12h/24h）はこの時間スケールでは否定。
- Google AI Pro ≠ Colab Pro+。「タブを閉じても動く」background execution は Pro+ のみ。

対策の優先度（未実施のものも含む）:
1. 実行し続けるセルでカーネルを「実行中」に保つ（本マニュアルの方式。ComfyUI 系ノートブックの定番）
2. TFE ping 相当を維持するラッパー（colabapi、hermes-colab-cli のトークン再同期パッチ）
3. ブラウザのノートブックタブを開いたままにする + JS キープアライブ
4. Colab Pro+ の background execution（最大24時間、唯一の公式ルート）

## 5. キープアライブ設計

| パラメータ | 値 | 根拠 |
|---|---|---|
| セルの長さ | 24ビート ≈ 480秒 | 長すぎると他 exec が入れない、短すぎると接続 overhead が増える |
| ビート間隔 | 20秒 | `colab exec` の無音切断（既定30秒）を確実に回避 |
| ビート内容 | `/health` の loaded / max_context | 監督ログがそのままサーバー監視になる |
| exec の `--timeout` | 0（無制限） | 無音切断そのものを無効化 |
| 監督の再接続間隔 | 5秒 | カーネルが空く時間を最小化 |
| ウォッチドッグ | 無音240秒で子を kill | WebSocket 切断で子プロセスが固まる実障害（2026-11-13 12:24）への対策 |
| 喪失検知 | 出力に "appears to be lost" / "Session not found" | 検知したら再試行せず停止し、監視ジョブの通知に委ねる |

## 6. 障害モードと教訓

| 事象 | 原因 | 対処 |
|---|---|---|
| `colab new` が 411 で失敗 | 割り当て POST が管理側 VM の出口プロキシを経由 | プロキシ環境変数を全て外して実行 |
| exec が30秒で切断される | CLI 既定の無音タイムアウト | `--timeout 0` + 20秒間隔の出力 |
| 無限セルがカーネルを塞ぎ後続 exec が永遠にキュー待ち | セルはクライアント切断後もカーネル側で実行継続 | チャンク方式にする。既に塞がれたら `colab restart-kernel`（サーバープロセスは無事） |
| 切り離しループが認証プロンプトで即死 | keyring に届かずトークン再同期不可 | 取得ループは対話的プロセスツリー内で回す |
| exec 子プロセスが無音のまま固まる | WebSocket 切断後もプロセスが終了しない | 監督のウォッチドッグ（無音240秒 kill） |
| トンネル経由で 403 | Strata の DNS リバインディング保護（allowed_hosts） | API キーを設定するとホストチェックが無効化される（URL が変わっても再設定不要） |

## 7. コンピュートユニット

- 消費は **接続時間 × GPU 種別の固定レート**。A100 実測 6.73/時、L4 実測 約1.94/時
- 生成量では減らない。**セッションを止めない限り減り続ける**
- 付与は月次（Google AI Pro 特典）。有効期限は付与から90日。日次回復は無い
- 残高確認: `colab usage`

## 8. 関連リポジトリ・出典

- Strata: https://github.com/Niko1221/Strata
- 公式モデル（ISTA-DASLab）: Hugging Face の Qwen3.8-Flash-Next IQ3_S
- Uncensored: `alesha-pro/Qwen3.8-Flash-Next-abliterated-GSQ-RCO-Strata-GGUF`
- 同種の先行実装 collabosm（延命設計の参考）: https://github.com/architectds/collabosm
- Colab FAQ: https://research.google.com/colaboratory/faq.html
