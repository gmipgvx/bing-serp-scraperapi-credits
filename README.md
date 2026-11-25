# Scrape Bing Search Results: Why Page 1 Returns 10 Results, Where Pagination Stops, and What Each SERP Costs in ScraperAPI Credits (Selectors + Full Plan Table)

Bing is the easy engine to parse and the frustrating one to scale. The markup is clean enough that a short script pulls titles and links on the first try, then the same script returns an empty page somewhere around result 40 and you spend an afternoon deciding whether the culprit is your proxy, your user agent, or Bing quietly deciding you've had enough.

Two numbers explain most of that afternoon. Bing often returns 10 organic results on page one no matter what count you request, and the total you can pull from a single query hits a ceiling in the low thousands. On the billing side, ScraperAPI charges 25 credits per Bing SERP request, so the free plan's 1,000 credits buy 40 Bing searches, not 1,000.

What follows is the DOM structure that still works, the failure modes that make scrapers go silent, the parameters that change what Bing hands back, and the plan maths you need before you pick a tier.

## Quick pick before the details

> **Most readers should start on the 7-day trial:** 5,000 credits, no credit card, which works out to roughly **200 Bing SERP calls** at 25 credits each. Once you know your real monthly volume, multiply your SERP count by 25 and pick the tier whose credit allowance clears it. The Scaling plan's 5,000,000 credits cover **200,000 Bing searches for $475/month**, or $427.50/month billed annually.

👉 [Check the current ScraperAPI tiers and credit allowances](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## ScraperAPI plans, credits, and how many Bing SERPs each one buys

The pricing page publishes more tiers than the two below. Hobby, Startup, Business and Professional sit alongside these with their own credit allowances and concurrency limits, and Enterprise is quoted through sales. Scaling and Advanced are the two with fully published figures, so those are the ones tabulated here:

| Plan | Credits per month | Concurrency | Price | Bing SERP calls/month | Buy |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 5 threads | $0 | 40 | [ Free plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Trial (7 days) | 5,000 total | — | $0, no card required | 200 | [ Start the trial](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 threads | $475/mo, or $427.50/mo billed annually ($5,130/yr, saves $570) | 200,000 | [ Scaling plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | 21,500,000 (plus 500K bonus credits while the offer runs) | 500 threads | $1,975/mo, or $1,777.50/mo billed annually ($21,330/yr, saves $2,370) | 860,000 | [ Advanced plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

The "Bing SERP calls" column is arithmetic, not a marketing figure: credits divided by 25. It is the column that decides your tier. A team monitoring 2,000 keywords weekly across three markets needs 6,000 SERPs a week, about 26,000 a month, which is 650,000 credits. Scaling covers it with room to spare. A side project tracking 150 keywords once a week needs 600 SERPs a month, or 15,000 credits.

## What each Bing request costs in credits

Bing and Google are priced as SERP requests rather than standard pages, and the difference is large enough to plan around.

| Request type | Credits |
| --- | --- |
| Standard page | 1 |
| Bing or Google SERP (including subdomains) | 25 |
| Amazon, Walmart, eBay | 5 |
| LinkedIn | 30 |
| JS rendering (`render=true`) | 10 |
| Premium proxies (`premium=true`) | 10 |
| Premium proxies plus JS rendering | 25 |
| Ultra premium (`ultra_premium=true`) | 30 |
| Ultra premium plus JS rendering | 75 |
| Bot-protection bypass (Cloudflare, DataDome, PerimeterX/HUMAN, Akamai) | 10 |
| Geotargeting, sessions, device type, output format, auto parsing | 0 |

Geotargeting costing nothing extra matters for Bing work, because pulling results for several countries is the standard way to get around the per-query ceiling. You pay 25 credits per search, not 25 plus a geo premium.

Two tools keep the estimate honest. Every response carries an `sa-credit-cost` header showing what that request actually consumed, and the dashboard's Domain Cost Estimator prices any URL before you run it. You can also set `max_cost` per scrape so a single request cannot run past a ceiling you define.

👉 [See how far your monthly credit allowance goes on Bing](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## What a Bing results page actually contains

Before any of the blocking discussion makes sense, you need the current selectors. Bing's organic results live inside `li.b_algo` containers, and the fields most projects need hang off that node:

- **Title and URL:** the `h2` inside the container, with the link as the `a` inside that `h2`
- **Snippet:** the paragraph inside the result's `div`, normalized to collapse whitespace
- **Date:** `span.news_dt`, formatted as `dd-mm-yyyy` for news-style results
- **Source site name:** `div.tptt`
- **Domain:** the URL with `https://` and a leading `www.` stripped
- **Related searches:** list items under `li.b_ans div ul li`

A minimal parse looks like this, and it is short enough to keep in one file:

python
for li in soup.select("li.b_algo"):
    link = li.select_one("h2 a")
    title = link.get_text(" ", strip=True)
    url = link["href"]
    snippet_el = li.select_one("div p")
    date_el = li.select_one("span.news_dt")


That is the whole parsing job. Everything else in this article is about getting pages that contain real results.

One date detail bites people: `span.news_dt` sometimes holds a long string rather than a clean date, so the reliable approach is to test whether the extracted text is longer than expected and then pull the `dd-mm-yyyy` pattern out with a regex. Storing the raw string as a date will break your database later.

## Why your scraper goes quiet halfway through

This is where most self-built Bing scrapers stop being useful. Bing has no strict CAPTCHA wall like Google's, but it does identify automated traffic and responds in ways that look like your code is broken.

### The three soft-block symptoms

Apify's documentation on its own Bing scraper lists them plainly, and they match what you see in logs:

1. **Ignored count.** You request 50 results per page and get 10. On page one, Bing may disregard the count parameter entirely.
2. **Empty pages.** The page renders as having no results while the query demonstrably has thousands.
3. **Unrelated results.** Soft blocking sometimes returns results that have nothing to do with your search terms.

Symptom three is the dangerous one, because a naive pipeline treats those rows as valid data and writes them to your keyword-tracking table. A cheap guard: check whether any of your query terms appear in the titles or snippets of the returned page, and reject the batch if none do. Retrying page one with different extra URL parameters usually produces a clean page; the production pattern is to fire several concurrent variants of the first page and cancel the rest as soon as one comes back clean.

### The Linux user agent trap

Bing serves different markup depending on the browser you claim to be. A Linux browser user agent gets a lighter page that omits the related searches block entirely. If your keyword-research pipeline reads that block and gets nothing back, the likely cause is the user agent string, not a missing feature.

The fix is to present as a Windows browser and render JavaScript. Setting the OS to Windows 11 with JS rendering enabled is what gets the related-searches list to appear. A short generic user agent like `Mozilla/5.0` is also more likely to be treated as a bot than a full Chrome string with platform details.

Not every query has a related searches block even on a fully rendered Windows page, so log a warning and return an empty list rather than crashing.

### The pagination ceiling

Bing shows millions of results for almost any query and delivers a small fraction of them. Beyond a certain depth it stops producing new URLs. Apify's scraper page puts the practical maximum around 2,000 results per keyword, which corresponds to page 200 at 10 results per page; the same documentation's FAQ states more conservatively that Bing never displays more than a few thousand results for one query. Different tools report different ceilings, and the underlying behaviour is consistent: deep pagination runs dry.

The practical consequence is that "scrape more results" is the wrong goal. If you need coverage, split the work across many near-duplicate queries and multiple markets and locations, rather than pushing `first` into the hundreds.

👉 [Put your query count against a real credit allowance](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## `first` is an offset, not a page number

The pagination parameter on Bing is `first`, and it does not mean "page N". It means "start the organic list at result number N". `first=1` is the top of page one, and `first=10` moves what would have been the tenth organic result into the first position.

Three rules follow:

- Do not compute page two as `first=2`. Advance by your expected page size, then verify.
- Do not assume every page holds a fixed count. If Bing hands you 10 when you asked for 50, your offsets drift and you silently skip results.
- Stop when a page returns no organic entries, and set a hard page or result limit in the loop so a soft block cannot turn into an infinite retry.

For a first pass, stepping `first` by 10 across the pages you need and running those requests concurrently is fine. Just treat the step as an estimate to be checked against what actually came back.

## Market, language, and filters: the parameters that change results

Bing's own query string carries most of the controls you need, and they behave the way Bing's URL behaves, independent of which tool sends the request.

**Market and language.** `mkt=en-US` sets market and language together; `setLang=en` pins the interface language; `cc=us` sets the two-letter country. Specify the market explicitly rather than letting the proxy's location decide, because an unset market means Bing infers one from the IP, and your results shift when your proxy pool rotates. Pick market and language as a pair and keep them stable across a monitoring run, otherwise your rank positions move for reasons that have nothing to do with your client's SEO.

**Safe search.** Three levels: `Off` returns adult text, images and video; `Moderate` keeps adult text but drops adult images and video; `Strict` drops all three. If you are building a brand-safety or SERP-comparison dataset, record which level produced each row.

**Filters.** Bing accepts complex filter expressions, including date ranges built from internal identifiers. Those identifiers are not documented in a way you can construct reliably. Copy the `filters` value out of a real Bing search URL after applying the filter in the interface, then reuse that exact string. Hand-writing an identifier produces either an error or, worse, a silently unfiltered page.

**Operators.** Standard Bing syntax works inside `q`: `site:`, `filetype:`, `NOT`, `OR`. Combining `site:` with a market is a cheap way to multiply coverage without touching pagination depth.

## Build it yourself or pay for a scraping API?

The decision comes down to four questions, and the answers point in clear directions.

**How many SERPs per month?** Under a few hundred, with one market and one language, a self-built scraper with retry logic is a reasonable weekend project. The parsing is 20 lines; the retries are the work. Past a few thousand requests a month, the retry logic, proxy pool and failure monitoring become a system you maintain, and the cost of that maintenance usually exceeds a subscription.

**Do you need more than one market?** Geotargeting multiplies your request count, because each market is a separate search. Multi-market monitoring on a home-grown proxy setup means sourcing residential IPs in each country.

**Do you need structured fields or raw HTML?** If downstream code expects JSON with positions, snippets and related queries, you either write and maintain that parser or buy it. If you only need the link list as a starting point for a crawler, raw HTML is fine.

**How fresh do results need to be?** This one catches people who switch tools. SerpApi, for example, serves a one-hour cache and does not count cached searches against the monthly quota, so a request for fresh results needs an explicit cache bypass, and that bypass cannot be combined with its async mode. If your rank tracker runs hourly and you need each run to reflect the live SERP, read the caching rules of whichever provider you pick before you commit.

For reference, here is how the options differ on the points that matter, using only what each vendor publishes:

| Option | Bing request model | Verified detail worth knowing | Public pricing |
| --- | --- | --- | --- |
| ScraperAPI | 25 credits per Bing SERP, geotargeting included at no extra credit cost | Soft blocks surface as empty or unrelated pages, so validate responses, not just status codes | Published per tier, with credit rates per request type |
| SerpApi | Structured Bing engine with `mkt`/`cc`/`location`, `first` offset, `output` of JSON, Markdown or HTML | `mkt` and `cc` are mutually exclusive, `location` and `uule` are mutually exclusive, cache is one hour and cached hits are free | Published, tabulated separately |
| Apify | Hosted Bing Search Scraper actor, accepts a keyword or a full Bing URL (only the query is taken from the URL) | `resultsPerPage` accepts 10 to 50, but Bing may return only 10 on page one; soft-block detection retries automatically | No fixed unit price; the vendor recommends a small test run to derive cost per scrape |

SerpApi's response shape is worth knowing even if you never use it, because it documents what a Bing page contains: `organic_results` with position, title, link and snippet; `related_searches`; plus ads, inline videos and knowledge panels that appear only for some queries. Its documentation is explicit that not every response includes every section, which is true of the underlying page as well.

## A short note on legality

Scraping publicly available search results is generally treated as permissible, subject to local and regional rules on personal data and to Bing's terms of use. Two habits keep you on the right side of it: keep request rates reasonable, and only collect personal data when you have a specific reason to. If your use case sits near that line, get legal advice rather than an article's opinion.

## FAQ

**How many results can I get from one Bing query?**
A few thousand at most. Bing advertises millions and stops serving new URLs well before that. Published figures differ between tools, which is why the useful planning number is your query count, not your page depth. To get 50,000 results, plan roughly 25 variations across markets rather than 5,000 pages.

**Why do I only get 10 results on page one?**
Bing may ignore a count parameter above its default and return 10 regardless of what you request. Raising the per-page setting does not reliably fix it. Pull more pages, or accept 10 and vary the query.

**Do I need a Bing API key to scrape search results?**
No. You are fetching public `bing.com/search` pages, so no key is involved. What you do need is IP rotation, a working parser and soft-block detection.

**Is the free plan enough to test with?**
1,000 credits is 40 Bing SERPs, which is enough to confirm your parser works and see the response shape. It is not enough to judge whether a monitoring schedule is viable. The 7-day trial's 5,000 credits give you 200 SERPs, which covers a small keyword set end to end.

**Does geotargeting cost extra?**
Not on ScraperAPI. Geotargeting, sessions, device type and output format are listed at no credit cost; the 25-credit SERP rate is what applies.

**What happens when I hit my concurrency limit?**
You get a 429 response telling you to reduce concurrency and retry. There is no overage charge; the request is refused rather than queued. When credits run out, scraping pauses until renewal unless you have auto-upgrade or pay-as-you-go enabled on an eligible plan.

## Which plan to buy

Match the tier to your monthly SERP count, calculated at 25 credits per Bing request:

- **Validating a parser or a new data source:** the 7-day trial, 5,000 credits, 200 Bing SERPs, no card.
- **Tracking a few dozen to a few hundred keywords in one market:** the smallest paid tier that clears your monthly count. Work out the number first, then pick, because a 1,000-credit allowance is only 40 Bing searches and will not survive a real keyword list.
- **Multi-market or multi-language monitoring:** Scaling, at 5,000,000 credits and 200 concurrent threads, gives 200,000 Bing searches and free country-level geotargeting.
- **Continuous multi-source pipelines:** Advanced, at 21,500,000 credits, 500 threads and priority routing, gives 860,000 Bing searches.
- **Structured JSON fields rather than HTML:** confirm the response format before you buy anything, from any vendor. A SERP endpoint that returns raw HTML still leaves you maintaining a parser.

Two things to confirm on the page before you enter card details: the refund terms, which the pricing FAQ addresses but which you should read in full for your billing period, and what your account does at credit exhaustion, since auto-upgrade is available on Hobby, Startup, Business, Scaling and Professional while pay-as-you-go is limited to Scaling, Professional, Advanced and Enterprise. Getting that wrong means a monitoring job that stops mid-month without an alert.

👉 [Start the 7-day trial with 5,000 credits and price out your Bing volume](https://www.scraperapi.com/pricing/?fp_ref=coupons)
