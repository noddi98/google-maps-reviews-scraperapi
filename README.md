# Scrape Google Maps Reviews: The Parameters You Need, the ScraperAPI Plan That Fits, and When a Dedicated Reviews API Is Cheaper

Three things decide whether a Google Maps reviews job works: a stable place identifier (`data_id` or `place_id`), the sort and pagination parameters that control which reviews you actually see, and a request layer that survives Google changing its markup or throttling your IPs. ScraperAPI sells the third piece. It is a general scraping and proxy API billed in credits, not a reviews endpoint, so you bring the parser and it handles delivery. That single distinction drives most of the decisions below.

The other thing to know before comparing plan names: ScraperAPI prices Google and Bing requests at **25 credits each**, against 1 credit for a standard page. Your reviews bill is set by that multiplier, not by which tier sounds right.

> **Short answer for most readers:** if you already have a scraper and just need Google to stop blocking it, start on the free tier (1,000 credits, 5 concurrent connections) and expect about **40 Google page loads** out of it at 25 credits each. A working reviews project usually lands on a mid tier, where the credit allowance covers a few thousand page loads a month. If you have no scraper and want review JSON back from one GET request, ScraperAPI is the wrong shape of tool and a dedicated reviews endpoint will save you days.
>
> 👉 [See what each ScraperAPI tier currently costs](https://scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan, and what each one changes for a reviews job

ScraperAPI's tiers scale by credit allowance, and upgrading moves you to a larger allowance at a better price per credit. Two features change how a long review crawl behaves when credits run out: auto-upgrade, available on Hobby through Professional, and pay-as-you-go, available on Scaling through Enterprise with a monthly cap you set. Neither is on by default, and knowing which tiers have them matters more than the sticker price.

Per-tier dollar amounts move with the billing cycle you pick and with current promotions, so they belong on the pricing page at the moment you buy, not in a table that goes stale.

| Plan | Credit allowance | When credits run out | Fit for review scraping | Price & signup |
| --- | --- | --- | --- | --- |
| **Free** | 1,000 credits, max 5 concurrent connections | Requests stop | Testing one place, roughly 40 Google page loads | [Check the free plan terms](https://scraperapi.com/pricing?fp_ref=coupons) |
| **Hobby** | Larger allowance than Free | Optional auto-upgrade to next tier | First real crawl, one city or a handful of places | [See the Hobby rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| **Startup** | Larger again | Auto-upgrade available | Weekly refresh of a few hundred places | [See the Startup rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| **Business** | Larger again | Auto-upgrade available | Daily monitoring across a chain | [See the Business rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| **Scaling** | Larger again | Auto-upgrade **and** pay-as-you-go with a monthly cap | First tier where a burst doesn't halt the crawl | [See the Scaling rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| **Professional** | Larger again | Auto-upgrade **and** pay-as-you-go | Continuous multi-market collection | [See the Professional rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| **Advanced** | Larger again | Pay-as-you-go only | High-volume backfills, no auto-upgrade safety net | [See the Advanced rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| **Enterprise** | Custom, highest allowance | Pay-as-you-go | Ongoing archives, contractual terms | [See the Enterprise terms](https://scraperapi.com/pricing?fp_ref=coupons) |

Two mechanics worth knowing before you pick a tier. Going over your limit returns a 429 asking you to bring concurrency back inside the plan's limit; the request is refused rather than queued, and **there is no extra charge for going over**. And annual plans can renew early for a fresh batch of credits, which is the cheapest way to recover from a mid-project overrun without changing tiers.

## What 1,000 reviews actually cost in credits

ScraperAPI bills per request, so the variable that decides your bill is how many reviews your parser gets out of each page load. Google Maps review panels typically load around ten reviews per expansion; check your own target place, because the number moves with how the panel renders.

| Reviews wanted | Page loads (at ~10 per load) | Credits at 25 per Google request |
| --- | --- | --- |
| 1,000 | 100 | 2,500 |
| 10,000 | 1,000 | 25,000 |
| 50,000 | 5,000 | 125,000 |

The free tier's 1,000 credits cover about 40 of those page loads, which is a single place and then some. Budget a buffer on top of each row: a retry is another 25 credits, and review panels that fail to expand are common on mobile-heavy listings. ScraperAPI's dashboard has a Domain Cost Estimator that shows the exact credit cost for any URL, and you can pass `max_cost` per scrape so one expensive request can't quietly eat a day's allowance.

For context on the multiplier: a standard page costs 1 credit, Amazon 5, LinkedIn 30, and bypassing bot protection on Cloudflare, Datadome or PerimeterX adds 10 credits per request. Google sits at the high end because ScraperAPI prices the hard targets accordingly.

👉 [Match a credit allowance to your review volume](https://scraperapi.com/pricing?fp_ref=coupons)

## The parsing work ScraperAPI doesn't do for you

ScraperAPI's own Google Maps walkthrough builds a headless-browser scraper in Python with Selenium, Selenium Wire and BeautifulSoup, then adds a ScraperAPI key so requests authenticate through its layer. What comes back is the page, not a clean review object. You write the extraction, the pagination loop and the de-duplication.

That's a fair trade if you're already comfortable with a browser driver. It's a bad trade if you were hoping to pass a place ID and receive JSON. For comparison, a dedicated reviews endpoint such as SerpApi's takes one GET request with `engine=google_maps_reviews` and a `data_id`, accepts `place_id` as a substitute, and returns a structured `reviews` array with rating, ISO dates, likes, reviewer profile fields, `extracted_snippet.original` and `extracted_snippet.translated`, and an optional owner `response` object. Whether a given field appears depends on what Google exposes at that moment, so treat optional fields as optional in your schema.

If you go the ScraperAPI route, the parameters below are the ones you'll be reconstructing by hand, and they're also the ones that decide whether your sample is any good:

- **Place identifier.** A Maps feature ID (`0x...:0x...`) or `place_id` (`ChIJ...`), not a search keyword. Search URLs return whatever Google ranks that day.
- **Sort order.** Most relevant, newest, highest rating, lowest rating. "Most relevant" is Google's default and is not chronological, which breaks naive incremental crawls.
- **Pagination.** Review panels load in batches and stop at different points per listing. Paginate until the panel stops expanding, not until a fixed count.
- **Language.** Review text arrives in the reviewer's language. Decide up front whether you store original, translated, or both.

## Limits that change your sample size and your budget

These are the constraints most tutorials skip, and each one has cost implications.

**Initial pages return fewer results than later ones.** On SerpApi's endpoint, the `num` parameter ranges from 1 to 20 and defaults to 10, but it can't be used on the initial page when `next_page_token`, `topic_id` and `query` are all unset, and that first page always returns 8 results. Plan your request count around that, not around `num`.

**Filters are mutually exclusive.** On the same endpoint, `topic_id` and `query` can't be combined, so "negative reviews mentioning parking" is two passes, not one.

**Hosted actors default low.** Apify's newer Google Maps reviews actor (automly, page updated June 2026) allows up to **50,000 reviews per place** but defaults to **100**, and two places at 100 gives you 200 reviews. Leave the default in place and you'll conclude the tool is broken. Its older sibling (compass, February 2022) recommends keeping runs at 1,000 reviews or fewer to control resource consumption, and warns that its default "All Reviews" origin pulls third-party sources alongside Google, which reduced native Google coverage after a Google-side change. That page is four years old, so re-check the current behaviour.

**Ranking skews the sample.** Apify's 2022 page notes that Google tends to push reviews containing text and photos toward the top, which is why those get captured first. If you sort by relevance and stop early, your "random sample" is really a sample of the most visible reviews.

**Owner responses and reviewer profiles are optional fields.** A `null` owner response means no public reply, not a failed scrape. Reviewer names, photos and profile links are personal data even when posted publicly, and may fall under GDPR; if you don't need them, drop those fields at collection rather than at analysis.

## ScraperAPI against the dedicated options

Prices and limits below are as published by each vendor on the date shown. Different page dates mean different pricing eras; don't blend them.

| Option | Shape | Verified limits | Pricing basis | Page date |
| --- | --- | --- | --- | --- |
| **ScraperAPI** | Proxy and rendering layer, credits | Google at 25 credits/request; free tier 1,000 credits, 5 concurrent | Per request, in credits | Current pricing page |
| **SerpApi Google Maps Reviews API** | Dedicated JSON endpoint | `data_id` or `place_id` required; `sort_by` four options; `num` 1–20, initial page fixed at 8; `topic_id` and `query` exclusive | Per search; current rate not checked here | Current docs |
| **Apify automly actor** | Hosted scraper, no code | 50,000 reviews/place, default 100; $5 free usage monthly covers ~10,000 reviews; no Google account, cookies or API key | Per review, cheaper on higher plans | 2026-06-05 |
| **Apify compass actor** | Hosted scraper, no code | Recommends ≤1,000 reviews per run; Reviews origin must be set to Google for full native coverage | Pay-per-event; $19/month Starter plan cited as covering 42,000+ reviews | 2022-02-17 |

## Which one fits your situation

**You have a scraper and Google is blocking it** → ScraperAPI. Start free, confirm the 25-credit charge on your actual URLs with the Domain Cost Estimator, then pick the tier whose allowance covers your page-load count with 20–30% headroom.

**You need 10,000+ reviews per month across many places and no scraper** → a dedicated reviews endpoint or hosted actor, because the engineering time you'd spend rebuilding pagination and parsing costs more than the difference in list price.

**You need owner responses, translations and reviewer metadata as separate fields** → check current output schemas first. SerpApi separates original and translated snippets; Apify automly returns `ownerResponseText`, `textTranslated`, `isLocalGuide` and `authorReviewsCount`, and writes `null` rather than omitting a field, which keeps your rows consistent. ScraperAPI gives you none of that out of the box.

**You need fifty reviews for a qualitative read** → don't buy anything. Open the listing and copy them. A credit-based API makes sense from a few hundred reviews upward.

**You need a full historical backfill of a large listing** → price it in requests before you commit anywhere. 125,000 credits is a different conversation from 2,500.

## Before you pay

Confirm four things on the pricing page, in this order: the credit allowance attached to the tier you're eyeing, whether your target URLs bill at the Google rate of 25, what happens at zero credits on that tier (429 and a stop, or pay-as-you-go if you're on Scaling or above), and whether auto-upgrade is something you want switched on. Then run one real place through before scaling up.

If the answer to "do I want to write a parser" is no, the credit maths above is the reason to stop reading and pick a dedicated endpoint instead.

👉 [Start on the free 1,000-credit plan and test one place](https://scraperapi.com/?fp_ref=coupons)
