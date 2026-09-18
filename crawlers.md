# 主要AIクローラー完全リスト（2026年5月版）

robots.txt で User-agent 別に制御するときに参照する。学習用クローラーと推論時参照クローラーを区別するのが重要。

## クローラー一覧

| 提供元 | User-Agent | 用途 | 学習/参照 |
|---|---|---|---|
| **OpenAI** | `GPTBot` | ChatGPT/モデル学習 | 学習 |
| OpenAI | `OAI-SearchBot` | ChatGPT Search のリアルタイム参照 | 参照 |
| OpenAI | `ChatGPT-User` | ユーザーが ChatGPT で URL を貼った時の取得 | ユーザー駆動 |
| **Anthropic** | `ClaudeBot` | Claude モデル学習（旧 anthropic-ai/Claude-Web を統合） | 学習 |
| Anthropic | `Claude-User` | claude.ai でユーザーがリンク参照した時 | ユーザー駆動 |
| Anthropic | `Claude-SearchBot` | Claude 検索機能用 | 参照 |
| **Google** | `Google-Extended` | Bard/Gemini/Vertex AI 学習用（Googlebot と別制御可） | 学習 |
| Google | `GoogleOther` | Google 研究/プロダクト一般用 | その他 |
| **Perplexity** | `PerplexityBot` | Perplexity 検索インデックス | 参照 |
| Perplexity | `Perplexity-User` | ユーザー操作時の取得 | ユーザー駆動 |
| **Apple** | `Applebot-Extended` | Apple Intelligence 学習用 | 学習 |
| **Meta** | `Meta-ExternalAgent` | Meta AI / Llama 学習 | 学習 |
| **ByteDance** | `Bytespider` | Doubao/字節跳動AI | 学習 |
| **Cohere** | `cohere-ai` | Cohere モデル | 学習 |
| **Amazon** | `Amazonbot` | Alexa/Amazon AI | 学習 |
| **DuckDuckGo** | `DuckAssistBot` | DuckDuckGo AI | 参照 |
| **Common Crawl** | `CCBot` | 多くのLLMの学習元データセット | 学習（間接） |

## 区別の重要ポイント

**学習系**（`GPTBot` `ClaudeBot` `Google-Extended` `Applebot-Extended` `Meta-ExternalAgent` `CCBot` 等）を `Disallow` すると：
- AIモデルの将来バージョンに自サイトデータが含まれない
- 既に学習済みのデータからは除去されない（取り消しは効かない）

**参照系**（`OAI-SearchBot` `Claude-SearchBot` `PerplexityBot` 等）を `Disallow` すると：
- AI検索の回答に自サイトが引用されなくなる
- ブランド露出・流入機会を失う

→ 多くのサイトは「学習NG・参照OK」のハイブリッド戦略（パターンB）が合理的。

## robots.txt サンプル

### パターンA: 完全許可
```
User-agent: *
Allow: /

Sitemap: https://example.com/sitemap.xml
```

### パターンB: 学習拒否・参照許可（推奨）
```
# === 学習用クローラーをブロック ===
User-agent: GPTBot
Disallow: /

User-agent: ClaudeBot
Disallow: /

User-agent: Google-Extended
Disallow: /

User-agent: Applebot-Extended
Disallow: /

User-agent: Meta-ExternalAgent
Disallow: /

User-agent: Bytespider
Disallow: /

User-agent: cohere-ai
Disallow: /

User-agent: CCBot
Disallow: /

User-agent: Amazonbot
Disallow: /

# === 参照用クローラーは許可（明示） ===
User-agent: OAI-SearchBot
Allow: /

User-agent: ChatGPT-User
Allow: /

User-agent: Claude-User
Allow: /

User-agent: Claude-SearchBot
Allow: /

User-agent: PerplexityBot
Allow: /

User-agent: Perplexity-User
Allow: /

User-agent: DuckAssistBot
Allow: /

# === 検索エンジン本体 ===
User-agent: Googlebot
Allow: /

User-agent: Bingbot
Allow: /

User-agent: *
Allow: /

Sitemap: https://example.com/sitemap.xml
```

### パターンC: 完全拒否
```
User-agent: GPTBot
Disallow: /
User-agent: OAI-SearchBot
Disallow: /
User-agent: ChatGPT-User
Disallow: /
User-agent: ClaudeBot
Disallow: /
User-agent: Claude-User
Disallow: /
User-agent: Claude-SearchBot
Disallow: /
User-agent: Google-Extended
Disallow: /
User-agent: PerplexityBot
Disallow: /
User-agent: Perplexity-User
Disallow: /
User-agent: Applebot-Extended
Disallow: /
User-agent: Meta-ExternalAgent
Disallow: /
User-agent: Bytespider
Disallow: /
User-agent: cohere-ai
Disallow: /
User-agent: CCBot
Disallow: /
User-agent: Amazonbot
Disallow: /
User-agent: DuckAssistBot
Disallow: /

# 通常検索は許可
User-agent: Googlebot
Allow: /
User-agent: Bingbot
Allow: /

Sitemap: https://example.com/sitemap.xml
```

## 技術的ブロック（robots.txt 不遵守クローラー対策）

robots.txt は紳士協定。守らないクローラーがいる場合は edge レイヤーでブロック：

### Cloudflare
- ダッシュボード → Security → Bots → **Block AI Bots** トグル
- カテゴリ別（学習/参照）で細かく設定可

### Nginx
```nginx
if ($http_user_agent ~* (GPTBot|CCBot|ClaudeBot|Bytespider|Meta-ExternalAgent)) {
    return 403;
}
```

### Apache (.htaccess)
```apache
RewriteEngine On
RewriteCond %{HTTP_USER_AGENT} (GPTBot|CCBot|ClaudeBot|Bytespider) [NC]
RewriteRule .* - [F,L]
```

### Vercel / Next.js middleware
```ts
// middleware.ts
export function middleware(req: NextRequest) {
  const ua = req.headers.get('user-agent') || '';
  if (/GPTBot|CCBot|ClaudeBot|Bytespider/i.test(ua)) {
    return new Response('Forbidden', { status: 403 });
  }
}
```

## 公式ドキュメントリンク

- OpenAI: https://platform.openai.com/docs/bots
- Anthropic: https://support.anthropic.com (search "ClaudeBot")
- Google: https://developers.google.com/search/docs/crawling-indexing/google-common-crawlers
- Perplexity: https://docs.perplexity.ai/guides/bots
- Apple: https://support.apple.com/en-us/119829

## 確認方法

```bash
# 自サイトに来てるAIクローラーをログから集計
awk '{print $14}' access.log | grep -oE "(GPTBot|ClaudeBot|PerplexityBot|CCBot|Google-Extended|Applebot-Extended)" | sort | uniq -c | sort -rn
```

Cloudflare 利用時は Analytics → Security Events で User-Agent 別に可視化可能。
