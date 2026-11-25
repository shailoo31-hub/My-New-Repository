---
name: quickcommerce
description: Live product search, prices, stock and delivery ETA from 22 Indian and UAE shopping apps (Blinkit, Zepto, Swiggy Instamart, BigBasket, DMart, JioMart, Flipkart Minutes, Amazon, Flipkart, Myntra, Nykaa, Meesho, AJIO, Myntra Now, Tira, FirstCry, Croma, Reliance Digital, vivo, OnePlus, Noon Minutes, talabat mart) through one REST API and a hosted MCP server. Use it whenever a task needs a real price, availability, pack size, rating or delivery time for a product at a specific location in India or the UAE, a price comparison across these apps, a price tracker, or proof-of-price screenshots.
---

# QuickCommerce API: live commerce data for AI agents

QuickCommerce API returns live product data, exactly as the app shows it at one delivery location, as clean JSON. One API key works for all 22 platforms. You pay per call from prepaid credits.

**First, get the API key.** If you don't have one, ask the user for their QuickCommerce API key (it starts with `sk_live_`). They can sign up at https://quickcommerceapi.com/auth/signup and get 100 free credits, no card needed. The key is on their dashboard at https://quickcommerceapi.com/dashboard. Send it on every call as the `X-API-Key` header. Never print the key back to the user or write it into a committed file; read it from an environment variable such as `QUICKCOMMERCE_API_KEY`.

- API base: `https://api.quickcommerceapi.com`
- Cost: 1 credit per call. Group calls cost 1 credit per platform. A screenshot costs 25 credits. Listing platforms, checking credits, resolving links and screenshot status are free.
- MCP server: `https://api.quickcommerceapi.com/mcp` (see [Connect via MCP](#connect-via-mcp))
- Rate limit: 100 requests per minute per key.

## The flow

1. **Get a location.** Every data call needs `lat` and `lon` for the delivery address. DMart, JioMart and Flipkart Minutes also need a 6-digit pincode.
2. **Search:** `GET /v1/search` (one platform) or `GET /v1/groupsearch` (several at once) with a keyword.
3. **Refresh one item:** `GET /v1/item` with the `id` from search, to get its current price and stock.
4. **Delivery time:** `GET /v1/eta` or `GET /v1/groupeta` for quick commerce apps.

## 1. Get a location

Results depend on where the order would be delivered, so a wrong location gives wrong prices.

- Ask the user for the delivery area or pincode if the task doesn't give one. Don't guess a city.
- Convert it to `lat`/`lon` yourself. Approximate is fine (the nearest dark store is picked from it).
- For DMart, JioMart and Minutes, also send the pincode, either as the `x-geolocation-pincode` header or the `pincode` query parameter.
- UAE platforms (NoonMinutes, TalabatMart) need a UAE `lat`/`lon`, for example Downtown Dubai `25.1972, 55.2744`, and no pincode. Prices come back in AED. Every product carries a `currency` field (`INR` or `AED`). Don't mix UAE and India platforms in one group call.

Handy Indian coordinates: Bengaluru HSR Layout `12.9116, 77.6389` (560102), Mumbai Andheri West `19.1364, 72.8296` (400053), Delhi Connaught Place `28.6315, 77.2167` (110001), Gurugram Sector 29 `28.4683, 77.0628` (122001), Hyderabad Gachibowli `17.4401, 78.3489` (500032), Pune Kothrud `18.5074, 73.8077` (411038).

## 2. Pick the platform

Platform names are case sensitive. Use them exactly as written here.

| Platform name | What it is | search | item | eta | Notes |
|---|---|---|---|---|---|
| `BlinkIt` | Blinkit (Zomato) | yes | yes | yes | |
| `Zepto` | Zepto | yes | yes | yes | |
| `Swiggy` | Swiggy Instamart | yes | yes | yes | |
| `BigBasket` | BigBasket (Tata) | yes | yes | yes | |
| `DMart` | DMart Ready | yes | yes | yes | needs pincode |
| `JioMart` | JioMart | yes | yes | yes | needs pincode |
| `Minutes` | Flipkart Minutes | yes | yes | yes | needs pincode |
| `Amazon` | Amazon India | yes | yes | no | |
| `Flipkart` | Flipkart marketplace | yes | yes | no | not Minutes |
| `Myntra` | Myntra | yes | yes | no | |
| `Nykaa` | Nykaa | yes | yes | no | |
| `Meesho` | Meesho | yes | yes | no | national prices |
| `Ajio` | AJIO | yes | yes | no | national prices; pincode adds delivery date |
| `MyntraNow` | Myntra Now (quick delivery) | yes | yes | no | results depend on lat/lon |
| `Tira` | Tira Beauty (Reliance) | yes | yes | no | each shade is its own item |
| `FirstCry` | FirstCry (baby and kids) | yes | yes | no | each size or pack is its own item |
| `RelianceDigital` | Reliance Digital | no | yes | no | item only, pincode-aware price |
| `Croma` | Croma | no | yes | no | item only, pincode-aware price |
| `Vivo` | vivo India store | no | yes | no | item only, id `spuId:skuId` |
| `OnePlus` | OnePlus India store | no | yes | no | item only, id `productCode:skuCode` |
| `NoonMinutes` | Noon Minutes (UAE) | yes | yes | yes | AED |
| `TalabatMart` | talabat mart (UAE) | yes | yes | yes | AED |

For "groceries" or "quick commerce" with no platform named, use `BlinkIt,Zepto,Swiggy,BigBasket` (add `DMart,JioMart,Minutes` when you have a pincode). The live list is at `GET /v1/supported-platforms` (free, no key).

## 3. Search

```bash
curl -H "X-API-Key: $QUICKCOMMERCE_API_KEY" \
  "https://api.quickcommerceapi.com/v1/search?q=amul+butter&platform=BlinkIt&lat=12.9116&lon=77.6389"
```

Response (trimmed):

```json
{
  "status": "success",
  "request_id": "a1b2c3d4-...",
  "credits_remaining": 99,
  "data": {
    "query": "amul butter",
    "platform": "BlinkIt",
    "total_results": 30,
    "products": [
      {
        "id": "36961",
        "name": "Amul Salted Butter",
        "brand": "Amul",
        "quantity": "100 g",
        "available": true,
        "mrp": 62,
        "offer_price": 60,
        "currency": "INR",
        "inventory": 12,
        "rating": 4.6,
        "rating_count": 21000,
        "deeplink": "https://blinkit.com/prn/x/prid/36961",
        "images": ["https://..."],
        "platform": { "name": "BlinkIt", "sla": "8 mins", "open": true }
      }
    ]
  }
}
```

**Compare several platforms in one call:**

```bash
curl -H "X-API-Key: $QUICKCOMMERCE_API_KEY" -H "x-geolocation-pincode: 560102" \
  "https://api.quickcommerceapi.com/v1/groupsearch?q=amul+butter&platforms=BlinkIt,Zepto,Swiggy,DMart&lat=12.9116&lon=77.6389"
```

The response has `results` keyed by platform name, each a list of products, plus `credit_cost`.

**Matching the same product across platforms.** Each app names products differently. Match on brand, then pack size (`quantity`), then name. Compare `offer_price` only between the same pack size, or compute price per unit. Skip products with `available: false` unless the user asked about stock. Search results are ranked by the platform, so the first result is not always the exact product.

## 4. Item details

Use the `id` from search as `item_id` to get the current price and stock of one product.

```bash
curl -H "X-API-Key: $QUICKCOMMERCE_API_KEY" \
  "https://api.quickcommerceapi.com/v1/item?item_id=36961&platform=BlinkIt&lat=12.9116&lon=77.6389"
```

`data.items` is always an array (some platforms return several variants). Each item has `item_id`, `name`, `brand`, `quantity`, `available`, `price`, `mrp`, `inventory`, `deeplink`, `store_id` and `fetched_at`.

- **Several items on one platform:** `GET /v1/items?item_ids=id1,id2,...&platform=...&lat=...&lon=...` (up to 10 ids, 1 credit per item found).
- **Have a product link, not an id?** `GET /v1/item/resolve?url=<product link>` returns the `item_id` and `platform` for `/v1/item`. It is free and works with share and short links.
- **Item-only stores** take ids from their product links: Croma is the number after `/p/` (e.g. `317418`), Reliance Digital is the slug after `/product/`, vivo is `spuId:skuId` (e.g. `10373:19585`), OnePlus is `productCode:skuCode` (e.g. `P1120230:5011122818`).

## 5. Delivery ETA

Quick commerce platforms only: BlinkIt, Zepto, Swiggy, BigBasket, DMart, JioMart, Minutes, NoonMinutes, TalabatMart.

```bash
curl -H "X-API-Key: $QUICKCOMMERCE_API_KEY" -H "x-geolocation-pincode: 560102" \
  "https://api.quickcommerceapi.com/v1/groupeta?platforms=BlinkIt,Zepto,Swiggy,DMart&lat=12.9116&lon=77.6389"
```

Each result has `platform`, `eta`, `open` and `store_id`. For one platform, use `GET /v1/eta?platform=...&lat=...&lon=...`. A closed store (`open: false`) means that app cannot deliver there right now.

## 6. Proof-of-price screenshot (optional, 25 credits)

`GET /v1/screenshot` opens the platform's own product page at the delivery location and returns a PNG, a one-page PDF and optionally a video, plus `page_price` compared against `api_price`. Use it only when the user asks for proof or evidence.

```bash
curl -H "X-API-Key: $QUICKCOMMERCE_API_KEY" \
  "https://api.quickcommerceapi.com/v1/screenshot?platform=BlinkIt&item_id=36961&lat=12.9116&lon=77.6389&sync=true"
```

With `sync=true` the call waits up to 90 s (set your HTTP timeout to 100 s). Without it, poll `GET /v1/screenshot/status?job_id=...` (free) every few seconds until `status` is `done` or `failed`. A failed capture is refunded. Download links last 1 hour.

## Ask before you spend

- Check the balance first for large jobs: `GET /v1/credits` (free) returns `summary.total_available`.
- Tell the user the cost before a run of more than about 50 credits (for example, 200 products x 4 platforms = 800 credits) and wait for a yes.
- Every response carries `X-Credits-Remaining` and `X-Request-Id` headers.
- Stay under 100 requests per minute. For bulk work, run about 5 to 10 calls in parallel, not more.

## Copy-paste helper (Python)

```python
import os, time, requests

BASE = "https://api.quickcommerceapi.com"
HEADERS = {"X-API-Key": os.environ["QUICKCOMMERCE_API_KEY"]}

def qc(path: str, retries: int = 3, **params):
    """Call a QuickCommerce endpoint and return its JSON. Retries 429 and 5xx."""
    pincode = params.pop("pincode", None)
    headers = {**HEADERS, **({"x-geolocation-pincode": str(pincode)} if pincode else {})}
    for attempt in range(retries + 1):
        r = requests.get(f"{BASE}{path}", params=params, headers=headers, timeout=60)
        if r.status_code == 429 or r.status_code >= 500:
            if attempt < retries:
                time.sleep(int(r.headers.get("Retry-After", 2 ** (attempt + 1))))
                continue
        if not r.ok:
            raise RuntimeError(f"{r.status_code} {r.text} (request {r.headers.get('X-Request-Id')})")
        return r.json()

# Cheapest Amul Butter 500 g near HSR Layout, Bengaluru
res = qc("/v1/groupsearch", q="amul butter 500g", platforms="BlinkIt,Zepto,Swiggy,DMart",
         lat=12.9116, lon=77.6389, pincode="560102")
for platform, products in res["results"].items():
    for p in products[:3]:
        print(platform, p["name"], p.get("quantity"), p.get("offer_price"), p.get("available"))
```

## When something goes wrong

Errors return JSON with an `error` code and a readable `message` (inside `detail` for most endpoints). Branch on the HTTP status and `error` code, not the message.

| Status | Meaning | What to do |
|---|---|---|
| 401 | Missing or invalid API key | Ask the user for a valid key. |
| 402 | Out of credits | Tell the user to top up at https://quickcommerceapi.com/pricing. Don't retry. |
| 404 | `item_not_found` or `store_closed` | The item isn't sold or the store is shut at this location. Try another location or report it. Don't retry. |
| 422 | Bad parameter or platform name | Fix the request (check the exact platform name, add the pincode). |
| 429 | Rate limit | Wait and retry with backoff. |
| 502 / 504 | The platform itself failed or timed out | Retry up to 3 times with backoff. Credits are refunded when a call fails. |

- An empty `products` list is a normal answer: the item isn't sold near that location.
- In group calls one platform can fail while others succeed. Report what you got and name the platforms that failed.
- Quote the `X-Request-Id` header when contacting support at https://quickcommerceapi.com/contact.

## All endpoints

All are `GET` on `https://api.quickcommerceapi.com` with the `X-API-Key` header.

| Endpoint | Purpose | Credits |
|---|---|---|
| `/v1/search` | Search one platform (`q`, `platform`, `lat`, `lon`) | 1 |
| `/v1/groupsearch` | Search several platforms (`q`, `platforms`, `lat`, `lon`) | 1 per platform |
| `/v1/item` | One item's live price and stock (`item_id`, `platform`, `lat`, `lon`) | 1 |
| `/v1/items` | Up to 10 items on one platform (`item_ids`, `platform`, `lat`, `lon`) | 1 per item found |
| `/v1/item/resolve` | Product link to `item_id` (`url`, or `sku_id` + `platform`) | free |
| `/v1/eta` | Delivery ETA for one platform (`platform`, `lat`, `lon`) | 1 |
| `/v1/groupeta` | ETAs for several platforms (`platforms`, `lat`, `lon`) | 1 per platform |
| `/v1/screenshot` | Product page screenshot as proof of price | 25 |
| `/v1/screenshot/status` | Screenshot job status and links (`job_id`) | free |
| `/v1/credits` | Credit balance and packs | free |
| `/v1/supported-platforms` | Platform lists by capability and country (no key needed) | free |

Pincode, where needed: `x-geolocation-pincode` header or `pincode` query parameter.

## Connect via MCP

Hosted server, Streamable HTTP: `https://api.quickcommerceapi.com/mcp`. Auth is the `X-API-Key` header.

| Tool | What it does | Credits |
|---|---|---|
| `search_products` | Search one platform | 1 |
| `group_search` | Search several platforms | 1 per platform |
| `get_item_details` | Live price and stock for one item | 1 |
| `check_delivery_eta` | ETA and store status for one platform | 1 |
| `group_eta` | ETAs for several platforms | 1 per platform |
| `capture_product_screenshot` | Proof-of-price screenshot | 25 |
| `get_screenshot_status` | Screenshot job status and links | free |
| `check_credits` | Credit balance | free |
| `list_platforms` | Supported platforms | free |

**Claude Code:**

```bash
claude mcp add --transport http quickcommerce https://api.quickcommerceapi.com/mcp --header "X-API-Key: sk_live_xxxx"
```

**Cursor, Claude Desktop and other MCP clients** (`mcp.json` or equivalent):

```json
{
  "mcpServers": {
    "quickcommerce": {
      "url": "https://api.quickcommerceapi.com/mcp",
      "headers": { "X-API-Key": "sk_live_xxxx" }
    }
  }
}
```

**Claude.ai and ChatGPT:** add a custom connector with the server URL. Setup guide: https://quickcommerceapi.com/mcp

## More

- Human docs: https://quickcommerceapi.com/docs
- Full reference for models: https://quickcommerceapi.com/llms-full.txt
- Try calls in the browser: https://quickcommerceapi.com/playground
- Pricing: https://quickcommerceapi.com/pricing
- Bulk catalogue and price history datasets: https://quickcommercedata.com
