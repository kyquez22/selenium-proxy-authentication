# selenium proxy authentication python: Working Fixes for Chrome's Auth Popup, Plus the Proxy-Port Shortcut Most Tutorials Skip

If you're searching for **selenium proxy authentication python**, you've probably already hit the wall: Chrome headless silently rejects `http://user:pass@host:port`, the old in-memory extension trick stopped working in recent Chrome versions, and half the Stack Overflow answers point to libraries that don't even install on modern Python anymore. This guide covers the three methods that still work in current Selenium 4.x, then shows a shortcut — a gateway-style proxy like ScraperAPI — that sidesteps the authentication problem entirely, along with its full current pricing so you can decide whether that route is worth the money.

> **Quick pick:** if you just need authenticated proxies working in Selenium today and don't want to write CDP auth-handling code, a proxy port service is the low-effort route. ScraperAPI gives you 1,000 free credits/month plus a 7-day trial of 5,000 credits with no credit card, and its paid plans start at **$49/month** (Hobby). 👉 [Grab the free 5,000-credit trial and test it in your own Selenium script](https://www.scraperapi.com/?fp_ref=coupons)

## Why username/password proxies fail in Selenium

The short answer: Chrome headless blocks credentials embedded in the proxy URL, and every workaround built around that limitation has been getting less reliable.

Passing a proxy without authentication is easy — `options.add_argument('--proxy-server=http://1.2.3.4:8080')` works for HTTP, HTTPS, and SOCKS (with the right prefix), and you can verify it by loading `httpbin.org/ip` and checking the returned IP. The problem starts when your proxy requires a username and password, which is how most commercial proxies are sold: a single endpoint plus credentials.

Three things break here:

1. **The URL form is blocked.** Chrome blocks `http://user:pass@host:port` in headless mode, so the "just put credentials in the URL" approach silently does nothing.
2. **The in-memory Chrome extension trick has become unreliable.** Chrome now silently disables `--load-extension` for non-test invocations, and Manifest V3's service-worker lifecycle means the `webRequest.onAuthRequired` listener often hasn't started by the time the first navigation fires. The extension loads, the listener doesn't, and you get an auth popup nobody answers.
3. **Selenium Wire itself is shaky on modern Python.** Selenium Wire was the standard fix for years — and it's still what ScraperAPI's official integration guide recommends — but it no longer imports on current Python versions. If it works in your environment, great; if you're on a fresh setup, plan for one of the alternatives below.

One more thing worth fixing before you start: Selenium 4.x syntax. The old `chrome_options=` and `executable_path=` arguments are deprecated and break on Selenium 4.10+. Use `options=` and a `Service` object instead. Since Selenium 4.6, Selenium Manager downloads the driver for you, so you don't need to manage driver paths at all.

## Method 1: Intercept the auth challenge via Chrome DevTools Protocol

For a username/password proxy in current Chrome, the cleanest approach is answering the proxy's authentication challenge through the Chrome DevTools Protocol — specifically `Fetch.continueWithAuth`. It has none of the timing issues that killed the extension trick, because you're responding to auth requests directly instead of hoping a background listener is alive.

The implementation outline:

1. Install `websocket-client` (`pip install websocket-client`).
2. Add `--remote-allow-origins=*` to your Chrome options so the debugger connection is accepted.
3. Read the debugger address from `capabilities["goog:chromeOptions"]["debuggerAddress"]` and connect to the page target's websocket.
4. Enable the `Fetch` domain, then when you receive a `Fetch.authRequired` event, respond with `Fetch.continueWithAuth` using `ProvideCredentials` and the username/password. When you get `Fetch.requestPaused`, send `Fetch.continueRequest` to let navigation proceed.

It's more code than the old tricks, but it's the method that actually survives current Chrome. This is also the right moment to decide whether writing and maintaining this plumbing is worth it to you — if you'd rather skip it, the proxy-port approach later in this article removes the need for any of it.

## Method 2: Let SeleniumBase handle the auth for you

SeleniumBase wraps proxy authentication internally, so the whole problem reduces to one argument:

python
pip install seleniumbase


python
import time
from seleniumbase import Driver

driver = Driver(headless=True, proxy="USER:PASS@IP:PORT")
driver.get("https://whatismyip.com")
time.sleep(6)
driver.quit()


That's the entire setup. If your goal is "authenticated proxy + headless Chrome, minimal code," this is the fastest path that doesn't involve writing CDP handlers yourself.

## Method 3: Skip credentials with IP whitelisting

Many proxy providers can bind your account to your server's IP address, which makes the credentials unnecessary — an unauthenticated `--proxy-server=http://host:port` then works as-is. This is worth checking before writing any auth code, because it turns a hard problem into a non-problem. The obvious trade-off: whitelisting breaks whenever your script runs from a machine with a changing IP (laptops, serverless, rotating cloud instances), so it fits dedicated servers better than distributed setups.

## The gateway shortcut: ScraperAPI's proxy port with Selenium

All three methods above share the same burden: your browser has to answer the proxy's authentication challenge itself. A gateway-style scraping API moves that problem off your machine — proxies rotate on the server side, so your browser only ever talks to one fixed proxy address, and the authentication is handled inside the local proxy layer rather than by Chrome. That's why the setup below uses a `user:pass@host:port` URL freely: Selenium Wire's built-in local proxy does the credential handshake, which Chrome headless would refuse to do on its own.

ScraperAPI publishes an official Python Selenium integration that works exactly this way. The recommended setup replaces native Selenium with Selenium Wire and points it at the proxy port:

python
pip install selenium-wire


python
from seleniumwire import webdriver  # must come from seleniumwire, not plain selenium

API_KEY = 'YOUR_API_KEY'

proxy_options = {
    'proxy': {
        'http': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'https': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'no_proxy': 'localhost,127.0.0.1'
    }
}

driver = webdriver.Chrome(
    ChromeDriverManager().install(),
    options=option,
    seleniumwire_options=proxy_options
)


The import matters: this only works with the webdriver from `seleniumwire`, which is what accepts the `seleniumwire_options` argument. After that, every request Selenium makes goes through the proxy port, and the rest of your script is normal Selenium. Two caveats worth knowing before you commit:

- **Selenium Wire is still the officially documented method**, but as noted above it no longer imports on modern Python. If your project runs on an older environment this is fine; otherwise, SeleniumBase (Method 2) supports the same `user:pass@host:port` proxy string and can point at ScraperAPI's proxy port too.
- **Don't use the API endpoint method with Selenium.** It's tempting to just rewrite URLs into `http://api.scraperapi.com/?api_key=...&url=...`, but ScraperAPI's own docs warn against it, with a concrete reason: when the page requests a relative resource like `/static/main.css`, that request gets sent to the API endpoint's domain instead of the target site, errors out, and your script only ever receives the first HTML response — which, in their words, "kinda defeats the purpose of using a headless browser." Clicking relative links to navigate fails the same way. Use the proxy port.

Full runnable examples (both proxy-port and endpoint variants) live in ScraperAPI's official GitHub repo under the Python Selenium folder, including one that scrapes quotes.toscrape.com end to end.

Because it's a managed service, you also get `render=true` for JS-heavy pages, `country_code` geotargeting, `premium=true`/`ultra_premium=true` for harder targets, and `session_number` to keep the same IP within a run — none of which you'd get from a raw proxy list without building the logic yourself. If that trade (some per-page cost for zero infrastructure) sounds right for your project, 👉 [the free trial lets you measure your real per-page cost before paying anything](https://www.scraperapi.com/?fp_ref=coupons).

## ScraperAPI plans and pricing

All plans bill by API credits per month (standard requests cost 1 credit; billing is per request, not by bandwidth). Current lineup:

| Plan | Price (monthly) | Credits / month | Concurrent threads | Billing cycle | Link |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | 1,000 | 5 | — | [Start free, no card needed](https://www.scraperapi.com/?fp_ref=coupons) |
| Student | discounted rate, shown on pricing page | — | — | monthly/annual | [See current student rate](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | $49 | 100,000 | 20 | monthly or annual | [Check Hobby plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | 1,000,000 | 50 | monthly or annual | [Check Startup plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | 3,000,000 | 100 | monthly or annual | [View Business plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 | 5,000,000 | 200 | monthly or annual | [View Scaling plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | — | — | monthly | [View Professional plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | — | — | monthly | [View Advanced plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | custom | custom | custom | custom | [Talk to sales](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Annual billing discounts the monthly rate — Hobby drops to **$44.10/month** billed annually, and third-party reviews put the annual discount at roughly 15–20% across tiers. The 7-day refund policy ("no questions asked," per the official docs) applies if the service disappoints, and you can cancel any time from the dashboard without a fee. Credits don't roll over month to month, so don't buy a bigger tier "for later" — unused credits are simply gone.

If you burn through a monthly plan's credits early, the monthly tiers offer an Auto-Upgrade to the next tier (which usually improves your price-per-credit), and the higher tiers support Pay-As-You-Go overage with an optional monthly spending cap — a limit, not a pre-charge.

## What a Selenium page actually costs through ScraperAPI

This is the part most reviews gloss over, and it matters a lot for Selenium users specifically. The sticker price assumes 1 credit per request, but your credits get multiplied depending on the request type:

- Standard requests from the default datacenter pool: **1 credit**
- `render=true` (JS rendering, which is exactly what most Selenium workflows need): roughly **10 credits** per request
- `premium`/`ultra_premium` pools for protected targets: **10–30 credits** per request
- Domain multipliers on top: Amazon requests cost **5×** credits, Google/Bing **25×**

Run the math on a real job and the effective per-page cost can be 20–30× the headline number. A third-party review's example: with JS rendering enabled on Amazon search pages, a Startup plan's 1,000,000 credits might realistically cover 40,000–100,000 actual pages. Budget against your real per-page cost, not the credit count on the pricing page.

The flip side: for standard e-commerce and search targets that aren't behind heavy anti-bot protection, the default pool clears most requests at 1 credit each, and the free plan plus trial give you a genuine way to measure your own per-page cost before paying anything.

## How it compares with the alternatives

For context, here's what the same job looks like on the services most often mentioned alongside ScraperAPI (pricing verified from a third-party comparison in August 2026):

| Service | Starting price | Free tier | Free trial |
| --- | --- | --- | --- |
| ScraperAPI | $49/month | Yes (1,000 credits) | Yes (7 days, 5,000 credits) |
| Apify | $29/month | Yes | No |
| Oxylabs | $30/month | No | Yes |
| Zyte | $100/month | No | Yes |
| Bright Data | $499/month | No | No |

The honest framing: if you need raw control over specific residential IPs or subnets, a proxy network like Bright Data or Oxylabs is built for that, and ScraperAPI deliberately doesn't compete there — you can't pick specific IPs or ASNs. Apify fits actor-marketplace workflows. ScraperAPI's niche is the middle ground: you want proxies, retries, and CAPTCHA handling handled for you, with a free path to test first.

## Where it doesn't fit

The documented limits are worth knowing before you buy:

- **Long-lived sessions.** `session_number` keeps the same IP for at most about an hour and is designed for single runs. Ad verification or account workflows that need to hold an identity across days need a dedicated residential or mobile provider instead.
- **Hard targets at volume.** Scraping LinkedIn or heavily protected sites at scale means most requests hit the 10–30× credit multipliers, which typically ends up pricier than a bandwidth-billed residential proxy plan.
- **Country-level targeting on cheap tiers.** Targeting outside the US and EU requires the Business plan ($299/month) or higher.
- **Support depth on entry tiers.** Hobby and Startup get email/ticket support only — if your scraper dies at 3 a.m., you're queueing.

## FAQ

**Does `user:pass@host:port` ever work in Selenium?**
Not in headless Chrome. Chrome blocks that URL form in headless mode, which is why the CDP auth handler, SeleniumBase, or IP whitelisting exist as workarounds. The one exception is when a local proxy layer like Selenium Wire performs the credential handshake on Chrome's behalf.

**Is Selenium Wire still a valid choice?**
It's still what ScraperAPI's official integration guide documents, but it no longer imports on modern Python. If your environment supports it, the proxy-port setup above is the simplest config; otherwise use SeleniumBase or CDP.

**Can I try ScraperAPI without paying?**
Yes — 1,000 free credits every month (5 concurrent connections) plus a 7-day trial with 5,000 credits, no credit card required. Paid plans carry a 7-day no-questions-asked refund.

**What happens if I run out of credits mid-month?**
Monthly plans can auto-upgrade to the next tier (usually better price-per-credit), and the higher tiers offer fixed-rate Pay-As-You-Go with a spending cap you control.

## Picking between the code route and the service route

If your Selenium scripts run against one or two unprotected targets and you're comfortable with a bit of Python, the CDP method costs you nothing but an afternoon, and SeleniumBase gets you there in one argument. If you're rotating proxies at any volume, hitting CAPTCHAs, or scraping JS-heavy pages where you'd otherwise need `render=true`-style handling anyway, the proxy-port route removes the auth problem and the rotation problem in one move — Hobby at $49/month covers 100,000 standard requests, and the trial tells you your real per-page cost before you spend a cent.

One last check before deciding: confirm your target domain's credit multiplier (Amazon and Google are the expensive ones), and make sure the Hobby plan's 20 concurrent threads actually match your script's parallelism. If both check out, 👉 [start with the 7-day trial and upgrade to Hobby only after you've measured your real usage](https://www.scraperapi.com/pricing/?fp_ref=coupons).
