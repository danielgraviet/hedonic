| Layer | Technology |
|---|---|
| Bot Framework | [Grammy](https://grammy.dev/) — modern Telegram bot framework for Node.js |
| Runtime | Node.js 20 LTS + TypeScript |
| AI / Extraction | Anthropic Claude API (`claude-haiku-4-5` for extraction, `claude-opus-4-5` for digest) |
| Database | Supabase (PostgreSQL) |
| Hosting | Railway (auto-deploy from GitHub) |
| Scheduler | `node-cron` within the Railway service |
| Email | [Resend](https://resend.com) — transactional email API, generous free tier |