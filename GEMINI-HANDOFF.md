# nanoclaw + Gemini 実装 引継ぎブリーフ

> このファイルは、**この nanoclaw repo 内で起動した Claude Code セッション**が実装を続行するための引継ぎ。
> 元の設計・経緯は別 repo の GitHub Issue: `kodaikumatani/homelab-k8s#65`。

## ゴール

既存 OSS nanoclaw を **Google Gemini をバックエンド**にして PoC で動かす（A→B の 2 段構えの **Phase 1 = `/add-opencode` 経由 Gemini**）。

## 確定済みの方針（user 合意済み）

| 項目 | 決定 |
|---|---|
| credential 注入 | **native creds**（`NANOCLAW_NATIVE_CREDENTIALS=true`、.env に key 直書き）。OneCLI は使わない |
| messaging channel | **Telegram**（bot token のみ、電話ペアリング不要） |
| Gemini API key | **Google AI Studio**（https://aistudio.google.com） |
| Gemini モデル | `gemini-2.5-pro`（small 用に `gemini-2.5-flash` でも可） |

## 前提状態（2026-06-21 時点で確認済み）

- repo は clone 済（このディレクトリ = `~/ghq/github.com/nanocoai/nanoclaw`、upstream nanocoai。fork は未）
- `node v24` あり / `pnpm`・`bun` 未導入（`nanoclaw.sh` が入れる想定）
- Docker = **Colima**（`~/.colima/default/docker.sock`）、**daemon 起動済**（`colima start` 実行済、docker 29.2.1 稼働）
- `.env.example` は空

## 進め方（推奨フロー）

1. **インストーラ起動**: `bash nanoclaw.sh`
   - Node/pnpm/Docker のチェック、credential 登録、container build、channel ペアリングまで一気通貫。
   - **native creds を使うので、Anthropic を OneCLI に登録するフェーズはスキップ/最小化したい**。`NANOCLAW_NATIVE_CREDENTIALS=true` を `.env` に先に置いてから始めるか、インストーラの分岐に従う。
2. **Telegram channel 追加**: `/add-telegram`（in-repo skill）。BotFather で取った bot token を渡す。
3. **OpenCode provider 追加**: `/add-opencode`（in-repo skill）。`providers` ブランチから provider コードを copy → 依存追加 → image 再ビルド。
4. **Gemini 設定**（下記「未解決の要確認」を詰めてから）。
5. **疎通確認**: Telegram から `@Andy ...` で投げ、Gemini が応答 + `Unknown provider: opencode` が出ないこと、UUID/session 警告が無いことをログで確認。

## ⚠️ 未解決の要確認（推測で進めない — ここが Phase 1 の肝）

`/add-opencode` の SKILL.md の実例は **DeepSeek / OpenRouter / Anthropic / Zen のみで Google/Gemini 例が無い**。しかも全例が **OneCLI 前提**。今回は **native creds × Google** なので、以下を実コードで確定すること:

1. **OpenCode の google provider 設定**: `git show origin/providers:src/providers/opencode.ts` と `container/agent-runner/src/providers/opencode.ts` を読み、`OPENCODE_PROVIDER=google` 時にどの env / base URL / key 変数を使うか確認。
   - 想定: `OPENCODE_PROVIDER=google`, `OPENCODE_MODEL=google/gemini-2.5-pro`。
   - base URL は Google AI Studio の Generative Language API（OpenAI 互換 or native）。`ANTHROPIC_BASE_URL` に何を入れるか（あるいは google provider では別扱いか）を opencode.ts で確認。
2. **native creds で Gemini key をどの env で渡すか**: `.claude/skills/use-native-credential-proxy/` と `src/container-runner.ts` を読み、`NANOCLAW_NATIVE_CREDENTIALS=true` 時にコンテナへ渡る key の env 名を確認。OpenCode google は通常 `GEMINI_API_KEY` か `GOOGLE_GENERATIVE_AI_API_KEY` を見るはず。native path がそれを通すか、ブリッジが要るかを確定。
3. 上記が skill の OneCLI 前提と噛み合わない場合、**native creds と opencode-google を繋ぐ最小の調整**（.env passthrough の env 名追加など）が要るか判断。

## per-group で Gemini を有効化（設定の置き場所）

`groups/<folder>/container.json` に:
```json
{
  "provider": "opencode",
  "model": "google/gemini-2.5-pro"
}
```
（runner は DB ではなく container.json の `provider` を読む。SKILL.md「Per group / per session」参照。）

host `.env`（native creds 前提のたたき台 — 上記確認後に確定）:
```env
NANOCLAW_NATIVE_CREDENTIALS=true
OPENCODE_PROVIDER=google
OPENCODE_MODEL=google/gemini-2.5-pro
OPENCODE_SMALL_MODEL=gemini-2.5-flash
# GEMINI_API_KEY=... または ANTHROPIC_BASE_URL=... は上記確認後に確定
```

## gotcha

- 既存 group があると `data/v2-sessions/<id>/agent-runner-src/providers/` の overlay が image を上書きするので、`/add-opencode` SKILL.md step 8 の overlay 同期を忘れない。
- container build cache: `Unknown provider: opencode` が出たら `docker builder prune -f && ./container/build.sh`。
- opencode-ai は CLI/SDK ともに **1.4.17 ピン**（1.14.x は API 破壊で非互換）。

## user の運用ルール（重要）

- **勝手にコマンド実行しない**。実行前に必ず説明して合図を待つ。
- **1 step ずつ**提示 → 実行 → 結果待ち。1 返答に複数指示を並べない。
- 大きな変更前に現状の健全性を確認、コード変更は先行で書いても**適用は合図待ち**。
