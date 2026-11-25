# Scrape eBay: Can You Still Get Sold Listings, What Do 1,000 Listings Cost, and Which ScraperAPI Plan Covers It? (Full Plan Table + Credit Math)

eBay's sold and completed listings are closed to signed-out visitors. Put `LH_Complete=1&LH_Sold=1` on a search URL and eBay redirects you to a sign-in page; in 18 test runs through Scrapfly's unblocker on a US IP in October 2026, half landed on "Sign in or Register" and half returned HTTP 403, with zero sold listings returned. Drop those two parameters and the same search serves a full results page. Everything else is still there: active listings, prices, shipping costs, seller feedback, item specifics, and the per-variant stock data eBay buries in a `MSKU` script tag.

What you can scrape is no longer the interesting question. What it costs is, and the answer depends almost entirely on whether your provider bills per request or per listing. Below is the full ScraperAPI plan table, the credit math for a 1,000-listing and a 10,000-listing job, and the fields you will lose if you guess at the parser.

> **Short answer for most readers:** the Startup plan covers a serious eBay monitoring job. 1,000,000 credits a month for **$149/month**, or **$134.10/month billed annually** ($1,609.20/year). At ScraperAPI's posted rate of 5 credits per eBay request plus 10 for the Akamai bypass, that is roughly 66,000 eBay page requests a month, or about 4 million listing records if you pull 60 results per search page and parse the HTML yourself.
> 👉 [See the Startup plan and current pricing](https://www.scraperapi.com/pricing?fp_ref=coupons)

## ScraperAPI plans compared

ScraperAPI publishes posted rates for the trial, Startup and Scaling tiers. Business, Professional, Advanced and Enterprise sit above them with larger allowances; their figures live on the pricing page rather than in a public table, so treat that row as a pointer, not a quote.

| Plan | Credits per month | What it handles on eBay | Price | Billing | Get it |
| --- | --- | --- | --- | --- | --- |
| Free trial | 5,000 total over 7 days | About 330 eBay requests at 15 credits each — enough to validate your parser and confirm the Akamai bypass works | $0, no credit card | 7-day trial; downgrade to the free tier when it ends | [Start the free trial](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | ~66,000 eBay requests/month; ~4M listing records via search HTML, or ~66,000 full product pages via the structured endpoint | **$149/mo**, or **$134.10/mo** annual ($1,609.20/yr) | Monthly or annual | [Check Startup pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling (most popular) | 5,000,000 | ~333,000 eBay requests/month; the tier where continuous category-wide monitoring stops being a budget question | **$475/mo**, or **$427.50/mo** annual ($5,130/yr) | Monthly or annual | [Check Scaling pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business, Professional, Advanced, Enterprise | Above 5M, increasing by tier | For multi-marketplace crawls or daily full-catalog refreshes; pay-as-you-go and custom enterprise rates apply at the top end | Posted on the pricing page | Monthly or annual | [See the higher tiers](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Two credit rules matter more than the headline allowance. eBay sits in the e-commerce bucket at **5 credits per request**, not the standard 1. Bot-protection bypass adds **10 credits per request** for Cloudflare, DataDome, PerimeterX and Akamai, and eBay runs on Akamai, so 15 credits per eBay page is the realistic planning number. Optional switches cost more: `render=true` adds 10, `premium=true` adds 10, and stacking premium proxies with JS rendering lands at 25. Geotargeting, sessions and output format are free. Every response carries its actual cost in the `sa-credit-cost` header, and the Domain Cost Estimator in the dashboard will price any URL before you commit to a job.

## What eBay still serves to a signed-out scraper

Search pages and item pages are both open, and the field coverage is wider than most tutorials suggest. A search result card gives you the title, the price, the shipping text, the "Located in" string, the subtitle, the thumbnail, and the rating plus rating count. The total result count sits in `.srp-controls__count-heading>span`, which is what you need before you can compute page depth.

On an item page, the reliable extraction points are:

| Field | Where it lives |
| --- | --- |
| Item ID | The 12-digit number after `/itm/` in the canonical link, e.g. `ebay.com/itm/406857460314` |
| Primary price | `.x-price-primary .ux-textspans` |
| Converted price | `.x-price-approx__price` (eBay auto-converts for some regions) |
| Title | `h1 span` |
| Seller | `[data-testid=x-sellercard-atf] a[href*="/str/"]` |
| Images | `.ux-image-filmstrip-carousel-item.image img` plus `.ux-image-carousel-item.image img` |
| Item specifics | `dt.ux-labels-values__labels` / `dd` pairs under `div.ux-layout-section--features` |
| Description | An iframe, `iframe#desc_ifr` — you get a URL, not text, and need a second fetch |

That last row is the one that quietly doubles the cost of any detail-page job. The description is a separate document, so a "complete" item record is two requests, not one.

**Multi-variant listings are a different problem entirely.** For phones, laptops and anything sold in model/storage/colour combinations, the per-variant price, stock level and image set are not in the HTML body at all. They sit in a JSON blob inside a `<script>` containing `MSKU`. Inside it, `selectMenus` gives you the option labels, `menuItemMap` maps each option to `matchingVariationIds`, and `variationsMap` holds the real data per variant: `binModel.price.value` with `convertedFromValue` and `convertedFromCurrency` for the original currency, `quantity.outOfStock`, `quantityInput.validations` where a `NumberValidation.maxValue` is the purchasable quantity, and `menuItemPictureIndexMap` for images. A parsed variant comes out looking like this:


{'id': '477101697616', 'Color': 'Deep Blue', 'price_original': 2199,
 'price_original_currency': 'USD', 'price_converted': 2199,
 'price_converted_currency': 'USD', 'out_of_stock': False,
 'quantity_available': 10, 'images': ['https://i.ebayimg.com/.../s-l1600.jpg']}


A CSS-selector parser returns one price for the whole listing and silently misses the other nine variants. If your use case is stock monitoring or resale arbitrage, this is the part you actually came for.

One more thing that changes your numbers without changing your code: the exit IP's country. The same reference implementation sets `country` to `"FR"` with the comment "change country for geo details like currency and shipping," and it is not decorative. Currency, shipping cost and which listings are visible at all shift with the IP's country. Any price you store needs the marketplace and the scrape timestamp attached, or it is not comparable to anything.

## What you cannot get anymore, and why no tool fixes it

Sold and completed listings are gone for signed-out visitors. `LH_Complete=1&LH_Sold=1` now routes to the sign-in page, and the October 2026 test series above found no configuration that changed that outcome, with or without JavaScript rendering. Apify's eBay actor documents the same behaviour from the other side: since 2025 eBay shows sold and completed results only to logged-in users, and that actor, which does not log in, silently falls back to active listings and says so in its run summary.

If a vendor promises historical sold data from eBay without an account, ask what it is actually returning.

eBay does have an official API, and it is worth knowing exactly what it gives you. The **Browse API** replaced the Finding API, which eBay retired on **4 February 2025**. Browse searches current listings by keyword, category, GTIN, product, compatibility criteria or image, and returns item and variation-group details. Anyone with a developer account can test it in the sandbox. Production access is intended for approved eBay partners and requires an application, an eligibility review and signed agreements, and approval is not guaranteed. Browse also does not serve historical sold listings, and it does not expose every field rendered on a live eBay page.

So the split is clean. Official API: current listings, gated production access, clean JSON. Scraping: current listings plus the rendered fields the API omits, no approval process, and you own the parsing. Historical sold data: neither, unless you log in.

👉 [Price out the Startup plan against your own request volume](https://www.scraperapi.com/pricing?fp_ref=coupons)

## What 1,000 eBay listings actually cost

This is where comparisons between scraping services go wrong, because the billing units are not the same thing. Apify's eBay scraper charges **$1.00 per 1,000 search results**, and **$2.00 more per 1,000** if you enable item details, so $3.00 per 1,000 listings with full detail. ScraperAPI charges per request, and a single eBay search page with `_ipg=60` returns 60 listings.

Run the numbers on 1,000 active listings:

| Route | Requests | Credits (at 15/request) | Cost on Startup | Cost on Scaling |
| --- | --- | --- | --- | --- |
| Search HTML, 60 listings per page | ~17 | ~255 | ~$0.04 | ~$0.02 |
| Structured eBay Product endpoint, one request per item | 1,000 | ~15,000 | ~$2.24 | ~$1.43 |

At 10,000 listings through search HTML, you need about 167 page requests and roughly 2,505 credits, which is under $0.40 on Startup.

The gap between the two rows is the whole argument. Search-level scraping is almost free per listing because one request carries 60 of them. The product-level route costs 60 times more because you are paying per item, and it is the route you need when you want descriptions, item specifics, seller feedback counts and returns policy without writing an extractor for each field yourself.

ScraperAPI's structured endpoint for eBay is `POST https://async.scraperapi.com/structured/ebay/product`. It takes a `productId` (the 12-digit item number) or a `productIds` array for batch jobs, plus optional `tld` (defaults to `com`) and `country_code`, and returns JSON with title, price and currency, seller name, seller review count and percentage, top-rated flag, images, available quantity, sold count, shipping cost and notes, return policy, condition, brand, model, colour, and the full item specifics array. A webhook callback is supported for async runs. `country_code` drives the language and currency of the returned page, which is the parameter that keeps your price data consistent across marketplaces.

What the endpoint does not have is a search equivalent. There is no structured eBay *search* API, so search-result collection still means requesting search URLs and parsing the cards, or accepting the per-item cost of feeding individual item IDs into the product endpoint. Plan the tier around that: search-heavy monitoring is cheap, detail-heavy enrichment is the line item that decides whether you need Startup or Scaling.

## Writing the scraper yourself vs. routing through an API

The extraction logic is genuinely not hard. The selectors above are stable enough that a working parser is an afternoon's work. What is hard is getting the HTML at all.

eBay sits behind Akamai. Datacenter IPs get blocked quickly, and the setup that holds up is residential proxies paired with a browser TLS fingerprint that looks real, plus session rotation and a retry when a page comes back blocked. That is a proxy bill, a fingerprint library, and a retry policy to maintain, and it is the same work whether you scrape 500 pages or 5 million.

Then come the parts nobody warns you about. Pagination runs on `_nkw` for the keyword, `_sacat` for category, `_sop` for sort order, `_pgn` for page number and `_ipg` for page size, and a concurrent fetch across pages needs exactly the rate limiting that keeps you off eBay's radar. Item pages need the iframe second fetch. Variant listings need the `MSKU` JSON parser. And a single search only exposes about 10,000 results, so any category-wide job has to be split by subcategory, price band or condition before you can collect it all.

A third-party Scrapy project that scrapes eBay reports roughly 100 search listings per minute with rate limiting and about 30 fully detailed products per minute, with over 95% field extraction success, on its own benchmarks. Those are that project's self-reported figures, not ScraperAPI's, but they are a fair order of magnitude for what hand-rolled infrastructure delivers before you add proxy costs.

A managed API collapses that stack into a request. You send a URL, you get HTML or JSON back, and the Akamai bypass, proxy pool and retry logic are someone else's problem. Whether that trade is worth it depends on your volume: at 500 pages a month, building is a hobby project; at 300,000, the maintenance is a job.

## FAQ

**How do I get past the 10,000-result cap on an eBay search?**
You don't. Split the query. Use `_sacat` to narrow to subcategories, add price ranges or condition filters, or run the same keyword with different sort orders and deduplicate on item ID afterwards.

**Where does the item ID come from?**
The URL. eBay item numbers are 12 digits, formatted as `ebay.com/itm/406857460314`. Read the canonical link on the page and take everything between `/itm/` and the query string.

**How do I paginate a search?**
Increment `_pgn` from 2 up to the page count, which you get by dividing the total result count from `.srp-controls__count-heading>span` by your `_ipg` value and rounding up. Keep `_ipg` fixed across the run or your page boundaries shift.

**Can I get sold prices at all?**
Not from signed-out scraping, and not from the Browse API. Anything showing historical sold data is either using logged-in sessions, a licensed data feed, or estimating.

**Is scraping eBay legal?**
eBay listing data is public, and scraping it at a polite rate with respect for the servers falls within what's generally treated as ethical scraping. Two caveats that actually matter: you must stay within eBay's terms of service and control your request rate, and if you store seller names or locations you are handling personal data, which brings GDPR into play for EU sellers. This is not legal advice; check your own jurisdiction and use case.

**Why do my prices change between runs?**
The exit IP's country. Currency, shipping estimates and listing visibility all follow the geolocation of the request, so pin `country_code` and record the marketplace and timestamp on every row.

## Which plan to buy

Match the tier to the shape of the job, not the total listing count.

**A few hundred listings a week for price spot-checks.** The free trial's 5,000 credits cover about 330 eBay requests, which is plenty to prove your parser works and see real cost per request in the response header before you pay anything.

**Daily monitoring of a few thousand listings, or a category you refresh continuously.** Startup at $149/month. Search-level scraping at 60 listings per request means a 10,000-listing daily refresh costs well under a dollar in credits, and the allowance leaves room for detail-page enrichment on the items that matter. Annual billing brings it to $134.10/month.

**Variant-level stock tracking, or full-detail enrichment across tens of thousands of items.** This is where the per-item cost of the structured endpoint starts to dominate, and Scaling's 5,000,000 credits at a lower effective rate per credit is the cheaper place to be. At $475/month the per-credit rate drops from $0.000149 to $0.000095, roughly a third less per request.

Before you check out, confirm two things on the pricing page: whether your eBay URLs actually price at 5 credits or higher in the Domain Cost Estimator once the Akamai bypass is on, and what your overage path is. Startup and Scaling both support auto-upgrade, and Scaling adds pay-as-you-go at a fixed per-credit rate up to a cap you set, which is the safer configuration for a job that spikes during sale events.

👉 [Compare ScraperAPI plans and start with the free trial](https://www.scraperapi.com/pricing?fp_ref=coupons)
