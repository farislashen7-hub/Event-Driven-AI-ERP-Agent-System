
# ERP AI Assistant

Natural language ERP assistant for small businesses. Staff talk to it on Telegram (text, voice notes, photos, documents) or through a dashboard chat widget, and an AI agent reads and writes the live Supabase database: products, sales, expenses, customers, suppliers, purchase orders, employees, and stock.

Built for the Egyptian market (Egyptian Arabic replies, EGP amounts). A daily job also emails and logs a low-stock report.

## Features

**AI agent (20 tools)**
- Products: list, add, update price/cost/stock
- Sales: query by date range and branch, record a sale
- Expenses: query by date range and branch, record an expense (plain insert or RPC with alert)
- Customers, suppliers, employees: list, add, update (employees)
- Purchase orders: create header, then add line items using the returned `po_id`
- Reports: daily summary (sales, expenses, net) and top-selling products from Postgres views
- Stock: per-warehouse stock adjustment (upsert)
- Never answers business questions from model memory. Missing data gets a fixed "not in the system" reply
- Asks for missing fields before any write, asks for confirmation before any delete, asks to clarify ambiguous periods

**Multi-channel input**
- Telegram text, voice (transcribed, Arabic), image (described by a vision model), and documents
- Documents: PDF, XLS, XLSX, CSV, TXT, HTML, XML, JSON, RTF, ICS. Anything else gets an "unsupported file" reply
- Dashboard chat: `POST /webhook/erp-chat` with `question`, `session_id`, `branch_id`

**Automation**
- Daily 12:00 low-stock report: reads `products_with_stock`, flags items at or below reorder level, emails it and appends rows to Google Sheets
- Per-session memory (last 10 messages), model fallback (OpenRouter, then Ollama)

## Tech stack

| Layer | Tools |
|---|---|
| Automation | n8n (webhook, Telegram trigger, AI Agent, tool nodes) |
| Database | Supabase (Postgres, views, RPC) |
| LLM | OpenRouter `gpt-4o-mini` (primary), Ollama `qwen2.5:3b` (fallback) |
| Voice / Vision | OpenAI (transcription, image analysis) |
| Channels | Telegram Bot API, dashboard webhook |
| Reports | Gmail, Google Sheets |

## Architecture

```
Telegram ─► Switch ─┬ text ─────────────────────────────┐
                    ├ voice ─► download ─► transcribe ──┤
                    ├ photo ─► download ─► vision ──────┤
                    └ document ─► download ─► route by  │
                         mime ─► extract ─► prompt ─────┤
                                                        ▼
Dashboard ─► POST /erp-chat ─► map prompt ─────► Auth + Sanitize
                                                        │
                                                        ▼
                                                    AI Agent ◄── memory (10)
                                                        │   ◄── 20 Supabase tools
                                                        ▼
                                                 Route Response
                                          Telegram ──┴── Dashboard
                                          sendMessage    respondToWebhook

Schedule (12:00) ─► products_with_stock ─► flag low stock ─┬─► Gmail
                                                           └─► Google Sheets
```

## Getting started

### 1. Import

Import the workflow JSON into n8n and re-link every credential.

### 2. Credentials

| Credential | Type | Used by |
|---|---|---|
| Supabase | `supabaseApi` | All ERP tools, daily report |
| OpenRouter | `openRouterApi` | Agent primary model |
| Ollama | `ollamaApi` | Agent fallback model |
| OpenAI | `openAiApi` | Voice transcription, image analysis |
| Telegram | `telegramApi` | Trigger, file download, replies |
| Gmail | `gmailOAuth2` | Daily report email |
| Google Sheets | `googleSheetsOAuth2Api` | Daily report log |
| Custom Auth | `httpCustomAuth` | `add_expense_with_alert` RPC call |

### 3. Database

Tables: `products`, `sales`, `expenses`, `customers`, `suppliers`, `employees`, `purchase_orders`, `purchase_order_items`, `product_stock`, `warehouses`.
Views: `daily_summary_view` (`summary_date`), `top_products_view` (`total_quantity_sold`), `products_with_stock` (`total_stock`).
RPC: `add_expense_with_alert`.
`sales` and `expenses` need a `branch_id` column. `product_stock` needs a unique constraint on `(product_id, warehouse_id)` for the upsert.

### 4. Channels

1. Telegram: create a bot with BotFather, add the token as a credential.
2. Dashboard: send `POST {N8N_URL}/webhook/erp-chat` with body `{ "question": "...", "session_id": "...", "branch_id": "branch_1" }`. Response is `{ "output": "..." }`.

### 5. Run

1. Activate the workflow.
2. Message the bot, for example "كام مبيعات النهارده؟".
3. Check the daily report at 12:00.

## Known issues

| Issue | Fix |
|---|---|
| `adjust_stock` upsert node has no table or fields configured, the tool does nothing | Set `product_stock` and map `product_id`, `warehouse_id`, `quantity` via `$fromAI` |
| Daily email and sheet use `low_stock_count`, `report_text`, `items[0]` but the code node outputs one row per product, so subject is `undefined` and sheet rows are wrong | Add an aggregate step that builds `low_stock_count` and `report_text`, and append rows directly |
| Two expense tools (`add_expense`, `add_expense_with_alert`), prompt only describes one. `add_expense` also writes both `expense_date` and `date` | Keep one, drop the extra `date` field |
| `$credentials.telegramApi.accessToken` in the file download URL is not available in expressions. Voice download also lacks `responseFormat: file` | Use the Telegram node's file download, or an HTTP node with predefined Telegram credentials |
| Image and voice nodes have empty OpenAI credentials | Link an OpenAI credential |
| `Auth + Sanitize1` only trims text and sets a default branch, there is no real auth | See Security notes |
| Date filters fall back to hardcoded 2026 | Default to current date range |
| Fallback model `qwen2.5:3b` is weak at tool calling | Use a 7B+ model |

## Security notes

- **Add auth before production.** Anyone who finds the Telegram bot or the webhook URL can read salaries and write records. Allowlist Telegram chat IDs and require a secret header on the dashboard webhook, checked in the first node.
- `branch_id` comes from the client and only scopes sales and expenses. Resolve it server-side from the authenticated user, and scope the other tables too.
- Keep all keys in n8n credentials. Never inline them in nodes or prompts.
- The agent prompt forbids revealing instructions, credentials, and tool details, and forbids deletes without confirmation. There are no delete tools.
- Use a least-privilege Supabase role or RLS for the agent instead of broad access.
- Do not commit exported workflow JSON with credential IDs, project URLs, or personal emails to a public repo.

## Project structure

```
workflow.json     exported n8n workflow
README.md
```

## Author

**Faris Ataa Lashen**, AI Automation Engineer (n8n, AI Agents, RAG, LLMs)
Portfolio: https://farislashen7-hub.github.io/portfolio · LinkedIn: https://linkedin.com/in/farislashen
