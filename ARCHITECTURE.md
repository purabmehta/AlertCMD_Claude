# AlertCmd — Master Build Documentation
### Version 1.0 — Session Handoff Document

---

## 1. Who You Are and What You're Building

**You are Purab Mehta.** You are a trader building a personal trading operations platform called **AlertCmd**. The end goal is a professional-grade system that:

- Screens crypto (and eventually equities) with zero delay
- Collects TradingView alerts in real time with no limits
- Filters and organizes alerts by strategy, direction, favorites, open trades
- Automates trade execution through Interactive Brokers after proper testing
- Passes and manages prop firm accounts with strict risk management
- Eventually becomes a **sellable product** — a TradingView → Interactive Brokers bridge with dashboards, trade logs, AI performance audit, and multi-account management

**Budget:** $200/month. Zero downtime and zero delay are the top priorities. This is a business expense.

---

## 2. Current State (Where We Are Right Now)

### What Exists Today

**GitHub Repo:** `github.com/purabmehta/AlertCMD_Claude`  
**Live Dashboard:** `https://purabmehta.github.io/AlertCMD_Claude`  
**Tech Stack:** Single-file React app (`index.html`) deployed via GitHub Pages

**Current Pages/Features:**
- `/direction` — Direction Board (Bullish/Bearish columns, Open Trades, Sell Calls)
- `/screener` — Crypto Screener with filter tabs
- `/live-feed` — Live Alert Feed from TradingView webhooks

**Screener filter tabs and their logic:**
- **Bullish Trade Entry:** `weeklyRegime === 'BULLISH' AND trigger12h === 'BULLISH_CROSS'`
- **Bearish Trade Entry:** `weeklyRegime === 'BEARISH' AND trigger12h === 'BEARISH_CROSS'`
- **Already Bullish:** `weeklyRegime === 'BULLISH' AND trigger12h IN ['BULLISH', 'BULLISH_CROSS']`
- **Already Bearish:** `weeklyRegime === 'BEARISH' AND trigger12h IN ['BEARISH', 'BEARISH_CROSS']`
- **Exit Long:** `stochRsi12hCross === 'cross_down' AND trigger12h IN ['BULLISH', 'BULLISH_CROSS']`
- **Exit Short:** `stochRsi12hCross === 'cross_up' AND trigger12h IN ['BEARISH', 'BEARISH_CROSS']`
- **Long Aggressive:** same as Bullish Trade Entry

**Screener data fields used:**
`symbol, weeklyRegime, weekly5, weeklyClose, trigger12h, macd12h, signal12h, stochK, stochD, stochRsi12hCross, lastPrice, priceChangePercent, quoteVolume, h12OpenTime, h12CloseTime, updatedAt`

### Current Infrastructure (Temporary — to be replaced)

| Component | Current | Problem |
|---|---|---|
| Screener backend | Cloudflare Worker (`screener.purabmehta.workers.dev`) | 50 subrequest limit, scans only 24 symbols per request, builds cache incrementally |
| Worker code | Binance-direct, PAGE_SIZE=24, KV cache | Slow to populate full 260-symbol universe |
| Alert storage | Not yet on Supabase | TBD |
| Dashboard hosting | GitHub Pages | Fine, will migrate to Cloudflare Pages |
| Old screener (reference) | VPS at `77.42.23.93` | Full 260 symbols, HTTP only, to be decommissioned |

### Old VPS API (Source of Truth — Reference Only)

```
http://77.42.23.93/api/screener/live?smaTf=1w&smaLength=5&triggerTf=12h&macdFast=12&macdSlow=26&macdSignal=9
```

Returns `{ ok: true, source: 'binance', rows: [...], totalRows: 260 }`

Field names on old VPS (differ from new Worker):
- `price` → maps to `lastPrice`
- `change24hPct` → maps to `priceChangePercent`
- `quoteVolume24h` → maps to `quoteVolume`
- Has `stochRsi12hCross` but NOT separate `stochK`/`stochD`

### Cloudflare Account

- **Account ID:** `10c90545decda1ce2c6aaa814f498e46`
- **Worker name:** `screener`
- **Worker URL:** `https://screener.purabmehta.workers.dev`
- **Current Worker hash:** `e14b13b5` (Binance-direct, PAGE_SIZE=24)

---

## 3. Target Architecture

### Infrastructure Stack

```
TradingView Webhook
        |
        v
Webhook Receiver (Railway.app)
        |
        +---> Supabase PostgreSQL ---> Realtime WebSocket ---> Dashboard
        |         (alerts, trades,                            (Cloudflare Pages)
        |          accounts, audit)
        |
        +---> Screener Service (Railway.app)
        |     Scans Binance every 5 min
        |     All 260+ symbols in parallel
        |     Caches in Supabase
        |
        +---> Rule Engine (Railway.app)
                  |
                  v
           IB Bridge (DigitalOcean Droplet)
           IB Gateway headless + ib_insync
                  |
                  v
           Interactive Brokers (multiple accounts)
```

### Budget Allocation (~$171/month total)

| Service | Purpose | Cost |
|---|---|---|
| Railway.app | Screener + webhook receiver + rule engine | $30/mo |
| DigitalOcean Droplet (4GB/2vCPU) | IB Gateway + bridge | $48/mo |
| Supabase Pro | PostgreSQL + Realtime + backups | $25/mo |
| TradingView Premium | Unlimited alerts + charting | $60/mo |
| Betterstack | Uptime monitoring + alerts + status page | $7/mo |
| Cloudflare Pages | Dashboard hosting + CDN + SSL | $0 |
| Domain | `yourdomain.com` for API + dashboard | ~$1/mo |
| **Total** | | **~$171/mo** |

Reserve $29/month for Polygon.io when you need real-time US stock data.

---

## 4. Build Roadmap

### Phase 1 — Fix the Screener (Week 1–2) ← START HERE

**Goal:** Full 260-symbol scan with instant load. No more incremental KV cache.

**Step 1.1 — Add HTTPS to existing VPS**

SSH into `77.42.23.93` and run:

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx -y
```

Buy a domain (e.g., `alertcmd.com`) from Cloudflare Registrar (~$10/year). Point an A record to `77.42.23.93`. Then:

```bash
sudo certbot --nginx -d api.yourdomain.com
```

Add nginx config to proxy to your Node.js app:

```nginx
server {
    listen 443 ssl;
    server_name api.yourdomain.com;
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
    }
}
```

This immediately gives you `https://api.yourdomain.com/api/screener/live` — HTTPS, full 260 symbols, instant.

**Step 1.2 — Update index.html to point to your VPS API**

In the GitHub repo, change:

```javascript
const SCREENER_BASE = 'https://screener.purabmehta.workers.dev';
```

to:

```javascript
const SCREENER_BASE = 'https://api.yourdomain.com';
```

Also update `loadScreener` field mapping since the VPS uses different field names:
- `r.price` not `r.lastPrice`
- `r.change24hPct` not `r.priceChangePercent`
- `r.quoteVolume24h` not `r.quoteVolume`

**Step 1.3 — Migrate backend to Railway (when ready to leave VPS)**

1. Create account at `railway.app`
2. Create a new project, connect your GitHub repo (or a new backend repo)
3. Deploy the same Node.js screener code
4. Set environment variables (Binance keys if needed, Supabase URL/key)
5. Railway gives you a URL like `https://screener-production.up.railway.app`
6. Update `SCREENER_BASE` in `index.html` to this new URL
7. Shut down VPS

**Verification:** After Step 1.1, visit `https://api.yourdomain.com/api/screener/live` in browser. Should return JSON with `totalRows: 260`. Screener should show all filter tabs with correct counts matching the old screener exactly.

---

### Phase 2 — Real-Time Alert Pipeline (Week 3–4)

**Goal:** TradingView alert → dashboard in under 200ms. No polling.

**Step 2.1 — Set up Supabase**

1. Create account at `supabase.com`
2. Create a new project (pick US East region)
3. Create these tables in the SQL editor:

```sql
-- Alerts from TradingView
CREATE TABLE alerts (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  received_at TIMESTAMPTZ DEFAULT NOW(),
  symbol TEXT NOT NULL,
  strategy TEXT,
  direction TEXT,
  timeframe TEXT,
  price NUMERIC,
  raw_payload JSONB,
  processed BOOLEAN DEFAULT FALSE
);

-- Open Trades
CREATE TABLE trades (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  opened_at TIMESTAMPTZ DEFAULT NOW(),
  closed_at TIMESTAMPTZ,
  symbol TEXT NOT NULL,
  direction TEXT,
  entry_price NUMERIC,
  exit_price NUMERIC,
  quantity NUMERIC,
  strategy TEXT,
  account_id TEXT,
  status TEXT DEFAULT 'OPEN',
  pnl NUMERIC,
  notes TEXT
);

-- Screener cache
CREATE TABLE screener_cache (
  symbol TEXT PRIMARY KEY,
  data JSONB NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable Realtime on alerts table
ALTER TABLE alerts REPLICA IDENTITY FULL;
```

4. Go to Database → Replication → enable Realtime on the `alerts` table

**Step 2.2 — Build the webhook receiver**

```javascript
// webhook-receiver.js
import express from 'express';
import { createClient } from '@supabase/supabase-js';

const app = express();
app.use(express.json());

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_SERVICE_KEY
);

app.post('/webhook', async (req, res) => {
  const payload = req.body;
  await supabase.from('alerts').insert({
    symbol: payload.symbol,
    strategy: payload.strategy,
    direction: payload.direction,
    timeframe: payload.timeframe,
    price: parseFloat(payload.price),
    raw_payload: payload,
  });
  res.json({ ok: true });
});

app.listen(3001);
```

Deploy this to Railway as a separate service.

**Step 2.3 — Add Supabase Realtime to the dashboard**

```javascript
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

const channel = supabase
  .channel('alerts')
  .on('postgres_changes',
    { event: 'INSERT', schema: 'public', table: 'alerts' },
    (payload) => {
      setAlerts(prev => [payload.new, ...prev]);
    }
  )
  .subscribe();
```

**Step 2.4 — Update TradingView alerts**

Set webhook URL to: `https://your-railway-webhook-url.up.railway.app/webhook`

Alert message format (JSON):

```json
{
  "symbol": "{{ticker}}",
  "strategy": "MACD_12H",
  "direction": "LONG",
  "price": "{{close}}",
  "timeframe": "{{interval}}"
}
```

**Verification:** Fire a test webhook:

```bash
curl -X POST https://your-webhook-url/webhook \
  -H "Content-Type: application/json" \
  -d '{"symbol":"BTCUSDT","strategy":"MACD_12H","direction":"LONG","price":"67000","timeframe":"12h"}'
```

Should appear on dashboard within 200ms.

---

### Phase 3 — Rule Engine (Month 2)

**Goal:** Every incoming alert evaluated against strategy rules before any action is taken.

**Step 3.1 — Define rules as config**

```javascript
export const STRATEGIES = {
  MACD_12H_BULLISH_ENTRY: {
    name: 'MACD 12H Bullish Entry',
    conditions: {
      weeklyRegime: 'BULLISH',
      trigger12h: 'BULLISH_CROSS',
    },
    risk: {
      maxPositionPct: 2,
      stopLossPct: 1.5,
      maxDailyLossPct: 5,
      maxDrawdownPct: 10,
    }
  },
};
```

**Step 3.2 — Rule engine logic**

```javascript
async function evaluateAlert(alert) {
  const strategy = STRATEGIES[alert.strategy];
  if (!strategy) return { action: 'SKIP', reason: 'Unknown strategy' };

  const account = await getAccountSummary(alert.account_id);
  if (account.dailyPnlPct < -strategy.risk.maxDailyLossPct) {
    return { action: 'HALT', reason: 'Daily loss limit reached' };
  }

  const openTrade = await getOpenTrade(alert.symbol, alert.account_id);
  if (openTrade) {
    return { action: 'SKIP', reason: 'Already in trade' };
  }

  return {
    action: 'EXECUTE',
    size: calculatePositionSize(account.equity, strategy.risk.maxPositionPct, alert.price),
  };
}
```

**Step 3.3 — Paper trading mode**

Add `PAPER_TRADING=true` environment variable. When true, logs what it would do but does not send to IB bridge. Run for at least 4–6 weeks before live trading.

---

### Phase 4 — IB Bridge (Month 3–4)

**Goal:** Rule engine → Interactive Brokers orders. Runs on DigitalOcean Droplet.

**Step 4.1 — Set up DigitalOcean Droplet**

1. Create account at `digitalocean.com`
2. Create Droplet: Ubuntu 22.04, 4GB RAM / 2 vCPU ($48/month)
3. SSH in: `ssh root@your-droplet-ip`

```bash
apt update && apt upgrade -y
apt install python3 python3-pip nodejs npm -y
pip3 install ib_insync fastapi uvicorn
```

**Step 4.2 — Install IB Gateway headless**

Download IB Gateway from Interactive Brokers. Use IBC to automate headless login:

```bash
wget https://github.com/IbcAlpha/IBC/releases/latest/download/IBCLinux.zip
unzip IBCLinux.zip -d /opt/ibc
```

Configure IBC with your IB credentials stored as environment variables (never in code).

**Step 4.3 — Python IB Bridge**

```python
from ib_insync import *
from fastapi import FastAPI
import asyncio

app = FastAPI()
ib = IB()

@app.on_event("startup")
async def connect():
    await ib.connectAsync('127.0.0.1', 4002, clientId=1)

@app.post("/order")
async def place_order(payload: dict):
    contract = Stock(payload['symbol'], 'SMART', 'USD')
    order = MarketOrder(
        action=payload['action'],
        totalQuantity=payload['quantity'],
    )
    trade = ib.placeOrder(contract, order)
    await asyncio.sleep(1)
    return {
        'orderId': trade.order.orderId,
        'status': trade.orderStatus.status,
        'filled': trade.orderStatus.filled,
        'avgFillPrice': trade.orderStatus.avgFillPrice,
    }
```

Run with: `uvicorn ib_bridge:app --host 0.0.0.0 --port 8000`

**Step 4.4 — Connect rule engine to IB bridge**

```javascript
await fetch('http://your-droplet-ip:8000/order', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-API-Key': process.env.IB_BRIDGE_KEY
  },
  body: JSON.stringify({
    symbol: alert.symbol,
    action: alert.direction === 'LONG' ? 'BUY' : 'SELL',
    quantity: calculatedSize,
  })
});
```

**Step 4.5 — Secure the bridge**

Use Cloudflare Tunnel (free) to expose the bridge privately — no public IP needed, no port forwarding.

**Verification:** Use IB paper trading account (separate login, same API). Paper trade for 6 weeks minimum before switching to live account.

---

### Phase 5 — AI Performance Audit (Month 5–6)

**Goal:** Nightly Claude review of your trades to catch rule violations.

**Step 5.1 — Nightly audit cron job (Railway)**

```javascript
import Anthropic from '@anthropic-ai/sdk';

const claude = new Anthropic({ apiKey: process.env.CLAUDE_API_KEY });

async function runNightlyAudit() {
  const { data: trades } = await supabase
    .from('trades')
    .select('*')
    .gte('opened_at', today)
    .order('opened_at');

  const response = await claude.messages.create({
    model: 'claude-opus-4-5',
    max_tokens: 1000,
    messages: [{
      role: 'user',
      content: `You are auditing a trader's performance for today.
Strategy rules: [paste your rules here]
Today's trades: ${JSON.stringify(trades, null, 2)}

Identify:
1. Any trades that violated the strategy rules
2. Any patterns (entering too early, moving stops, overtrading)
3. Positive behaviors to reinforce
4. A score from 1-10 for rule-following discipline today`
    }]
  });

  await supabase.from('audits').insert({
    date: today,
    report: response.content[0].text,
    trades_count: trades.length,
  });
}
```

**Step 5.2 — Audit dashboard page (/audit)**

Shows: daily discipline score trend chart, rule violations log, most frequent violation types, win rate filtered by discipline score.

---

### Phase 6 — Multi-Account Dashboard (Month 6+)

**Goal:** View all IB accounts in one place.

**Step 6.1 — Accounts table in Supabase**

```sql
CREATE TABLE accounts (
  id TEXT PRIMARY KEY,
  name TEXT,
  type TEXT,
  equity NUMERIC,
  daily_pnl NUMERIC,
  total_pnl NUMERIC,
  max_drawdown_pct NUMERIC,
  status TEXT,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

IB bridge polls each connected account every 30 seconds and updates Supabase. Dashboard shows all accounts in a grid with equity, daily P&L, drawdown, and health indicator.

---

## 5. Code Reference

### Current index.html Key Constants

```javascript
const SCREENER_BASE = 'https://screener.purabmehta.workers.dev'; // change this in Phase 1
const SUPABASE_URL = '';      // add in Phase 2
const SUPABASE_ANON_KEY = ''; // add in Phase 2
```

### GitHub Workflow (IMPORTANT — always follow this)

Always edit `index.html` via the GitHub web editor using the CM6 dispatch method:

1. Navigate to `github.com/purabmehta/AlertCMD_Claude/edit/main/index.html`
2. Fetch raw file, apply changes in JS, dispatch to CM6 editor view
3. Commit — GitHub Pages auto-deploys in ~90 seconds

**Do NOT use Find+Replace in the GitHub UI** — causes stray character insertion.

Always use:
```javascript
view.dispatch({ changes: { from: 0, to: view.state.doc.length, insert: newCode } })
```

---

## 6. Accounts and Services Reference

| Service | URL | Notes |
|---|---|---|
| GitHub | `github.com/purabmehta/AlertCMD_Claude` | Source of truth for dashboard code |
| Live Dashboard | `purabmehta.github.io/AlertCMD_Claude` | Current live site |
| Cloudflare | `dash.cloudflare.com` | Account ID: `10c90545decda1ce2c6aaa814f498e46` |
| Cloudflare Worker | `screener.purabmehta.workers.dev` | Temporary — to be replaced in Phase 1 |
| Old VPS | `77.42.23.93` | Running full screener, HTTP only, to be decommissioned |
| Supabase | TBD | Set up in Phase 2 |
| Railway | TBD | Set up in Phase 1 or 3 |
| DigitalOcean | TBD | Set up in Phase 4 |

---

## 7. Immediate Next Actions (Start Every New Session Here)

1. **Check screener status:** Visit `https://purabmehta.github.io/AlertCMD_Claude/#/screener` → All/Debug tab. Note how many pairs are showing.

2. **Phase 1 is the priority:** Add SSL to existing VPS (`77.42.23.93`). Fastest path to full 260-symbol coverage. Requires SSH access to the VPS.

3. **Buy a domain first:** Go to `cloudflare.com/products/registrar` — suggest `alertcmd.com`. Needed for Step 1.1.

4. **The Cloudflare Worker stays running as fallback** while VPS HTTPS is being set up.

---

## 8. Key Technical Decisions (Already Made)

- **Single `index.html` for the dashboard** — all React in one file, deployed via GitHub Pages. Keep until app gets significantly more complex.
- **PostgreSQL (Supabase) over MongoDB** — need joins (trades + alerts + accounts).
- **Python for IB bridge** — `ib_insync` is the best IB API wrapper, Python only.
- **Railway over Heroku** — better pricing, no sleep on free tier, better DX.
- **No Redis for now** — Supabase handles caching at this scale. Add Redis if latency becomes an issue after 6 months.
- **No microservices** — keep 3–4 services max until the product is proven.
- **Cloudflare Tunnel for IB bridge** — no public IP needed, free, handles HTTPS.

---

*Document version 1.0. Update after each major phase completion.*
