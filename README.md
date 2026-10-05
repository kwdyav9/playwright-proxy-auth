# playwright python proxy: fix the silent auth failures, rotate per context, and stop burning paid bandwidth

Most "Playwright proxy not working" threads end the same way: the proxy was configured, the browser launched, nothing raised, and the request came back either from your own IP or with a 407. Playwright has an unusual proxy API, and it fails quietly in exactly the two places people test first.

This is the setup that works, what the parameters actually do, and how to stop a headless browser from eating proxy bandwidth three times faster than it needs to.

## What people actually mean by "playwright python proxy"

The keyword covers three different jobs, and the right configuration depends on which one you're doing:

1. **Routing a single browser through one fixed exit.** Testing how a page looks from a specific country, or getting through a geo-block. One proxy, one context, done.
2. **Rotating exits mid-crawl.** You need a different IP per page or per N pages because the target starts throttling after a few hundred requests from the same address.
3. **Long session stability.** A multi-step flow — login, paginate, submit — where the IP must stay put for the whole sequence or the site flags you.

Playwright handles all three, but it only gives you the plumbing. It has no rotation policy, no retry logic, and no idea what a "session" is beyond the browser context you create. That part comes from the proxy gateway.

## The proxy dict, and the credential trap that costs you an afternoon

Playwright takes proxy settings as a dictionary, not as a URL string. This is the difference that breaks everyone coming from `requests`:

python
# You would pass this to requests. Playwright does not reliably parse it.
proxy_url = "http://user:pass@gw.example.com:823"

# Playwright wants this instead
proxy = {
    "server": "http://gw.example.com:823",
    "username": "user",
    "password": "pass",
}


Credentials embedded in the `server` string get dropped, and the gateway answers with `407 Proxy Authentication Required`. Nothing raises in Python. The browser starts fine, the page loads, and you get a proxy error page or an unauthenticated response you have to inspect to notice. If your proxy worked in `requests` and stopped working in Playwright, this is almost always the reason.

### Global proxy versus per-context proxy

`chromium.launch(proxy=...)` applies one proxy to the entire browser process. Every context and every page inherits it.

`browser.new_context(proxy=...)` sets the exit per context. This is how you rotate: one context per session, each pointing at a different exit, with its own cookies and storage. Pages inside a context share that context's exit IP — so if you're parallelising by opening ten pages in one context, you have not rotated anything. You have ten pages on one IP.

The other documented field is `bypass`, a comma-separated list of hosts that skip the proxy. Useful when you want your own internal services reachable while everything else goes through the exit.

### SOCKS5 is the one that fails without telling you

Playwright's documentation lists HTTP and SOCKS proxies as supported transports, and then lists `username` and `password` as options "if HTTP proxy requires authentication." Those are two different lists. SOCKS5 credentials are the one case Playwright accepts and silently discards: you pass all four fields, the browser starts, and the authentication never happens.

There's a Playwright issue asking for SOCKS5 authentication support that has been open since late 2021. A request that old is not a feature that quietly works.

Three routes around it:

- **Use the HTTP/HTTPS endpoint instead.** This is the documented, supported path for authenticated proxies. If your provider offers HTTP, take it.
- **Run a local relay** that listens without auth and authenticates upstream on your behalf, then point Playwright at `socks5://127.0.0.1:<port>` with no credentials. One extra process to supervise.
- **Debug by exit IP, never by page load.** A silent fall-through to a direct connection looks identical to success until you check the address. `page.goto("https://api.ipify.org")` and compare against your own IP before you scrape anything real.

## A working setup against DataImpulse

DataImpulse runs a single gateway: `gw.dataimpulse.com` on port `823` for HTTP/HTTPS and `824` for SOCKS5. Authentication is username/password or IP whitelisting, and countries are selected by appending parameters to the username rather than by changing the port or host.

Here's the sync version with the proxy set at launch:

python
from playwright.sync_api import sync_playwright

PROXY = {
    "server": "http://gw.dataimpulse.com:823",
    "username": "your_login",
    "password": "your_password",
}

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True, proxy=PROXY)
    page = browser.new_page()
    page.goto("https://api.ipify.org", timeout=30000)
    print("exit IP:", page.text_content("body"))
    browser.close()


Check that output first. If it prints your own address, the proxy was dropped somewhere.

The per-context pattern is what you want for anything beyond a single page. Rotating residential gateways hand out a new exit on each new connection, so a fresh context per unit of work is effectively a fresh IP:

python
from playwright.sync_api import sync_playwright

BASE = {
    "server": "http://gw.dataimpulse.com:823",
    "username": "your_login__cr.us",
    "password": "your_password",
}

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    for url in target_urls:
        context = browser.new_context(proxy=BASE, locale="en-US")
        page = context.new_page()
        page.route("**/*", lambda r: r.abort()
                   if r.request.resource_type in {"image", "media", "font"}
                   else r.continue_())
        page.goto(url, wait_until="domcontentloaded")
        # extract here
        context.close()
    browser.close()


Notice the `page.route` line. It is not cosmetic — it's the single biggest lever on your proxy bill, and the next-but-one section explains why.

## Rotation, sticky sessions and the parameters that control them

DataImpulse accepts targeting and session parameters by appending them to the login with a double underscore, using dots and semicolons:


login__cr.de                      # Germany
login__cr.de,au                   # Germany or Australia
login__cr.au;sessid.abc123        # Australia, pinned to one IP for ~30 minutes


`cr` is the country filter. Country-level selection is included in the base price. Finer targeting — city, ZIP, ASN — is described by DataImpulse as a paid add-on on residential plans, and independent reviews report that it consumes roughly double the traffic, which lands in the same place cost-wise. If your job only needs the right country, don't pay the multiplier.

`sessid` is the sticky mechanism. Attaching the same string gives you the same IP for an average of about 30 minutes — useful for a login flow, a cart test, or a multi-page form. It is explicitly not a guarantee. Residential exits come from real devices, and when the device behind your IP goes offline, the gateway swaps you to another address mid-session. Playwright can't detect that; your scraper has to notice the response changed or the cookie stopped working.

Sticky sessions are also available as a persistent port range in the dashboard — sessions configurable from 1 minute up to 120 minutes, with 30 minutes as the realistic average. Reported failure modes are worth knowing: a session interval you configured but cannot be guaranteed, and automatic rotation when the underlying device drops.

The practical advice is to configure targeting and session type in the DataImpulse dashboard's proxy generator rather than hand-writing the username suffix. The dashboard produces the exact string and a matching cURL command, so you're not guessing whether it's `sessid` or something else in the current build.

## Headless rendering is expensive — here's the actual math

Running a full Chromium under Playwright downloads images, fonts, stylesheets, tracking scripts and analytics beacons. Fetching the same page with `httpx` downloads the HTML. On a per-gigabyte proxy plan, that difference is the bill.

Take a mid-size scrape: 1,000,000 pages at an average 500 KB of transferred HTML — roughly 500 GB. At $1/GB that's about $500, before rendering. Enable image and media loading in a headless browser and the per-page payload routinely multiplies. DataImpulse's own pricing guidance makes exactly this point: budget more bandwidth when the workload is Playwright or Puppeteer, because browser rendering pulls more data than raw HTML retrieval.

Four things that cut it, all cheap to implement:

- **Abort image, media and font requests** with `page.route`. Most scraping targets give you the data you need from the DOM without them.
- **Reuse contexts** where rotation isn't required. A new context means a new TLS handshake, a new cookie jar and no connection reuse. Rotating per request is the right call for block-prone targets and the wrong call for everything else.
- **Use `wait_until="domcontentloaded"`** instead of waiting for full `networkidle` when the content you need is already in the DOM. You stop downloading things you will never read.
- **Send the request through the proxy only when it needs the proxy.** Direct fetches cost nothing. Escalate to the paid exit when you actually get a 403 or a bot wall.

Also worth budgeting: success rate is a cost multiplier, not a footnote. A pool with a 99% success rate and one with a 60% success rate can be quoted at the same $/GB and be nowhere near the same price per usable page, because you pay for the failed requests too.

## Which DataImpulse tier to point Playwright at

DataImpulse splits its network into four products. Picking correctly is more consequential than picking a provider.

**Datacenter — $0.50/GB.** IPs from server ranges, cheapest and fastest. Runs browser automation well against targets with no serious anti-bot layer: internal tools, public records, documentation, unguarded e-commerce. The trade-off is that datacenter ASNs are the first thing a serious WAF checks, so keep the residential budget for the sites that actually need it.

**Residential — $1/GB.** The default for Playwright work. Consumer ISP addresses across 90M+ IPs in 195+ countries, rotating and sticky sessions, HTTP/HTTPS and SOCKS5. This is the tier that gets through search engines, marketplaces and social platforms, and it's the one to point at any target that starts returning a challenge page.

**Mobile — $2/GB.** Real 4G/5G/3G/LTE carrier addresses. Buy this only when mobile network identity measurably changes the outcome — app-centric targets, mobile-first platforms, or sites that treat cellular ranges differently. It's twice the price of residential and usually unnecessary for general scraping.

**Premium Residential — $5/GB.** A cleaner, faster sub-pool with a dedicated account manager and all targeting options included. Justified for high-stakes, high-volume jobs where a 5% improvement in success rate pays for the 5× price difference. Overkill for a first Playwright project.

One honest limitation: this lineup is built around rotating pools and scheduled stickiness. If your workflow needs one fixed residential-looking IP for weeks — long-lived marketplace or ad accounts, antidetect browser profiles — a rotating residential plan is not a substitute, and DataImpulse doesn't market a standalone static ISP product as its primary offering.

## All DataImpulse plans

The minimum purchase is $5, there's no subscription, and purchased traffic does not expire. Prices below are the publicly listed tiers for each product.

| Proxy type | Plan | Traffic | Price | Effective $/GB | Billing | Get it |
| --- | --- | --- | --- | --- | --- | --- |
| Residential | Intro (new users) | 5 GB | $5 | $1.00 | Pay-as-you-go, no expiry | [Grab the 5 GB residential entry pack](https://bit.ly/dataimPulse) |
| Residential | Basic | 50 GB | $50 | $1.00 | Pay-as-you-go, no expiry | [Buy the 50 GB residential plan](https://bit.ly/dataimPulse) |
| Residential | Advanced | 1 TB | $800 | $0.80 | Pay-as-you-go, no expiry | [Order the 1 TB residential plan](https://bit.ly/dataimPulse) |
| Residential | Custom | 5 TB+ | From $4,000 | From $0.80 | Negotiated | [Request custom residential volume pricing](https://bit.ly/dataimPulse) |
| Datacenter | Intro (new users) | 10 GB | $5 | $0.50 | Pay-as-you-go, no expiry | [Start with 10 GB of datacenter traffic](https://bit.ly/dataimPulse) |
| Datacenter | Basic | 100 GB | $50 | $0.50 | Pay-as-you-go, no expiry | [Buy the 100 GB datacenter plan](https://bit.ly/dataimPulse) |
| Datacenter | Advanced | 1 TB | $450 | $0.45 | Pay-as-you-go, no expiry | [Order the 1 TB datacenter plan](https://bit.ly/dataimPulse) |
| Datacenter | Custom | 5 TB+ | From $2,250 | Negotiated | Negotiated | [Request datacenter volume pricing](https://bit.ly/dataimPulse) |
| Mobile | Intro (new users) | 2.5 GB | $5 | $2.00 | Pay-as-you-go, no expiry | [Test 2.5 GB of mobile traffic](https://bit.ly/dataimPulse) |
| Mobile | Basic | 25 GB | $50 | $2.00 | Pay-as-you-go, no expiry | [Buy the 25 GB mobile plan](https://bit.ly/dataimPulse) |
| Mobile | Advanced | 1 TB | $1,600 | $1.60 | Pay-as-you-go, no expiry | [Order the 1 TB mobile plan](https://bit.ly/dataimPulse) |
| Mobile | Custom | 5 TB+ | From $8,000 | Negotiated | Negotiated | [Request mobile volume pricing](https://bit.ly/dataimPulse) |
| Premium Residential | Intro (new users) | 1 GB | $5 | $5.00 | Pay-as-you-go, no expiry | [Try 1 GB of premium residential](https://bit.ly/dataimPulse) |
| Premium Residential | Basic | 10 GB | $50 | $5.00 | Pay-as-you-go, no expiry | [Buy the 10 GB premium residential plan](https://bit.ly/dataimPulse) |
| Premium Residential | Advanced | 1 TB+ | From $4,000 | From $4.00 | Negotiated | [Order premium residential at scale](https://bit.ly/dataimPulse) |

Payment methods include cards via Stripe, PayPal, wire transfer, Alipay, Apple Pay and Google Pay, plus crypto through Cryptomus (BTC, ETH, USDT, LTC). Intro plans carry a 7-day money-back guarantee on card payments provided less than 80% of the traffic is used; crypto purchases on Intro plans are not refundable.

## Troubleshooting: what the status codes actually mean

| What you see | What it usually means | Fix |
| --- | --- | --- |
| `407 Proxy Authentication Required` | Credentials in the `server` URL, or wrong login/password | Split `username`/`password` into their own dict keys |
| `407` with correct credentials | Traffic balance is exhausted | Check the dashboard usage panel |
| `407` under heavy concurrency | Too many simultaneous connections on the account | Lower parallel contexts, then retry |
| `503` / no IP returned | No exit matched your targeting | Drop city-level targeting, keep the country |
| `403` | The destination blocked the exit | Try a different country or a fresh sticky session once |
| `429` | The destination is rate-limiting you | Back off; don't retry in a tight loop |
| Everything "works" but data is wrong | Silent direct connection | Print the exit IP from `api.ipify.org` |

Two rules that save money. Retry with a changed variable, not blind — a failed request through a paid proxy is a failed request you already paid for. And never put a residential proxy behind a loop that hammers a site it can't get through; you'll finish the account balance and still have no data.

## Straight answers to the questions that show up in search

**Do I need to buy before I can test?** There's no free tier. The $5 entry pack is the smallest commitment across the four product types — 5 GB residential, 10 GB datacenter, 2.5 GB mobile, or 1 GB premium, depending on which you pick. That's enough to build the integration and measure success rate on your actual targets.

**Does the traffic expire at the end of the month?** No. Credits stay in the account until consumed, which matters for Playwright work because crawl volumes are lumpy. Monthly plans that reset are quietly more expensive for anyone whose usage comes in bursts.

**Rotating or sticky — which do I set?** Rotating for anything page-by-page. Sticky for anything with a state that has to survive more than one request. You can change it per context, so a single job can use both: sticky while you're logged in, rotating for the bulk crawl afterwards.

**Will randomising the user agent and adding stealth plugins fix my blocks?** Sometimes, and less often than the tutorials imply. A clean residential exit with a plausible fingerprint does most of the work. Both the fingerprint and the ASN reputation are scored before your JavaScript even runs, so a stealth patch on a blocked datacenter IP still gets challenged.

**Does the proxy hide me from the target?** It separates your infrastructure IP from the destination and lets you pick the location the site sees. It doesn't make an authenticated scrape anonymous, and it doesn't make an over-aggressive crawl acceptable. Rate, terms and data rules still apply.

## Bottom line

The two things that break Playwright proxy setups are the credential format and the silent SOCKS5 drop. Fix the dict, keep the auth flow on HTTP/HTTPS, and verify the exit IP before you scrape a single real page.

After that, the cost question is mostly about two settings you control: how much of each page you let the browser download, and which tier you point it at. Blocking images and media and defaulting to datacenter with a residential escalation is often the difference between a $500 crawl and a $150 one.

If you want to test it against your own targets before committing to volume, 👉 [start with the $5 / 5 GB residential pack](https://bit.ly/dataimPulse) — non-expiring traffic means an afternoon of integration work and a few gigabytes of headroom, and you can 👉 [compare the four proxy tiers](https://bit.ly/dataimPulse) on the same account before you scale anything.
