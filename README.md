# Helixo Countdown Timer — Shopify App

A Shopify app that allows merchants to add customizable countdown timers to product pages, creating urgency for promotions and boosting conversions.

## Features

- **Fixed Timers**: Set specific start/end dates — all visitors see the same countdown
- **Evergreen Timers**: Session-based timers that start when a visitor arrives (stored in localStorage)
- **Product/Collection Targeting**: Apply timers to all products, specific products, or specific collections using Shopify Resource Picker
- **Full Customization**: Background color, text color, accent color, position, display text, urgency messages
- **Visual Urgency Cues**: Pulsing urgency text when timer nears expiry
- **Basic Analytics**: Impression tracking per timer
- **Live Preview**: See timer styling in real-time while editing

## Architecture Decisions

### Tech Stack
- **Backend**: Node.js + Express (Shopify CLI node-express template)
- **Frontend Admin**: React + Shopify Polaris UI components
- **Database**: MongoDB Atlas (multi-tenant, indexed by shop domain)
- **Storefront Widget**: Vanilla JS in a Liquid Theme App Extension (no framework overhead)
- **Session Storage**: SQLite (Shopify default for session tokens)

### Why These Choices

1. **Theme App Extension over ScriptTag**: Theme extensions are the modern Shopify approach — they survive app uninstalls, work with Online Store 2.0 themes, and don't require theme code injection. Merchants can position the block in the theme editor.

2. **MongoDB over SQL**: Timer configs are document-shaped (nested style objects, variable-length arrays for targeting). MongoDB's flexible schema fits this naturally. Multi-tenant isolation is handled via `shop` field indexing.

3. **Vanilla JS widget over Preact/React**: The storefront widget uses plain JavaScript (~3KB) rather than bundling a framework. This keeps the widget well under the 30KB gzipped target and avoids impacting storefront performance.

4. **Separate storefront API**: The `/api/storefront/*` endpoints are public (no Shopify session auth) but rate-limited and validated by shop domain. This allows the widget to fetch timer data without going through Shopify's auth flow.

5. **`Cache-Control` headers**: Storefront timer responses are cached for 60 seconds, reducing DB load while keeping timers reasonably fresh.

### Data Architecture

```
Timer {
  shop: String (indexed)         // Multi-tenant isolation
  title: String
  timerType: "fixed" | "evergreen"
  startDate/endDate: Date        // For fixed timers
  evergreenDuration: Number      // Minutes, for evergreen timers
  targetType: "all" | "specific_products" | "specific_collections"
  targetProductIds: [String]
  targetCollectionIds: [String]
  style: {                       // Full appearance customization
    backgroundColor, textColor, accentColor,
    position, displayText, urgencyText,
    urgencyThresholdMinutes, showLabels
  }
  isActive: Boolean
  impressions: Number            // Analytics counter
}
```

Indexes: `{ shop, isActive }` and `{ shop, targetType }` for efficient queries.

## Setup Instructions

### Prerequisites
- Node.js 18+
- npm 8+
- Shopify CLI 3.x (`npm install -g @shopify/cli@latest`)
- MongoDB Atlas account (free tier works)
- Shopify Partners account with a development store

### Installation

```bash
# 1. Clone the repository
git clone <repo-url>
cd helixo-dev

# 2. Install dependencies
npm install
cd web && npm install
cd frontend && npm install
cd ../..

# 3. Configure environment
# Create web/.env with your MongoDB connection string:
echo "MONGODB_URI=mongodb+srv://<user>:<password>@<cluster>.mongodb.net/helixo-timers?retryWrites=true&w=majority" > web/.env

# 4. Run the app
shopify app dev
```

The CLI will prompt you to select a dev store and open the app in your browser.

### Running Tests

```bash
cd web
npm test
```

### Linting

```bash
cd web
npm run lint
```

## API Documentation

All admin endpoints require Shopify session authentication (handled automatically by the app).

### Admin API (Authenticated)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/timers` | List all timers for the current shop |
| `GET` | `/api/timers/:id` | Get a single timer by ID |
| `POST` | `/api/timers` | Create a new timer |
| `PUT` | `/api/timers/:id` | Update an existing timer |
| `DELETE` | `/api/timers/:id` | Delete a timer |
| `POST` | `/api/timers/:id/toggle` | Toggle timer active/inactive |

### Storefront API (Public, Rate-Limited)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/storefront/timers?shop=X&productId=Y&collectionIds=Z` | Get active timers matching a product |
| `POST` | `/api/storefront/impression` | Track a timer impression |

### Rate Limiting

Storefront endpoints are rate-limited to **120 requests per minute per IP** using `express-rate-limit`. Admin endpoints are protected by Shopify's session authentication.

### Input Validation

All timer creation/update requests are validated using `express-validator`:
- Title: required, max 200 chars, HTML-escaped
- Timer type: must be "fixed" or "evergreen"
- Dates: ISO 8601 format, end date must be after start date
- Duration: 1–10080 minutes
- Colors: hex format (#XXXXXX)
- Shop domain: validated against `*.myshopify.com` pattern

### Error Response Format

```json
{
  "success": false,
  "errors": [
    { "field": "title", "message": "Title is required" }
  ]
}
```

## Performance Optimizations

1. **Widget size**: ~3KB (vanilla JS, no framework) — well under 30KB gzipped target
2. **No CLS**: Timer container uses `display:none` until data loads, then appears without shifting layout
3. **Caching**: 60-second Cache-Control on storefront API responses
4. **Efficient queries**: MongoDB compound indexes on `{ shop, isActive }` and `{ shop, targetType }`
5. **Minimal payload**: Storefront API returns only fields needed by the widget (id, type, endDate/duration, style)
6. **Tabular nums**: Timer digits use `font-variant-numeric: tabular-nums` to prevent width jitter during countdown

## Security

- All admin API calls validated via Shopify session authentication
- Shop ownership verified on every database query (timers filtered by `shop` field)
- User inputs sanitized with `express-validator` (XSS prevention via `.escape()`)
- Shop domain parameter validated against `*.myshopify.com` regex pattern
- No sensitive data (API keys, tokens) exposed in client-side code
- `.env` file excluded from git via `.gitignore`

## Assumptions

1. One timer is shown per product page (first matching active timer takes priority)
2. Evergreen timer state is stored in the visitor's `localStorage` — clearing browser data resets it
3. Impression tracking counts page loads, not unique visitors
4. The widget fetches timer data on each page load (cached for 60s server-side)
5. Timer dates are stored in UTC; the countdown widget displays relative time remaining

## What I Would Improve With More Time

1. **Bulk operations**: Select and delete/pause multiple timers at once
2. **Richer analytics**: Click-through tracking, conversion attribution, charts over time
3. **A/B testing**: Compare timer styles and measure which converts better
4. **Scheduling queue**: Use a job scheduler to auto-deactivate expired timers in the background
5. **Widget CDN**: Serve the widget JS from a CDN instead of inline in Liquid for better caching
6. **E2E tests**: Cypress/Playwright tests for the full admin flow
7. **Webhook cleanup**: Listen for `APP_UNINSTALLED` to clean up a shop's timers from MongoDB
