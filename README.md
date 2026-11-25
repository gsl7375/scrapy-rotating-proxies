# Scrapy rotating proxies: how to set up scrapy-rotating-proxies, why concurrency runs per proxy, and when a proxy list stops paying for itself

Two paths here, and they don't mix. If your spider draws from a list of proxy IPs you own or rent, install `scrapy-rotating-proxies` (version 0.6, the last release with a changelog entry on PyPI, dated 2018-12-28) and add two middleware lines to `settings.py`. If you send traffic through one rotating endpoint or a proxy API, don't install that middleware at all. It interferes with the rotation the provider is already doing, and you'll spend an afternoon debugging a problem you built yourself.

Below is the full configuration for the list-based route, three things the quick-start posts usually skip (per-proxy concurrency, what a misdetected ban actually costs, and a setting people copy into the upstream package that only exists in a fork), and the point where maintaining your own pool costs more than it saves.

## Quick pick

> If your proxy list shrinks faster than you can replace it, running your own pool is a treadmill. ScraperAPI's free plan gives you 1,000 API credits and up to 5 concurrent connections, which is enough to test a real target before paying anything. A standard page costs 1 credit. Amazon costs 5, Google and Bing 25, LinkedIn 30, and sites behind Cloudflare, Datadome or PerimeterX add 10 per request when they get bypassed. 👉 [Start on the free plan and price out your target domain](https://www.scraperapi.com/?fp_ref=coupons)

## ScraperAPI plans, and what actually differs between them

Before the table, a caveat you should know about. The pricing page I checked lists the plan names, the free tier limits, the credit multipliers and the overage behaviour, but it does not publish a per-plan credit allowance or a price figure in the body content. Rather than paste numbers I can't stand behind, the table below covers only what the page confirms. Open the live pricing page for the current figure before you budget.

| Plan | Confirmed differences | Billing | Who it fits | Link |
| --- | --- | --- | --- | --- |
| Free | 1,000 credits, max 5 concurrent connections | Free | Testing one target end to end | [Claim the free credits](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | Auto-upgrade available when credits run out | Monthly or annual | Solo scraping, one or two sites | [See the Hobby plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | Auto-upgrade available | Monthly or annual | Small projects with steady volume | [See the Startup plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | Auto-upgrade available | Monthly or annual | Multi-site crawls, no overage handling | [See the Business plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | Auto-upgrade and pay-as-you-go | Monthly or annual | Bursty jobs that spike past the allowance | [See the Scaling plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | Auto-upgrade and pay-as-you-go | Monthly or annual | Teams that want a hard monthly cap | [See the Professional plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | Pay-as-you-go | Monthly or annual | High volume, cost controlled per credit | [See the Advanced plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | Pay-as-you-go | Custom | Committed annual volume | [Talk to sales about Enterprise](https://www.scraperapi.com/?fp_ref=coupons) |

Two mechanics matter more than the plan names. First, going over your concurrency limit returns a 429 and the request is turned away rather than queued, and you aren't charged extra for the attempt. Second, running out of credits pauses scraping by default. Auto-upgrade (Hobby through Professional) moves you up a tier automatically, and pay-as-you-go (Scaling through Enterprise) keeps you on your current plan at a fixed per-credit rate up to a monthly cap you set. Both are opt-in. If neither is on, a spider that hits its ceiling at 2am sits there until the credits renew.

The credit multiplier is the number to plan around, because it decides whether an allowance lasts a week or a month. One thousand credits is 1,000 ordinary pages, 200 Amazon pages, or 40 Google result pages. Sites behind bot protection are the expensive case: 10 extra credits per request on top of the base cost.

## Setting up scrapy-rotating-proxies in ten minutes

Install it:

bash
pip install scrapy-rotating-proxies


Then give it a proxy list in `settings.py`, either inline:

python
ROTATING_PROXY_LIST = [
    'proxy1.com:8000',
    'proxy2.com:8031',
]


or as a file path, one proxy per line:

python
ROTATING_PROXY_LIST_PATH = '/my/path/proxies.txt'


If both are present, `ROTATING_PROXY_LIST_PATH` wins. That's worth remembering when you edit the inline list and nothing changes.

Enable both middlewares with their fixed priorities:

python
DOWNLOADER_MIDDLEWARES = {
    'rotating_proxies.middlewares.RotatingProxyMiddleware': 610,
    'rotating_proxies.middlewares.BanDetectionMiddleware': 620,
}


The order isn't cosmetic. 610 puts rotation ahead of ban detection at 620, so a request that fails gets retried through a different proxy rather than the same dead one.

One exception trips people up. Any request with `proxy` already set in its `meta` bypasses the middleware entirely. To force a single request direct, set `request.meta['proxy'] = None`. To pin one request to one specific proxy, set `request.meta['proxy'] = "<my-proxy-address>"`. Useful for hitting a login endpoint through a sticky IP while the rest of the crawl rotates.

## Your concurrency setting is per proxy, not per spider

This is the part almost nobody mentions, and it explains a lot of "why is my crawl so slow" and "why did my proxies all get banned at once."

Once `RotatingProxyMiddleware` is active, Scrapy's normal concurrency options become per-proxy for proxied requests. `DOWNLOAD_DELAY`, the `AUTOTHROTTLE_*` family, `CONCURRENT_REQUESTS_PER_DOMAIN`, all of them. Set `CONCURRENT_REQUESTS_PER_DOMAIN = 2` and the spider opens at most two connections **to each proxy**, regardless of which domain the request targets.

Do the arithmetic before you tune. With 2 and a pool of 50 proxies, you've authorised up to 100 simultaneous requests, all against the same target. The target sees 100 concurrent hits from 50 IPs. If your pool is 200 proxies, the ceiling is 400. Whatever anti-bot system is watching sees a burst that looks nothing like human traffic.

The practical fix is to size the concurrency number against your pool, not against the target's capacity. If you want roughly 10 concurrent requests in total across a 40-proxy pool, set the concurrency to 1 and keep the pool small, or accept the multiplication and lower `DOWNLOAD_DELAY` is not the answer here (it's also per proxy). The per-proxy behaviour is documented, not a bug, but it silently inverts the intent of the setting.

## Ban detection decides how fast you burn through proxies

Default behaviour is a simple heuristic: a response with a status code other than 200, an empty response body, or an exception means the proxy is dead. That's it.

It misfires constantly on real sites. A 404 page, a redirect chain that returns nothing useful, a 429 from rate limiting rather than an IP ban, all get logged as a dead proxy. The middleware pulls that proxy from rotation and schedules a recheck.

You can override the rules with `ROTATING_PROXY_BAN_POLICY`:

python
# settings.py
ROTATING_PROXY_BAN_POLICY = 'myproject.policy.MyBanPolicy'


The policy class needs `response_is_ban` and `exception_is_ban` methods that return `True` (banned), `False` (not banned) or `None` (unknown). Subclassing the default is the shortest route:

python
from rotating_proxies.policy import BanDetectionPolicy

class MyPolicy(BanDetectionPolicy):
    def response_is_ban(self, request, response):
        ban = super(MyPolicy, self).response_is_ban(request, response)
        ban = ban or b'captcha' in response.body
        return ban

    def exception_is_ban(self, request, exception):
        return None


That example treats an HTTP 200 with the word "captcha" in the body as a ban, and stops counting exceptions at all. You can also implement the two methods directly on the spider if you'd rather not create a policy module:

python
class MySpider(scrapy.Spider):
    def response_is_ban(self, request, response):
        return b'banned' in response.body

    def exception_is_ban(self, request, exception):
        return None


Getting these rules right matters because a failed request and a bad proxy need different responses. If the proxy is at fault, retrying through a different one is the correct move. If the page is at fault, retrying through five different proxies just burns five proxies.

Which is the number to keep in mind: every improperly detected ban costs you `ROTATING_PROXY_PAGE_RETRY_TIMES` alive proxies, default 5. A rule that misclassifies one in ten responses will chew through a small pool in an afternoon.

Dead proxies get rechecked on a randomized exponential backoff. The first check comes soon, then progressively later, capped by `ROTATING_PROXY_BACKOFF_CAP`. The settings list puts `ROTATING_PROXY_BACKOFF_BASE` at 300 seconds (5 minutes); the FAQ section of the same docs describes the initial delay as random between 0 and 5 minutes. Treat 300 as the configured base and expect the first recheck to land somewhere in that randomised window rather than exactly on it.

## Settings cheat sheet

| Setting | Default | Change it when |
| --- | --- | --- |
| `ROTATING_PROXY_LIST` | empty | Small pool you're happy to edit inline |
| `ROTATING_PROXY_LIST_PATH` | none | Pool changes often; overrides the list above |
| `ROTATING_PROXY_LOGSTATS_INTERVAL` | 30 seconds | Logs are too noisy or you need finer visibility into dead/good counts |
| `ROTATING_PROXY_CLOSE_SPIDER` | `False` | Set `True` if a crawl with zero live proxies is worthless and you'd rather it stop than hang |
| `ROTATING_PROXY_PAGE_RETRY_TIMES` | 5 | Your ban rules are loose and a wrong call is expensive; override per request with `request.meta['max_proxies_to_try']` |
| `ROTATING_PROXY_BACKOFF_BASE` | 300 seconds | Proxies recover quickly and 5 minutes is too long to wait |
| `ROTATING_PROXY_BACKOFF_CAP` | 3600 seconds | You want a faster ceiling on rechecks for short-lived crawls |
| `ROTATING_PROXY_BAN_POLICY` | `rotating_proxies.policy.BanDetectionPolicy` | The default heuristic misreads your target |

`ROTATING_PROXY_CLOSE_SPIDER` deserves a note. With the default `False`, when no proxies are alive the middleware rechecks every dead proxy rather than stopping, so the spider sits and waits. On a long-running job that's fine. On a batch job with a deadline, it looks like a hang.

## Priority proxy lists exist only in the fork

Search around and you'll find `ROTATING_PROXY_PRIORITY_LIST`, `ROTATING_PROXY_PRIORITY_LIST_PATH` and `ROTATING_PROXY_PRIORITY_WEIGHT` in tutorials. They are not in the upstream 0.6 package. Copy them into a vanilla install and nothing happens, or you get a config that silently ignores what you wrote. They belong to the `iamashks` fork.

What the fork adds is two pools with a weighted split. `ROTATING_PROXY_PRIORITY_LIST_PATH` takes precedence over `ROTATING_PROXY_PRIORITY_LIST`, same as the main list. `ROTATING_PROXY_PRIORITY_WEIGHT` defaults to 0, and the values read like this:

- `0`: 100% of requests use the normal list
- `0.5`: 50/50 between the normal list and the priority list
- `1`: 100% of requests use the priority list

The obvious use is free proxies plus paid ones, or a datacenter pool plus a Tor circuit, shifting the weight as the free pool degrades. If that's your setup, the fork is the reason to switch. If you're running a single pool, stick with upstream and its documented settings.

## The real cost of a free proxy list

Free proxy lists are the cheapest option and the hardest to keep working. The reason isn't only that the IPs are slow. Anti-bot vendors scrape the same public lists and block those IPs in bulk, so the pool you downloaded this morning is partly dead by the afternoon. Every hour you spend writing logic to detect dead IPs and source replacements is an hour not spent on parsing or data quality.

That's the honest trade. A list-based setup means you own three jobs: sourcing IPs, deciding what counts as a ban, and deciding when to drop an IP permanently. The middleware handles rotation and the recheck schedule. It does not find proxies for you, and it ships no built-in ban rules for any specific site. The documentation says so directly.

If you're at the point where the pool maintenance is the project, the alternatives are a single rotating endpoint or a proxy API. Both move IP selection, rotation and ban detection to the provider. The trade is money for time and, usually, a better success rate on protected targets.

## Wiring an endpoint or API into Scrapy instead

There are two integration shapes, and they behave differently enough that they need separate treatment.

**Single endpoint with username and password.** You point every request at one host and port, and the provider rotates behind it. A small downloader middleware does the work:

python
import base64

class MyProxyMiddleware:
    def __init__(self, user, password, endpoint, port):
        self.user = user
        self.password = password
        self.endpoint = endpoint
        self.port = port

    def process_request(self, request, spider):
        credentials = f'{self.user}:{self.password}'
        auth = 'Basic ' + base64.b64encode(credentials.encode()).decode()
        request.meta['proxy'] = f'http://{self.endpoint}:{self.port}'
        request.headers['Proxy-Authorization'] = auth


Give this middleware a lower priority number than Scrapy's built-in `HttpProxyMiddleware`, or the built-in one processes the request first and your proxy setting gets overwritten.

ScraperAPI's proxy port works this way. The connection string takes the form `http://scraperapi.render=true:YOUR_API_KEY@proxy-server.scraperapi.com:8001`, with request options encoded in the username field. And this is exactly the case where `scrapy-rotating-proxies` must stay out of `DOWNLOADER_MIDDLEWARES`. The endpoint is already rotating. Adding a rotating middleware on top means two systems making rotation decisions about one connection, and the docs for this class of provider warn that the middleware can break the endpoint's correct functioning.

**API endpoint that takes the target URL.** You rewrite each request so it goes to the provider with your target URL as a parameter, and the response comes back as the page HTML. In Scrapy that means changing what you yield:

python
from urllib.parse import urlencode

API_KEY = 'YOUR_API_KEY'

def get_proxy_url(url):
    payload = {'api_key': API_KEY, 'url': url}
    return 'https://api.scraperapi.com?' + urlencode(payload)

yield scrapy.Request(get_proxy_url(book_url), callback=self.parse_book_page)


Two Scrapy-specific gotchas from the official integration docs. Set `ROBOTSTXT_OBEY = False`, because Scrapy's robots.txt check goes to the target's robots file and interferes when requests are routed through the API. And drop `DOWNLOAD_DELAY` and `RANDOMIZE_DOWNLOAD_DELAY`, which only lower your throughput here and add nothing.

Request parameters are where the country targeting and rendering live: `render=true` for JavaScript, `country_code=us` for US proxies, `premium=true` for residential and mobile IPs, `session_number=123` to reuse a proxy across requests, `keep_headers=true` to send your own headers. Premium proxies cost 10 credits, or 25 when combined with JavaScript rendering. Rendering itself is available on every plan, but confirm the credit cost against your specific target with the Domain Cost Estimator in the dashboard before you scale a job, since a standard page at 1 credit and a protected page at 11 are very different budgets.

That's the decision point for most people still maintaining a list. If the pool is stable and the target is unprotected, the middleware costs nothing and works. If you're replacing dead IPs weekly or writing ban rules per site, the maintenance is the expense, and 👉 [the current ScraperAPI plans and credit allowances](https://www.scraperapi.com/?fp_ref=coupons) are the faster route to a working spider.

## Common questions

**I added the middleware but some requests still go out without a proxy.** Check `request.meta` for a `proxy` key. Any request carrying one bypasses `RotatingProxyMiddleware` completely. It's also the mechanism for opting a single request out with `proxy = None`.

**A proxy is alive but keeps getting marked dead.** The default heuristic treats any non-200 status, empty body or exception as a death. A site that returns 403 for a valid page or 200 with a soft block will produce false positives. Write a `BanDetectionPolicy` that matches your target's actual block signals, and return `None` for the cases you genuinely don't know rather than guessing `True`.

**What does raising `ROTATING_PROXY_PAGE_RETRY_TIMES` cost me?** Every misdetected ban consumes that many live proxies before the failure is reclassified as a page failure. Raising it to 10 makes each wrong call twice as expensive, and on a 20-proxy pool that's half your capacity gone to one bad rule.

**Can I put scrapy-rotating-proxies in front of a single rotating endpoint?** No. The endpoint handles selection and rotation server-side, and stacking a rotation middleware on top can interfere with how it functions. Use one or the other.

**Do I need to change anything for HTTPS targets?** Scrapy skips SSL verification by default, which is why the proxy-port examples work without extra flags. If you've enabled verification in your project, expect certificate errors against proxy hosts and handle them explicitly rather than turning verification off globally.

## Which route to take

Running a small pool you control, on one or two unprotected targets: use upstream `scrapy-rotating-proxies` with default settings, and spend your tuning time on the ban policy rather than the concurrency numbers. Write the policy before you scale the crawl, not after the pool empties.

Splitting traffic between free and paid pools, or between a datacenter list and Tor: the `iamashks` fork with priority lists is the reason to move, and `ROTATING_PROXY_PRIORITY_WEIGHT = 0.5` is the starting point for a 50/50 split.

Replacing dead IPs more than once a week: a single endpoint or a proxy API removes that job. ScraperAPI's free tier gives you 1,000 credits and 5 concurrent connections to test your actual target before spending anything.

Two things to confirm before you pay for any plan, here or elsewhere. Check what `render=true` costs in credits for your specific target using the dashboard's Domain Cost Estimator, because JavaScript rendering on a protected site can multiply the per-request cost well past 1. And check the concurrency cap on the tier you're buying, since that ceiling, not the credit allowance, is what limits a wide crawl.

👉 [Check the live ScraperAPI pricing and start with the free credits](https://www.scraperapi.com/?fp_ref=coupons)
