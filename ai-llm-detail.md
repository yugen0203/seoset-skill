# AI/LLM最適化 詳細ガイド（2026年5月版）

SKILL.md セクション15の詳細解説。AI回答エンジンへの引用最適化（GEO/AEO）と llms.txt 規格を実装するときに参照する。

## 全体像：3つのレイヤー

AI最適化は次の3層で考える。混同すると的外れな対策になる。

| レイヤー | 目的 | 主な対策 |
|---|---|---|
| **① クローラー制御** | AI企業に学習/参照を許可 or 拒否 | `robots.txt` で User-agent 別制御 → 詳細は [`crawlers.md`](crawlers.md) |
| **② コンテンツ提供** | AIが理解しやすい形で提供 | `llms.txt` / `llms-full.txt` / 構造化データ |
| **③ 引用最適化** | AI回答に引用されやすい書き方 | Answer-first / E-E-A-T / FAQ構造 |

---

## ① クローラー制御

主要AIクローラー17種の完全リスト・各社挙動・robots.txt サンプルは [`crawlers.md`](crawlers.md) に分離。

### 戦略パターン（4択）

**A. 完全許可** — メディア/SaaS/ブランド広報サイト向け。AI回答に引用されることがブランド露出になる。robots.txt にAI系の記述なし、または `Allow: /` のみ。

**B. 学習拒否・参照許可** — オリジナルコンテンツ重視サイト向け。学習はNGだがリアルタイム検索で引用されるのはOK。
- `GPTBot` `ClaudeBot` `Google-Extended` `CCBot` を `Disallow`
- `OAI-SearchBot` `ChatGPT-User` `Claude-User` `PerplexityBot` を `Allow`

**C. 完全拒否** — 有料記事/会員制/著作権センシティブ向け。全AI系を `Disallow: /`。
- ※ robots.txt はクローラー側の遵守任意。技術的ブロック（Cloudflare AI Bots設定 / WAF / User-Agent block at edge）併用が確実。

**D. ai.txt（提案中）** — Spawning が提唱する、用途別（学習/推論/検索など）の細かい制御規格。2026年5月時点ではまだデファクトではないが、将来の選択肢として把握。

→ サンプル robots.txt は [`templates/robots.txt`](templates/robots.txt) 参照。

---

## ② llms.txt（規格詳細）

### 何か
- **2024年9月、Jeremy Howard（Answer.AI / fast.ai 創始者）が提唱**
- ルート直下 `/llms.txt` に **Markdown** でサイト要約を置く規格
- LLMのコンテキスト窓は限られているため「HTML全部読ませる」より「整理済みMarkdown1枚読ませる」方が圧倒的に高品質

### robots.txt との違い

| | robots.txt | llms.txt |
|---|---|---|
| 目的 | アクセス**制御** | コンテンツ**提供** |
| 対象 | クローラー全般 | LLM推論時 |
| 形式 | プレーンテキスト規則 | Markdown |
| 強制力 | クローラーの慣習に依存 | LLM/エージェントが任意に読む |

### 公式仕様

```markdown
# プロジェクト名・サイト名

> 1〜2文の要約。何のサイトか、誰向けか、を端的に。

任意の補足説明段落（オプション）。製品の特徴・前提知識など。

## Docs

- [Quickstart](https://example.com/docs/quickstart): 5分で始める導入手順
- [API Reference](https://example.com/docs/api): 全エンドポイントの仕様

## Examples

- [Sample App](https://example.com/examples/app): 実装例

## Optional

- [Changelog](https://example.com/changelog): バージョン履歴
- [Blog](https://example.com/blog): 開発ブログ
```

**ルール**:
- H1（`#`）は1つだけ＝サイト名
- ブロッククォート（`>`）で要約
- H2（`##`）でセクション分け
- リンクリスト形式 `[タイトル](URL): 説明`
- `## Optional` セクションは「コンテキスト不足時にスキップしてOK」の合図

→ コピペ用テンプレ [`templates/llms.txt`](templates/llms.txt)

### `llms-full.txt`（任意）
- 全コンテンツを Markdown で1ファイルに連結
- LLMが**1リクエストで全文読める**
- 例: Anthropic公式ドキュメントは約30万トークン規模を1ファイル提供
- ナビゲーション/装飾HTMLを除いた純粋なテキストにすること

### 実装例

**Next.js (App Router)**
```ts
// app/llms.txt/route.ts
export async function GET() {
  const docs = await getAllDocs();
  const content = `# MyProduct

> 高速デプロイPaaS。開発者向けに...

## Docs

${docs.map(d => `- [${d.title}](${d.url}): ${d.summary}`).join('\n')}
`;
  return new Response(content, {
    headers: { 'Content-Type': 'text/plain; charset=utf-8' }
  });
}
```

**Nuxt 3**
```ts
// server/routes/llms.txt.get.ts
export default defineEventHandler(async (event) => {
  setHeader(event, 'Content-Type', 'text/plain; charset=utf-8');
  const docs = await getAllDocs();
  return `# MyProduct\n\n> 説明\n\n## Docs\n${docs.map(...)}\n`;
});
```

**静的サイト** — ビルド時にスクリプト（例: `scripts/build-llms-txt.mjs`）で MDX/Markdown のフロントマターを集約 → `public/llms.txt` 出力

### 主要採用例（2026年5月時点）
- **Anthropic** — `docs.anthropic.com/llms.txt` ＋ `llms-full.txt`
- **Cloudflare** — 開発者ドキュメント全体
- **Vercel** — Next.js公式ドキュメント
- **Stripe** — APIドキュメント
- **Mintlify / Fern** — ドキュメントSaaSが自動生成機能を提供
- **Hugging Face / Pydantic / Zapier** など

### 検証ツール
- [llmstxt.org](https://llmstxt.org) — 仕様書、ジェネレーター紹介
- [llmstxt.directory](https://llmstxt.directory) — 採用サイトカタログ
- VS Code拡張: `llms-txt-validator`

### 注意点
- **SEO効果は間接的** — Google検索順位への直接影響は現時点で確認されていない
- **AIエージェント時代の布石** — Claude/GPT/Devin/Cursor 等のエージェントが「このドキュメントを参照して」と言われた時に効率的に読める
- **過度な期待は禁物** — まだ全LLMが標準対応してるわけではない（2026年5月現在、Claude/Cursor/Continue等は読む、ChatGPT本体は明示参照時のみ）

---

## ③ AI回答への引用最適化（GEO / AEO）

**GEO** = Generative Engine Optimization、**AEO** = Answer Engine Optimization。Google AI Overview / ChatGPT Search / Perplexity / Claude などが回答する際に**自サイトを引用させる**戦略。

### 統計（複数のSEO業界調査より）
- AI Overview に引用されるページの **約70%は検索1〜10位**（やはり従来SEOがベース）
- ただし AI Overview 引用率と検索順位は**完全には一致しない**（10位以下からも引用される）
- **明確な答え+構造化+引用元の信頼性** が共通因子

### 実装パターン

**1. Answer-first writing（結論先出し）**
```markdown
## ECサイトのCVRを上げる方法は？

ECサイトのCVRを上げる最も効果的な方法は次の3つです：
1. ページ表示速度を3秒以内にする
2. カート離脱者へのリマーケティング
3. レビュー表示

### 1. ページ表示速度を3秒以内にする
（詳細...）
```
冒頭2〜3文に「答え」を凝縮 → AIがそのまま引用しやすい

**2. 質問形式の見出し（H2/H3）**
- 「〇〇とは？」「〇〇の方法」「〇〇の違い」
- LLMはユーザー質問とH2マッチングで該当セクションを抜き出す傾向

**3. FAQ構造化データ（FAQPage JSON-LD）**
- Google検索結果での表示は2024年以降縮小されたが、**AI参照には依然有効**
- 質問→回答の対応がLLMに分かりやすい
- → [`templates/faq-jsonld.html`](templates/faq-jsonld.html)

**4. E-E-A-T シグナル**
- **Experience**（経験）: 実際の使用例・スクショ・データ
- **Expertise**（専門性）: 著者プロフィール ＋ `Person` JSON-LD
- **Authoritativeness**（権威性）: 被リンク、メディア露出、`sameAs` で SNS/学術プロフィールリンク
- **Trustworthiness**（信頼性）: 運営会社情報、プライバシーポリシー、HTTPS、引用ソース明記

**5. 構造化データを厚めに**
- `Article` + `Person`(author) + `Organization`(publisher) + `BreadcrumbList`
- `Citation` プロパティで参照元を機械可読に

**6. Statistics・Original Data・Quotes**
- 独自調査・統計データはAIが引用したがる（出典として価値が高い）
- 「〇〇調査によると」と引用される形式を狙う

**7. llms.txt との二段構え**
- 一般訪問者向けはHTML/SEO最適化
- AIエージェント・コーディングツール向けは llms.txt で別途整理

---

## ④ 計測

| 指標 | ツール |
|---|---|
| AI Overview 出現率 | Search Console（2024年からAI関連トラフィック区分追加） |
| Perplexity 引用 | [Otterly.AI](https://otterly.ai), [Peec AI](https://peec.ai) |
| ChatGPT 参照流入 | Referrer に `chatgpt.com` / `perplexity.ai` などが入るので GA4 で分離 |
| AIクローラーアクセス | サーバーログを User-Agent で集計、Cloudflare Web Analytics の Bot 区分 |
| Brand mentions in LLMs | [Profound](https://tryprofound.com), [AthenaHQ](https://athenahq.ai) |

---

## 推奨アクションプラン（小→大）

1. **今すぐ（30分）**: robots.txt にAIクローラー制御を明示（許可するにせよ拒否するにせよ態度表明）
2. **今週（数時間）**: ヒーロー記事5〜10本を Answer-first にリライト ＋ FAQ JSON-LD 追加
3. **今月**: `llms.txt` を手書きで作成（Top 20ページくらいに絞ってOK）
4. **継続**: 独自データ・実体験ベースの記事を増やす（AIが書けない領域）

---

## 進め方（このスキル使用時）

1. ユーザーのサイト種別・規模を確認
2. クローラー制御の意向を確認（許可/拒否/ハイブリッド）→ パターン A〜C を提案
3. `templates/` の雛形をベースに具体的な robots.txt / llms.txt を生成
4. 既存記事に対しては Answer-first リライト案を提示
5. 計測の継続フローを最後に提案
