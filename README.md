# How to Scrape eBay Listings Without Getting Blocked: Python Code, Pricing Breakdown, and the Tool That Handles CAPTCHAs For You

I tried to pull eBay search results with a plain `requests.get()` call last year. Got a 503 on the eleventh request. Then a CAPTCHA. Then nothing — just an empty page where the listings used to be. If you've landed here searching for how to scrape eBay listings, you've probably already hit some version of that wall.

eBay doesn't block you for fun. It's actively watching for repeated requests, missing headers, and IP addresses that look like a data center instead of a browser. Scrape it directly at any real volume and you'll get rate-limited or fed garbled HTML within minutes. That's the actual problem this post is solving — not "how do I write a for-loop," but "how do I keep getting clean data after request #50."

## What "Scraping eBay Listings" Actually Involves

Scraping eBay listings means programmatically pulling structured data — titles, prices, condition, seller ratings, shipping cost, bid counts — from eBay's search results or product pages, instead of copying it by hand. The catch is that eBay renders a lot of that data with JavaScript and rotates its page structure often enough that a hardcoded parser breaks every few months.

There are basically three ways to do this:

1. **Raw requests + BeautifulSoup**, free but fragile, and you're on your own for proxies and CAPTCHA handling
2. **Headless browser (Selenium/Playwright)**, more reliable for JS-heavy pages, but slow and resource-heavy at scale
3. **A scraping API**, where someone else's infrastructure handles proxies, JS rendering, and retries, and you just get the data back

Which one makes sense depends entirely on volume. Pulling 50 listings once? Raw Python is fine. Tracking 10,000 SKUs daily for repricing? You need the third option, full stop.

## Step 1: Build a Basic Scraper (For Small, One-Off Pulls)

If you just need a handful of listings and don't mind babysitting the script, here's a working starting point using Python and BeautifulSoup:

python
import requests
import json
from bs4 import BeautifulSoup

url = "https://www.ebay.com/sch/i.html?_nkw=airpods+pro&_sacat=0"
headers = {"User-Agent": "Mozilla/5.0"}

r = requests.get(url, headers=headers)
soup = BeautifulSoup(r.text, "lxml")

listings = soup.find_all("div", class_="s-item__info clearfix")
results = []

for item in listings:
    title = item.find("div", class_="s-item__title")
    price = item.find("span", class_="s-item__price")
    if title and price:
        results.append({
            "title": title.text.strip(),
            "price": price.text.strip()
        })

print(json.dumps(results, indent=2))


This works for maybe the first 20-30 requests. Then eBay notices the pattern — same IP, no JS execution, identical headers every time — and starts serving CAPTCHAs instead of listings. At that point you're choosing between building your own proxy rotation system or outsourcing the headache entirely.

**Quick summary:** raw scraping is free and fine for a one-time pull under a hundred items, but breaks down fast under repeated or scheduled requests because eBay actively detects bot patterns.

## Step 2: When You Need to Scale, Skip the Proxy Maintenance

This is where ScraperAPI comes in — it's built specifically to absorb the part of scraping nobody enjoys: rotating proxies, solving CAPTCHAs, rendering JavaScript, and retrying failed requests automatically. You send a URL, it sends back clean HTML or parsed JSON.

The eBay-specific setup looks like this:

python
import requests

payload = {
    'api_key': 'YOUR_API_KEY',
    'url': 'https://www.ebay.com/sch/i.html?_nkw=airpods+pro&_sacat=0',
    'country': 'us',
    'output_format': 'markdown'
}

response = requests.get('https://api.scraperapi.com/', params=payload)
print(response.text)


That `output_format` parameter is worth pointing out — set it to `markdown` or `text` and you get LLM-ready output instead of raw HTML, which saves a parsing step if you're feeding the data into another model downstream.

A few things this handles that a DIY scraper doesn't:

- **JS rendering**, so listings that load dynamically actually show up in the response
- **Automatic retries**, so a single failed request doesn't kill your whole batch job
- **Geotargeting**, since eBay shows different prices and inventory depending on which country the request appears to come from — useful if you're comparing listings across `ebay.com`, `ebay.co.uk`, and `ebay.de`
- **A pool of residential and premium IPs**, so you're not burning through one IP until it gets flagged

👉 [Check ScraperAPI's current eBay scraping setup and free trial](https://www.scraperapi.com/?fp_ref=coupons)

## Pricing: What It Actually Costs to Scrape eBay at Scale

Here's the honest math. ScraperAPI runs on a credit system — every request to a standard page costs 1 credit, but JS-rendered or premium-proxy requests (which eBay search and product pages typically require) cost more. The plans themselves are flat monthly tiers, not pay-per-request, which makes budgeting predictable if you're running this as a recurring job.

| Plan | Monthly Price | Annual Price | API Credits | Concurrent Threads | Geotargeting |
|---|---|---|---|---|---|
| Hobby | $49/mo | $44.10/mo | 100,000 | 20 | US & EU only |
| Startup | $149/mo | $134.10/mo | 1,000,000 | 50 | US & EU only |
| Business | $299/mo | $269.10/mo | 3,000,000 | 100 | Global (country-level) |
| Scaling | $475/mo | $427.50/mo | 5,000,000 | 200 | Global (country-level) |
| Professional | $975/mo | $877.50/mo | 10,500,000 | 300 | Global (country-level) |
| Advanced | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | Global (country-level) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | Global (country-level) |

👉 [See full plan details and start the trial](https://www.scraperapi.com/pricing/?fp_ref=coupons)

For most people reading this — someone tracking a few hundred to a few thousand eBay listings for price monitoring or reselling research — **Hobby or Startup covers it**. The Business plan only really makes sense once you're running country-level geotargeting or pulling millions of records a month.

Worth doing the math yourself before committing: Startup at $149/month for a million credits works out to roughly $0.0015 per request if you use the full allocation. Compare that against the hours you'd spend building and maintaining your own proxy rotation, and for anything beyond a side project, it's not really close.

One thing that takes the pressure off testing this: there's a 7-day free trial with 5,000 API credits, no credit card required. That's enough to actually run a real eBay scraping job — not just a toy example — before deciding if it's worth paying for.

👉 [Start the 7-day free trial](https://www.scraperapi.com/signup/?fp_ref=coupons)

## What You Can Actually Pull From an eBay Listing

Once requests are getting through reliably, here's the data that's realistically extractable from eBay's public pages:

- Listing title, price, and currency
- Item condition (New, Used, Refurbished, etc.)
- Seller username and feedback percentage
- Shipping cost and estimated delivery
- Bid count and time remaining (for auctions)
- Image URLs
- Category breadcrumbs and item specifics (UPC, MPN, etc.)

That's plenty for price monitoring, competitor tracking, or building a deal-finding tool. What you won't get through scraping — because eBay doesn't expose it publicly — is buyer-side data like purchase history or private messages. Don't go looking for that; it's not there and trying to get it crosses into account-level access eBay actively shuts down.

## Common Mistakes That Get Scrapers Blocked

Quick list, since this comes up constantly:

1. **Sending identical request headers every time.** eBay fingerprints this fast.
2. **No delay between requests.** Even a basic rate limit (1-2 seconds) cuts block rates noticeably.
3. **Ignoring JavaScript-rendered content.** Some listing data only appears after JS executes — a raw HTML pull misses it.
4. **Reusing one IP for thousands of requests.** This is the single biggest reason scrapers get flagged.
5. **Not handling pagination correctly.** eBay search results paginate with specific URL parameters; guessing wrong gets you duplicate or empty pages.

A managed API sidesteps most of these automatically, which honestly is the whole pitch — you're paying to not think about it.

## ScraperAPI vs. Building It Yourself: Quick Comparison

| | DIY (Requests + Proxies) | ScraperAPI |
|---|---|---|
| Upfront cost | Free (your time) | From $49/month |
| Proxy management | Manual, ongoing | Handled automatically |
| CAPTCHA handling | You build it | Included |
| JS rendering | Requires headless browser setup | Built in |
| Maintenance when eBay changes layout | You fix it | Handled on their end |
| Best for | One-off pulls, learning | Recurring or large-scale jobs |

If this is a weekend project to grab 50 listings once, write the raw script and move on. If you're checking prices daily, tracking competitors, or feeding a pricing model, the math tips toward a managed API pretty quickly — the time saved alone usually covers the subscription cost.

## FAQ

**Is it legal to scrape eBay listings?**
Scraping publicly available data is generally treated differently from accessing private or login-gated information, but eBay's terms of service do restrict automated access. Most people scraping for personal price tracking or research stay in a gray area that's rarely enforced against individuals; high-volume commercial scraping is where eBay tends to push back. This isn't legal advice — if you're building something commercial at scale, it's worth having an actual lawyer look at your specific use case.

**Why does my scraper keep getting CAPTCHAs from eBay?**
Almost always a pattern problem — same IP making too many requests too fast, missing or generic headers, or no JavaScript execution where eBay expects it. Rotating IPs and adding realistic delays fixes most of this.

**Can I scrape eBay without coding?**
Yes — ScraperAPI's DataPipeline lets you submit a list of eBay URLs and schedule scraping jobs without writing code, useful if you want recurring price checks without maintaining a script.

**How many eBay listings can I scrape per day on the free trial?**
The trial includes 5,000 API credits over 7 days, no credit card required. Since eBay pages typically use premium/JS-rendered credits, that's enough for a few hundred to a couple thousand listings depending on page type — plenty to test whether it fits your workflow.

**Does ScraperAPI work for other eBay regions like ebay.co.uk or ebay.de?**
Yes, geotargeting is included on every plan starting from Hobby, though country-level targeting beyond US/EU requires Business tier or above.

If you're just starting out and want to confirm this actually solves your specific use case before paying anything, that's what the trial is for — run your real eBay URLs through it, see the data come back clean, then decide.

👉 [Get started with ScraperAPI's eBay scraper](https://www.scraperapi.com/?fp_ref=coupons)
