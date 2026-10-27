# Trellis のブログのカバー画像の元

Zenn の記事を Trellis のブログ（trellis-grow.com/blog/）へ同期するとき（`zenn-sync`）に使う。Zenn は読まない。

| ファイル | 中身 | 作る道具 |
|---|---|---|
| `<slug>.json` | 分類（`trellis`・`web`・`ai`）、カバーの言葉（`catch` 2行・`sub` 1行）、挿絵のファイル名、読み上げの説明（`alt`）、挿絵の主題とモチーフ（`theme`・`motif`） | 手で書く |
| `<slug>.art.jpg` | 挿絵（文字なし。右に主題、左に文字を重ねる余白） | Trellis のリポジトリの `.agents/skills/blog-cover/scripts/gen_art.py`（Nano Banana 2 Lite。画風は同じスキルの `settings/style.json` で固定） |
| `<slug>.jpg` | カバー画像（1200×630） | Trellis のリポジトリの `blog-studio/tools/cover` |

作り方の正本は Trellis のリポジトリの `.agents/skills/blog-cover/SKILL.md`。

- `images/` に置かないのは、Zenn が `images/` の中の拡張子を検査し、画像以外があるとデプロイを止めるため。
- 記事の frontmatter には何も足さない。
- 設計: Trellis のリポジトリの `20-library/web-delivery/blog-studio/docs/2026-10-26_ブログ一覧とアイキャッチ_設計.md`
