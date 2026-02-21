```
User speaks → Wispr Flow transcribes → Telegram message
    → Grammy bot receives text
    → Claude API extracts structure (JSON)
    → Supabase stores raw + structured entry
    → Bot sends "✓ Logged"

Every Sunday 7pm:
    → Cron fires → Query Supabase (last 7 days)
    → Claude synthesizes digest
    → Send via Telegram + Email
```


## File Structre 
hedonic/
├── src/
│   ├── bot/
│   │   ├── index.ts          # Grammy bot setup + webhook registration
│   │   ├── handlers.ts       # Message handler — main entry point for each message
│   │   └── responses.ts      # Telegram reply templates
│   ├── extraction/
│   │   ├── extractor.ts      # Claude API call + extraction logic
│   │   └── prompts.ts        # All Claude prompts in one place
│   ├── digest/
│   │   ├── digest.ts         # Weekly digest generation logic
│   │   ├── scheduler.ts      # node-cron setup
│   │   └── email.ts          # Resend email delivery
│   ├── db/
│   │   ├── supabase.ts       # Supabase client initialization
│   │   └── queries.ts        # All database queries in one place
│   └── index.ts              # App entry point
├── .env.example              # Environment variable template
├── package.json
├── tsconfig.json
└── railway.json              # Railway deployment config

