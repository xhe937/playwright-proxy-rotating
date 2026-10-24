# playwright proxy python: Set Up Rotating Proxies in Playwright the Right Way, Avoid the Auth Error, and Pick a Plan That Matches Your Credit Math

Setting a proxy in Playwright with Python takes two lines. Making that setup survive a real scraping run against a site with anti-bot protection is where things get complicated, because the proxy itself doesn't solve CAPTCHAs, rotate residential IPs, or render JavaScript-heavy pages for you. This guide covers the native Playwright proxy setup in Python, the one integration trap that breaks most people's first attempt with a scraping API, and what the whole thing actually costs once you attach a service like ScraperAPI to the pipeline.

> **Quick picks:** If you just want a working setup, skip the proxy port entirely and route Playwright through the ScraperAPI endpoint — the free trial gives you 5,000 credits with no credit card. For paid use, Hobby ($49/month, 100,000 credits) covers small projects, and Scaling ($475/month, 5,000,000 credits) is the plan the site itself marks as most popular. 👉 [Try ScraperAPI free — 5,000 credits, no credit card required](https://www.scraperapi.com/signup?fp_ref=coupons)

## How proxy support works in Playwright Python

Playwright accepts a proxy dictionary at two levels, and knowing the difference saves you a lot of memory later on.

At the browser level, every page in the process uses the same proxy:

python
from playwright.sync_api import sync_playwright

proxy = {"server": "http://user:pass@proxy-host:8000"}

with sync_playwright() as p:
    browser = p.chromium.launch(proxy=proxy)
    page = browser.new_page()
    page.goto("https://httpbin.org/ip")


At the context level, each context gets its own proxy while sharing the same browser process:

python
context = browser.new_context(proxy={
    "server": "http://proxy-host:8000",
    "username": "user",
    "password": "pass",
})


The `proxy` object has four fields: `server` (required, and it must include the protocol — Playwright won't add `http://` for you), plus optional `username`, `password`, and `bypass` (a comma-separated list of hosts that skip the proxy). Both HTTP and HTTPS proxies are written as `http://...`; a single HTTP proxy tunnels HTTPS traffic via CONNECT. SOCKS goes in as `socks5://host:port`.

The context-level option is the reason Playwright handles rotation better than Selenium or Puppeteer. In Selenium, every new proxy means a whole new WebDriver session; in Puppeteer, you typically launch a new browser. Playwright runs dozens of isolated contexts — each with its own cookies, cache, and storage — inside one Chromium process. A single process running 20 contexts on 20 different proxies uses a fraction of the RAM of 20 separate browsers, which on a small scraping box is often the difference between 4 concurrent workers and 40.

One limitation worth memorizing: **Playwright does not support authenticated SOCKS5**. It will connect to a SOCKSv5 proxy, but `username` and `password` get silently dropped, so a SOCKS proxy has to accept you by IP whitelisting. If your proxy needs credentials, use an HTTP endpoint instead.

Another detail teams miss: match the browser's `locale` and `timezone_id` to the proxy's geography. A US residential IP paired with a browser reporting `Europe/Berlin` is an obvious mismatch that anti-bot systems flag.

## Why ScraperAPI's proxy port fails inside Playwright

You'll find older forum posts suggesting you plug ScraperAPI's proxy port (`proxy-server.scraperapi.com:8001`) directly into `launch()`. That approach fails, and the failure is structural rather than a config typo.

The error you get is `IP Details: Proxy Authentication Required`. ScraperAPI authenticates via API key passed as a URL query parameter. Playwright's proxy configuration only supports Basic Auth (username + password) or IP-based auth — it has no mechanism to send query strings through a proxy connection. The two authentication models simply don't mix, which is why the official ScraperAPI guide explicitly tells Playwright users to avoid the built-in proxy settings.

There's a clean way around this, and it's the method ScraperAPI documents for exactly this use case.

## The method that works: route Playwright through the ScraperAPI endpoint

Instead of pointing Playwright at the target site, you point it at a ScraperAPI URL that wraps the target. ScraperAPI's server side handles proxy rotation, headless rendering, CAPTCHA solving, and retries, then hands back the final HTML. Your Playwright code — `page.locator()`, `$$eval`, screenshots, network interception — works on that response exactly as before.

Setup takes three steps:

1. Install the Python bindings: `pip install playwright` then `playwright install chromium`.
2. Store your key in a `.env` file at the project root: `SCRAPERAPI_KEY=your_api_key_here` — no quotes around the value; that trips up a lot of first runs. Load it with `python-dotenv`.
3. Build the wrapped URL and `goto()` it. Here's the async version with JavaScript rendering enabled:

python
import os, asyncio
from urllib.parse import quote
from dotenv import load_dotenv
from playwright.async_api import async_playwright

load_dotenv()
SCRAPERAPI_KEY = os.environ["SCRAPERAPI_KEY"]

async def main():
    target_url = "https://httpbin.org/ip"
    scraper_api_url = (
        f"https://api.scraperapi.com?api_key={SCRAPERAPI_KEY}"
        f"&render=true&url={quote(target_url)}"
    )
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        await page.goto(scraper_api_url, wait_until="domcontentloaded")
        print(await page.inner_text("body"))
        await browser.close()

asyncio.run(main())


If the printed IP is ScraperAPI's exit node rather than your own, the integration works. The sync API follows the same structure with `sync_playwright` and no `await`s.

The parameters you'll actually use:

| Parameter | What it does |
| --- | --- |
| `render=true` | Headless Chromium renders the page server-side |
| `country_code=us` | Routes through US IPs (free) |
| `session_number=123` | Sticks to one proxy IP across requests — good for logins and pagination (free) |
| `premium=true` | Premium residential pool for harder targets |
| `ultra_premium=true` | Highest-tier pool; paid plans only |
| `wait_for_selector=.product-title` | Waits for a CSS selector before returning (needs `render=true`, free) |

A typical JS-rendered, US-geotargeted request looks like `...&render=true&country_code=us&url=...`.

## What a request actually costs in credits

This is the part most tutorials skip, and it changes which plan makes sense.

Standard pages cost 1 credit. Parameters add on top: `render=true` +10, `premium=true` +10, `screenshot=true` +10, `ultra_premium=true` +30. Here's the catch that consistently surprises people: **the stacking is non-linear**. `render=true` plus `premium=true` costs +25, not +20. `ultra_premium=true` plus `render=true` costs +75, not the +40 you'd get from adding the parts. Run your numbers before kicking off a large batch.

Domain-based multipliers apply automatically and you can't opt out. Amazon requests cost 5 credits, Google and Bing SERPs cost 25, LinkedIn costs 30, and sites behind Cloudflare, DataDome, or PerimeterX add 10 credits per request when ScraperAPI bypasses them.

Put together, the Hobby plan's "100,000 credits" means very different things: roughly 100,000 requests against simple HTML blogs, about 6,600 Amazon product pages with `render=true`, or around 4,000 Google SERP pages. This is why testing against your real targets during the trial matters more than the headline credit number — the dashboard has a Domain Cost Estimator for exactly this, and you can set a `max_cost` per scrape so a single request can't blow past your limit. Every response also carries a `sa-credit-cost` header showing what that request consumed; log it during development so you know your real cost curve before scaling.

Failed scrapes don't burn credits — anything other than a 200 or 404 response costs nothing.

## ScraperAPI plans and pricing

All seven paid tiers share the same core feature set: JS rendering, premium residential and mobile IPs, CAPTCHA and anti-bot bypassing, JSON auto-parsing, custom headers and sessions, desktop/mobile user agents, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. Annual billing takes 10% off every plan.

| Plan | Monthly price | Annual price | API credits | Concurrent threads |
| --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 |
| Hobby | $49 | $44.10/mo | 100,000 | 20 |
| Startup | $149 | $134.10/mo | 1,000,000 | 50 |
| Business | $299 | $269.10/mo | 3,000,000 | 100 |
| Scaling | $475 | $427.50/mo | 5,000,000 | 200 |
| Professional | $975 | $877.50/mo | 10,500,000 | 300 |
| Advanced | $1,975 | $1,777.50/mo | 21,500,000 | 500 |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ |

[Start Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) · [Start Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) · [Start Business](https://www.scraperapi.com/pricing?fp_ref=coupons) · [Start Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) · [Start Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) · [Start Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons)

A few plan-level details that don't show up in the headline numbers:

- **Geotargeting is restricted on the lower tiers.** Hobby and Startup only get US and EU IPs and a 30-day analytics window. Country-level targeting and unlimited analytics start at Business.
- **Exceeding your thread cap returns a 429.** Requests are rejected rather than queued, and you aren't charged for them.
- **When credits run out, scraping pauses by default.** Hobby through Professional can opt into auto-upgrade, which bumps you to the next plan automatically. Scaling and above can instead use pay-as-you-go at a fixed per-credit rate, capped at a monthly spending limit you set — so a runaway loop can't drain the account.
- 👉 [See current plan prices and the annual discount on ScraperAPI](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Which plan fits your workload

The decision is mostly a function of your target sites and request volume, not of features — every paid plan includes the same bypass and rendering stack.

- **Mostly static HTML, a few thousand pages a month:** Free plan (1,000 credits, 5 concurrent connections) or Hobby. If your targets are plain server-rendered pages, leave `render=true` off and the credits stretch nearly 1:1.
- **JS-heavy e-commerce at moderate volume:** Startup. Remember the multiplier math — an Amazon product page with rendering costs 5–15 credits depending on your flags, so 1,000,000 credits is closer to 60,000–70,000 product pages than a million requests.
- **Search engine scraping or LinkedIn:** Do the division first. At 25–30 credits per request, Google SERPs alone can eat a Business plan's 3,000,000 credits in about 120,000 queries. If that's your main workload, price out Scaling or Professional before committing.
- **Production pipelines with no tolerance for pauses:** Scaling or above, with pay-as-you-go enabled and a monthly cap set.

👉 [Start the free trial and test your actual target URLs before picking a tier](https://www.scraperapi.com/?fp_ref=coupons)

## When you don't need a scraping API at all

If your targets have no serious anti-bot protection, native Playwright proxies may be all you need. Static sites with light rate limits run fine on a pool of datacenter proxies distributed across contexts, and the per-context model keeps resource usage low. Self-managed proxy providers in that space typically charge per port or per GB, which works out cheaper than credit-based APIs for easy targets at high volume.

The trade-off is maintenance. Running your own stack means handling CAPTCHA solving, fingerprint consistency, retries, and IP pool quality yourself — and the anti-bot arms race doesn't pause. A reasonable rule: if your Playwright setup rarely gets blocked and costs you almost nothing to run, keep it. If you're burning engineering hours fighting Cloudflare challenges, a managed API is the better spend.

## FAQ

**Can Playwright use SOCKS5 proxies?**
Yes, with `socks5://` in the `server` field. But authenticated SOCKS5 is not supported — credentials are silently dropped, so the proxy must accept you by IP whitelisting. For credential-based auth, use an HTTP proxy.

**Why do I get "Proxy Authentication Required" with ScraperAPI in Playwright?**
Because ScraperAPI's proxy port authenticates with a query-string API key, and Playwright's proxy config only supports Basic Auth or IP auth. Use the API endpoint method instead: wrap the target URL and `page.goto()` the wrapped address.

**Does ScraperAPI charge for failed requests?**
No. Only 200 and 404 responses consume credits; everything else is free.

**Do `country_code` and `session_number` cost extra credits?**
No. Geotargeting, session sticking, `device_type`, `output_format`, `keep_headers`, `autoparse`, and `wait_for_selector` are all free add-ons.

**Is there a free option to test with?**
Yes — 1,000 free credits on the free plan (5 concurrent connections), or the 7-day trial with 5,000 credits, no credit card required. Support can grant extra testing credits if you ask.

## Picking between the two approaches

Static or lightly protected targets → plain Playwright with per-context proxies, and stop reading here. JS-heavy pages, login flows, Google/Amazon/LinkedIn, or anything behind Cloudflare or DataDome → the ScraperAPI endpoint method with `render=true`, starting from Hobby and letting auto-upgrade tell you when you've outgrown it. Before paying for anything, spend the trial credits on your real targets and check the `sa-credit-cost` header — a week of that data is worth more than any plan comparison table.

One last pre-purchase check: confirm your targets in the Domain Cost Estimator, since Google at 25 credits and Amazon at 5 will reshape your math faster than any feature list. 👉 [Get started with ScraperAPI — free trial, 5,000 credits, no credit card](https://www.scraperapi.com/?fp_ref=coupons)
