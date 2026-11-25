# Crawling API: What Does a 10,000-Page Crawl Cost, Which Defaults Cut Your Coverage, and When Is the Cheapest Tier Enough?

Three settings decide what a crawling API costs you: how pages are counted, which output format you request, and which links the crawler is allowed to follow. On Firecrawl's published rates, a 10,000-page site costs 10,000 credits as a plain markdown crawl and 50,000 credits for the same crawl with JSON output. That five-fold gap sits entirely in the format flag.

Below is the arithmetic with numbers taken from vendor documentation, the default settings that quietly shrink a crawl to a fraction of a site, and what to confirm on ScraperAPI's plan page before you commit to a tier you may not need.

> **Quick picks**
> - You already have a URL list: buy the smallest tier that covers your monthly page count, then check the concurrency it includes.
> - You need link discovery across a whole site for RAG: price the format premium and the scope defaults first, because headline price per page is only half the comparison.
> - You need a robots.txt override or zero data retention: on at least one vendor those are contract features, not plan features.
>
> 👉 [Check ScraperAPI's current plans and per-tier limits](https://www.scraperapi.com/?fp_ref=coupons)

## The three levers that set the bill

**How a page is counted.** Firecrawl charges one credit per crawled page and bills pages as they finish, with `limit` capping how many pages a single job can crawl. crawlcrawl counts every URL as one page against your daily or monthly quota regardless of whether the fetch was a cache hit, and states this explicitly in its bulk endpoint documentation. Any per-page plan you compare should be read the same way: what counts as a page, and does a retry or a cache hit count again.

**What you ask for.** Firecrawl's pricing page notes that the JSON format adds 4 Firecrawl credits per page on top of the base page cost. Markdown, HTML, links and summaries do not carry that surcharge. If your pipeline needs structured fields rather than raw text, budget 5x the base page cost, not 1x.

**How far the crawler is allowed to walk.** Scope defaults vary more between vendors than prices do, and they decide whether a "10,000-page" allowance ever gets spent on the pages you wanted.

## What 10,000 pages actually costs

Take Firecrawl's published tiers, all of which are monthly equivalents billed yearly:

| Plan | Included per month | Price (billed yearly) | Concurrent requests | Overage |
| --- | --- | --- | --- | --- |
| Free | 500 searches or 1,000 pages | $0 | 2 | Low rate limits |
| Hobby | 2,500 searches or 5,000 pages | $16/mo | 5 | $9 per extra 1.5k credits |
| Standard | 50,000 searches or 100,000 pages | $83/mo | 25 | $47 per extra 35k credits |
| Growth | 250,000 searches or 500,000 pages | $333/mo | 50 | $177 per extra 175k credits |
| Scale | 500,000 searches or 1,000,000 pages | $599/mo | 100 | $397 per extra 350k credits |
| Enterprise | Custom credits | Custom | Custom | Bulk discounts |

*Prices are Firecrawl's own published figures; the page states "billed yearly" and publishes no monthly-billed price, so don't assume one.*

A 10,000-page markdown crawl is 10,000 credits, which is one tenth of the Standard allowance and five times the Hobby allowance. Add JSON and it becomes 50,000 credits, half of Standard. Either way a 10,000-page job lands on Standard, and Scale's million credits are for teams crawling six figures of pages every month, not once.

The overage rates are worth reading closely before you decide the top tier is the safe choice. Growth's overage works out to about $1.01 per 1,000 credits; Scale's is about $1.13 per 1,000. Scale's bundled rate is lower than Growth's ($0.60 versus $0.67 per 1,000 pages at the yearly-billed prices), but if your usage is spiky and you expect to lean on overage, the cheaper bundle does not automatically mean the cheaper month.

There is one more arbitrage. At 10,000 pages a month, Hobby plus overage costs about $36 in extra credits on top of $16, roughly $52, against $83 for Standard. You would trade 25 concurrent requests for 5. Firecrawl's page does not state whether overage is capped, so confirm that before planning around it.

## Defaults that will leave pages out

Firecrawl's API reference documents defaults that surprise people who assume a crawl means "the whole site":

- `crawlEntireDomain` defaults to false. From a start URL like `/features/feature-1`, the crawler follows deeper child paths and will not follow `/pricing` or `/`.
- `allowExternalLinks` defaults to false, and when enabled the crawler follows external links exactly one hop. Links pointing at an external site's homepage are skipped and reported in the crawl errors endpoint with the code `EXTERNAL_LINK`, and so are redirects to an external homepage.
- `allowSubdomains` defaults to false, so `blog.example.com` is out of scope unless you ask for it.
- `limit` defaults to 10,000 pages, and `sitemap` defaults to `include`, meaning the crawler reads the sitemap as well as following links.
- `ignoreQueryParameters` and `regexOnFullURL` both default to false, so query-string variants are treated as separate pages and path regexes match the pathname only.

crawlcrawl ships a different set of traps: `max_pages` defaults to 1,000 with a hard ceiling of 100,000, `depth` defaults to 5 (valid range 1 to 50), and `subdomains` defaults to false. A "full site" crawl left on defaults caps at a thousand pages and stays on one host.

Whichever vendor you pick, three switches deserve a look before the first production run: domain scope (siblings, parents, subdomains), depth or page cap, and whether the sitemap is used as a seed.

## robots.txt is not uniformly negotiable

Firecrawl follows robots.txt by default. Its `ignoreRobotsTxt` and `robotsUserAgent` parameters are marked Enterprise only, with a support contact required to enable them. crawlcrawl sets `respect_robots` to true by default but exposes it as a request field you can flip per crawl. If your use case is a compliance or accessibility audit where robots rules need to be overridden and logged, that difference matters more than a few dollars of credit.

## Concurrency, 429s and failed pages

crawlcrawl allows 5 concurrent runs; submitting a sixth while five are queued or running returns a 429. Its webhook fires once on a terminal state with five attempts on a 1s/2s/4s/8s backoff, and a 4xx response is treated as dead immediately with no retry, while 5xx, network errors and timeouts are retried.

Firecrawl applies your team's concurrency limit unless `maxConcurrency` is set, and setting `delay` forces concurrency down to 1, which is the wrong move on a large crawl unless the target site is actively rate-limiting you.

Failed pages are reported rather than hidden on both: Firecrawl lists failed and robots-blocked URLs at `GET /v2/crawl/{id}/errors`, and crawlcrawl returns 4xx and 5xx target pages as pages with a status field, so you filter by status when you only want successes.

## The three options side by side

|  | Firecrawl | crawlcrawl | Bright Data |
| --- | --- | --- | --- |
| Per-page billing | 1 credit per finished page; JSON adds 4 credits/page | Every URL counts as 1 page, cache hits included | Not published |
| Default page cap | 10,000 | 1,000 (hard cap 100,000) | Not published |
| Default scope | Child paths only; subdomains and external links off | Subdomains off; depth 5 | Not published |
| robots.txt | Followed by default; override is Enterprise only | Followed by default; can be set false per crawl | Not published |
| Concurrency | Per tier, 5 to 100 on published plans | 5 concurrent runs, sixth returns 429 | Not published |
| Pricing | Five published tiers plus Enterprise | Not published | Enterprise, quote only |

Bright Data's crawl product is sold through a contact-us Enterprise tier with no published price, and its public page has not been refreshed recently enough to treat its scale figures as current. Treat it as the option you shortlist when procurement needs a custom contract, SLA and audit logs.

## ScraperAPI: the four numbers to settle before you pay

If your workload is a long list of URLs rather than a link-discovery problem, the cost model you want is one where you pay for pages fetched and the plans scale with volume. ScraperAPI is the plan page to open first for that shape of work, and four numbers on it decide everything:

1. **Does the plan include a crawl mode, or only single-URL fetching?** Discovery changes the tool you need. If you must supply every URL yourself, your engineering time becomes part of the price.
2. **What does one page cost in credits, and does rendering or geo-targeting add to it?** A rendered page and a plain HTML fetch are not the same unit on most credit-based plans, and that multiplier decides whether a starter tier lasts a week or a quarter.
3. **How many concurrent requests does each tier include?** On Firecrawl the jump from 5 to 25 concurrent requests is the real reason to move from Hobby to Standard, not the page allowance.
4. **How is the billing period structured, and do unused credits carry over?** Monthly and annual commitments price the same allowance differently, and expiring credits change your effective rate.

👉 [Open ScraperAPI's pricing page and check the credit cost of a rendered page](https://www.scraperapi.com/?fp_ref=coupons)

Most readers do not need the largest tier anywhere in this comparison. A 10,000-page markdown crawl fits inside Firecrawl's Standard plan ten times over; the tiers above it exist for teams running recurring crawls of hundreds of thousands of pages. Buy for your measured monthly page count plus a margin, then upgrade when the overage bill exceeds the next tier's price for two consecutive months.

## FAQ

**Is a free tier enough to test with?** Firecrawl's free plan covers 500 searches or 1,000 pages with 2 concurrent requests. That is enough to validate output quality on a docs section, not enough to benchmark a full-site crawl, because concurrency, not credits, will be your bottleneck.

**Does JavaScript rendering cost extra?** On Firecrawl, every page goes through the scrape pipeline and renders JavaScript by default at 1 credit per page. The 4-credit surcharge applies to the JSON format, not to rendering. Check this separately on any credit-based plan, since some vendors bill rendered pages at a higher rate.

**Are failed pages billed?** Firecrawl bills pages as they finish and directs you to the errors endpoint for failures and robots-blocked URLs; its pricing page does not spell out how failed pages are charged. Get that in writing before a large run against a flaky site.

**Can I crawl only one section, like /docs?** Firecrawl's `includePaths` takes regex patterns, uses Rust regex (RE2-style) syntax without look-around or backreferences, and caps each field at 1,000 patterns of 2,000 characters. The starting URL is also checked against the pattern, so a mismatch can return zero pages, and you will spend the run discovering that.

**Can I switch from annual to monthly billing?** Firecrawl publishes only yearly-billed monthly equivalents, with no monthly price listed, so budget on the annual commitment and ask support for the monthly rate if you need flexibility.

## Which one to buy

Only need to fetch a list of URLs you already have, at volume: the lowest tier that covers your monthly page count, and confirm the concurrency number rather than the credit number first.

Need whole-site discovery with markdown or JSON for a RAG corpus: check domain scope, page cap and format premium before price. Those three decide whether your allowance reaches the pages you care about.

Need robots.txt overrides, zero data retention or an SLA: that is an Enterprise conversation, and no published plan will include it.

Two things to confirm before you enter card details anywhere: whether unused credits expire at the end of the billing period, and how many days you have to request a refund.

👉 [Compare ScraperAPI's plans and start with the smallest tier that fits](https://www.scraperapi.com/?fp_ref=coupons)
