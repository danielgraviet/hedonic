The core table. Stores both the raw user text and all structured fields extracted by Claude.

| Field | Type | Description |
|---|---|---|
| `id` | `uuid` | Primary key, auto-generated |
| `created_at` | `timestamptz` | Timestamp of the entry (auto-set by Supabase) |
| `telegram_user_id` | `bigint` | Telegram user ID (enables multi-user support later) |
| `raw_text` | `text` | The exact message the user sent, unmodified |
| `sentiment` | `text` | Extracted sentiment: `positive`, `negative`, or `mixed` |
| `intensity` | `int2` | Intensity score 1–10 (1 = mild, 10 = peak experience) |
| `category` | `text` | Primary category (see categories below) |
| `subcategory` | `text` | Optional finer detail (e.g. `coding`, `running`, `meal`) |
| `time_of_day` | `text` | Inferred if mentioned: `morning`, `afternoon`, `evening`, `night` |
| `keywords` | `text[]` | Array of extracted keywords and themes |
| `context_notes` | `text` | Additional context Claude extracted (solo vs social, location, etc.) |
| `extraction_model` | `text` | Which Claude model was used (for future comparison) |

**Valid categories:** `deep_work` · `creative` · `learning` · `social` · `physical` · `emotional` · `other`
