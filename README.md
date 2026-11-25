# google scraper python: the requests + BeautifulSoup route, the captcha wall, and what a managed SERP API costs per 1,000 searches

A `requests` + BeautifulSoup script still returns the titles and links on a Google results page, and it takes about twenty lines of Python. The tutorials you have already read were right about that part. What they leave out is the bill for keeping it running: page two, results as they appear in another country, a few thousand queries a day, and Google's HTML moving under your parser.

ScraperAPI prices a Google SERP request at 25 credits. On the Startup plan that works out to roughly **$3.73 per 1,000 searches** on monthly billing, or **$3.35** if you pay yearly. That number is the whole argument, so the rest of this piece is the code path, the three points where the DIY version breaks, and the plan grid with credits converted into actual Google searches.

> **Quick pick.** A one-off job of a few hundred queries: write the script. A recurring job (rank checks, price monitoring, feeding fresh results to an LLM): Hobby's 100,000 credits is 4,000 plain Google searches a month, enough for most side projects; Startup at 1,000,000 credits is 40,000 searches for $149/month, or $134.10/month billed yearly. The free tier's 1,000 credits is 40 searches, which tells you whether your key works and nothing more. 👉 [See the current plan grid and prices](https://scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan, with credits converted into Google searches

| Plan | API credits / month | Plain Google SERP requests / month | Concurrent threads | Price (monthly / yearly) | Sign up |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 40 | 5 | $0 | [Free tier details](https://scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 4,000 | 20 | — | [Hobby pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 40,000 | 50 | $149 / $134.10 | [Startup pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 120,000 | 100 | — | [Business pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200,000 | 200 | — | [Scaling pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 420,000 | 300 | — | [Professional pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 860,000 | 500 | — | [Advanced pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 880,000+ | 500+ | — | [Enterprise pricing](https://scraperapi.com/pricing?fp_ref=coupons) |

Dashes in the price column are tiers whose rate is quoted on the pricing page rather than in the plan description. Yearly billing takes 10% off every tier, so the annual figure is always the cheaper per-month number.

Two things in that grid matter more than the price ladder. The search-per-month column assumes a plain Google request at 25 credits, and add-ons push it higher. Geotargeting is listed as US & EU on Hobby and Startup, then becomes country-level from Business upward, so if your queries target Brazil or Japan, the cheap tiers are not the answer.

## What a Google results page actually contains

SERP stands for Search Engine Results Page, the page you land on after typing a query. It is not ten blue links. A single page can hold a featured snippet, paid ads above and below the organic list, a video carousel, a People also ask block, a local pack with map results, and related searches at the bottom.

Which of those your script treats as "a result" is a decision you make in the parser, because Google does not label the blocks for you. That is the quiet reason people move from HTML scraping to a structured response: the page is heterogeneous, and mapping it yourself is ongoing work.

## The self-built version, in about twenty lines

Two dependencies, both from pip: `requests` and `bs4`. Google takes the query in the `q` URL parameter and expects spaces as `+`:

python
query = "google scraper python".replace(" ", "+")
url = f"https://www.google.com/search?q={query}"


The user-agent changes what you get back. Google serves different results to a phone and a desktop, so pick one on purpose and send it in the headers:

python
headers = {
    "user-agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:65.0) Gecko/20100101 Firefox/65.0"
}
resp = requests.get(url, headers=headers)


Parsing is the part everyone copies. Walk the result blocks, take the first anchor's `href` as the link and the `h3` text as the title:

python
from bs4 import BeautifulSoup

if resp.status_code == 200:
    soup = BeautifulSoup(resp.content, "html.parser")
    results = []
    for block in soup.find_all("div", class_="r"):
        anchors = block.find_all("a")
        if anchors:
            results.append({
                "title": block.find("h3").text,
                "link": anchors[0]["href"],
            })
    print(results)


Those two selectors, `div class="r"` and `h3`, come from tutorials published in 2019 and 2020. Google's markup is not a contract, so run one query and print `len(results)` before you build anything on top of the parser. If the wrapper class has moved, `soup.find_all('h3')` still lists the headings, which at least tells you the page came back at all.

## Three places the script stops being enough

### Captchas and rate limits

Push too many requests through in a short window and Google starts throwing captchas at you. The author of the pattern above described his own script as "pretty simple and error-prone" and flagged this exact wall. It does not just slow you down. It forces retry logic, backoff, a CAPTCHA solving service, and a proxy pool to spread the traffic, which is where a weekend project turns into infrastructure.

### Selector drift

Every block on the page is styled, and styles change. The parser you wrote against last quarter's markup returns an empty list the day Google ships a layout change, and nothing in your code tells you whether the page was blocked or simply moved. You find out by noticing that your data stopped.

### Location, language and pagination

The same query returns different results in Berlin, Toronto and Singapore. Reaching another country's results from your own IP means a proxy in that country plus the correct `hl` and `gl` parameters, and pagination means another URL (`&start=10`, `&start=20`) and another request that can hit the captcha wall. All three of those get handled once, in a loop, or not at all.

## What the DIY version really costs

Add up the recurring items before you decide. A residential proxy plan, a CAPTCHA service, and the hours you spend re-verifying selectors whenever the markup shifts. None of those line items appear in a tutorial, and all three come back every month. The honest comparison is not "free script vs paid API." It is your proxy and CAPTCHA spend plus your maintenance time against a per-request price.

👉 [Check the credit cost table before you compare it to your proxy bill](https://scraperapi.com/pricing?fp_ref=coupons)

## What the managed route looks like in code

ScraperAPI's documented Google endpoint is asynchronous: you post the job, and the result is pushed to a webhook you supply.

python
import requests

payload = {
    "apiKey": "API_KEY",
    "query": "google scraper python",
    "country_code": "de",
    "tld": "de",
    "hl": "en",
    "gl": "de",
    "start": 0,
    "output_format": "json",
    "callback": {"type": "webhook", "url": "https://your-server.example/serp"},
}

r = requests.post(
    "https://async.scraperapi.com/structured/google/search",
    json=payload,
)
print(r.status_code, r.text)


What the parameters buy you:

- `query` for a single search, or a `queries` array to batch several in one job.
- `tld` picks the Google domain (`com`, `co.uk`, `de`, `fr`, `co.jp`, `com.au`, `in` and others). When a Google domain has to be scraped from a different country, both `tld` and `country_code` are required. The payload above returns google.de results in English from a German location.
- Google's own parameters carry through: `num` for result count, `start` for the offset, `tbs` for a time window (past hour, day, week, month, year), `uule` for a named region, `hl` and `gl` for language and country bias.
- `output_format` takes `json` (default) or `csv`.

There is no BeautifulSoup step on your side, because the response is already structured JSON rather than a page of HTML.

## The credit math, in search terms

Google and Bing are the expensive targets on ScraperAPI's credit table at **25 credits per request**, subdomains included. A standard page is 1 credit, Amazon, Walmart and eBay are 5, LinkedIn is 30. Add-ons stack on top: bot-protection bypass, `render=true`, `premium=true` and `screenshot=true` each add 10 credits, so a Google request that also needs rendering is not a 25-credit request any more.

Geotargeting, sessions, device type, output format and auto parsing cost nothing extra. Those are exactly the parameters you would otherwise build proxy rotation to fake, which is why they are free here.

Run the arithmetic against your own volume:

- Free tier: 1,000 credits = **40 Google searches**
- Hobby: 100,000 credits = **4,000 Google searches**
- Startup: 1,000,000 credits = **40,000 Google searches** for $149/month, or $1,609.20/year at $134.10/month

At Startup's monthly rate that is $3.73 per 1,000 searches. A rank-tracking job covering 200 keywords twice a day stays inside Hobby with room to spare. A pipeline that re-checks 5,000 product queries daily does not.

Two overage behaviours are worth knowing before you pick a tier. When credits run out, scraping pauses until the subscription renews and the balance resets. Auto-upgrade, which moves you to the next plan automatically, is opt-in on Hobby, Startup, Business, Scaling and Professional. Pay-as-you-go at a fixed per-credit rate with a monthly cap you set is available on Scaling, Professional, Advanced and Enterprise. Every response reports its own cost in the `sa-credit-cost` header, and `max_cost` caps a single request so a difficult page cannot quietly eat a day's budget.

Plans also carry a **7-day, no-questions-asked refund**, which is long enough to run your real query list through the endpoint and see what the credit cost actually is.

👉 [Open the plan page and price out your own query volume](https://scraperapi.com/pricing?fp_ref=coupons)

## Common questions

**Is scraping Google legal?** Scraping publicly available data, including Google SERP data, is generally treated as legal, but the answer varies with your jurisdiction and your use of the data, so it is worth getting legal advice about your specific case rather than relying on a blog post.

**Do I still need BeautifulSoup?** Only if you are working with raw HTML. The structured endpoint returns JSON, and switching `output_format` to `csv` skips the parsing step entirely.

**How do I get the output into a spreadsheet?** Whatever JSON comes back, `pandas.json_normalize()` flattens the nested fields into columns and `df.to_csv("export.csv", index=False)` writes the file. That step is the same whether the JSON came from your own parser or from an API.

**What if I need results from a country outside the US and EU?** Country-level geotargeting starts at Business. On Hobby and Startup you are limited to US & EU targeting, which is the single most common reason people upgrade after a trial.

**How fast can I run requests?** Concurrency is capped per plan: 5 threads on Free, 20 on Hobby, 50 on Startup, up to 500+ on Advanced and Enterprise.

## Which one to pick

Match the tool to the job, then confirm the numbers on the page before checkout.

- **A few hundred queries, once.** Write the twenty-line script. Verify the selectors on one query first, and do not bother with proxies.
- **Weekly rank checks under about 4,000 queries a month.** Hobby covers it at 100,000 credits, and US & EU targeting is probably all you need.
- **Daily jobs, multiple regions, or an LLM pipeline that cannot tolerate an empty result.** Startup or above. Compare $3.73 per 1,000 searches against your proxy plan plus CAPTCHA service plus the afternoons you lose when a parser breaks.
- **Country-level targeting outside the US and EU.** Business is the floor, whatever your volume.

Before you pay, check three things on the plan page: the current price of the tier you want, since yearly billing takes 10% off and the per-month figure is what actually leaves your account, whether your target countries are covered at that tier, and the credit cost of your specific request type once add-ons like rendering are switched on.

👉 [Compare ScraperAPI plans and start with the tier that matches your query volume](https://scraperapi.com/pricing?fp_ref=coupons)
