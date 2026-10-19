<p align="center"><a href="https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars"><img src="assets/banner.svg" alt="Mobile.bg Bulgaria Car Scraper — Python example" width="100%"></a></p>

# Mobile.bg Bulgaria Scraper in Python: used-car ads to pandas, JSON and CSV

<p align="center">
<a href="https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repo is a short Python wrapper around the **[Mobile.bg Bulgaria Car Scraper](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars)**, an Actor that runs on Apify. Hand it a mobile.bg results link, or nothing at all, and it gives back one row per car ad with the numbers analysts actually need: asking price in euros with the lev amount beside it, model year, kilometres on the clock, horsepower, displacement and the photo links. Apify handles the fetching on its side, so the code here only starts a run, waits, and writes `results.json` plus `results.csv`. From there a notebook takes over.

> **Actor:** [Mobile.bg Bulgaria Car Scraper](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [mobile-bg-bulgaria-scraper-javascript](https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-javascript)

## 🔥 What you get

- **Aim it with a link, or don't**: paste a results page you already narrowed on mobile.bg into `searchUrl`, or leave it out and the Actor reads the general cars-and-SUVs feed.
- **Two currencies, both integers**: `price` holds euros and `price_bgn` holds the lev figure when the ad shows one, so arithmetic starts immediately.
- **The spec sheet in columns**: year, `mileage_km`, `engine_power_hp`, `engine_size_cc`, fuel, gearbox, body style, colour and the first-registration month each get their own field.
- **Cyrillic that reads correctly**: the site serves the windows-1251 encoding, and the Actor decodes it properly, so `Дизелов` arrives as `Дизелов` and not as mojibake.
- **Optional detail visit**: with `enrichDetails` on, each ad page is opened as well to add the town, a private-or-dealer flag and the seller's own text, and to fill spec gaps the result card left.
- **One row per ad**: an ad that shows up on two result pages is written once, keyed by `listing_id`.
- **Refusals are its problem**: when the site rejects a request or answers with a server error, the Actor waits, opens a new connection and tries again, up to five times per page.

## 🎯 Use cases

### Price a model against its age and kilometres

Pull a few hundred ads for one make, then regress `price` on `year` and `mileage_km` in a notebook. The coefficients give you a rough depreciation rate per year and per 10,000 km for the Bulgarian market, which is a useful baseline before you value a fleet or a trade-in.

### Snapshot the market every night with cron

A crontab line such as `0 3 * * * python main.py --out snapshots/$(date +%F)` leaves you a folder of daily files. Concatenate them later and track how long each `listing_id` stays up and how its price moves before it disappears.

### Compare Bulgarian prices with another EU market

Because `price` is already in euros, you can line Mobile.bg rows up against German or Croatian car data without a currency step. Join on make, model and year and the gap per model becomes a pivot table, which is the first question in any import calculation.

### Profile the diesel versus petrol mix

Count `fuel_type` values per body style or per price band. It shows quickly how much of the Bulgarian second-hand stock is still diesel.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-python.git
cd mobile-bg-bulgaria-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Mobile.bg Bulgaria Car Scraper](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Strip away the file handling and this is what remains. `call()` returns only after the Actor has finished; `iterate_items()` then pages through the dataset lazily, so a thousand cars never sit in memory twice.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/mobile-bg-bulgaria-cars").call(run_input={'maxResults': 10, 'enrichDetails': True, 'proxyConfiguration': {'useApifyProxy': True, 'apifyProxyGroups': ['RESIDENTIAL']}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

Whatever sits in `input.json` is sent to the Actor without changes. The sample has no `searchUrl`, so it reads the default feed, stops at 10 cars and keeps the detail pass on; at the listed prices that is $0.20 for the start plus $0.02 for the rows. Keep the residential proxy group: the Actor refuses to run without it, because the site turns away other traffic. To study one make, filter on mobile.bg, copy the address bar into `searchUrl` and raise `maxResults` toward its limit of 1,000.

```json
{
  "maxResults": 10,
  "enrichDetails": true,
  "proxyConfiguration": {
    "useApifyProxy": true,
    "apifyProxyGroups": [
      "RESIDENTIAL"
    ]
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `searchUrl` | string | no | — | — |
| `maxResults` | integer | no | `50` | — |
| `enrichDetails` | boolean | no | `true` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true, "apifyProxyGroups": ["RESIDENTIAL"]}` | — |

## 📤 Output

Each record is a single ad. Four fields are always set (`listing_id`, `listing_url`, `title`, `scraped_at`); plan for nulls in everything else. Category-like values such as fuel, gearbox, body and colour keep the site's Bulgarian words, which keeps them faithful but means you translate them yourself. `make` is the first word of the ad title and `model` the next one.

| Field | Type | Always present |
|---|---|---|
| `listing_id` | string | yes |
| `listing_url` | string | yes |
| `title` | string | yes |
| `make` | string or null | no |
| `model` | string or null | no |
| `year` | integer or null | no |
| `price` | integer or null | no |
| `currency` | string or null | no |
| `price_bgn` | integer or null | no |
| `mileage_km` | integer or null | no |
| `fuel_type` | string or null | no |
| `transmission` | string or null | no |
| `engine_power_hp` | integer or null | no |
| `engine_size_cc` | integer or null | no |
| `body_type` | string or null | no |
| `color` | string or null | no |
| `first_registration` | string or null | no |
| `location` | string or null | no |
| `region` | string or null | no |
| `seller_type` | string or null | no |
| `seller_name` | string or null | no |
| `photo_urls` | array | no |
| `description` | string or null | no |
| `posted_date` | string or null | no |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| result-row | $0.002 |

Prices as configured on 2026-10-16. The [Actor page](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

A starting analysis with pandas (`pip install pandas`): translate the common fuel labels, then for each make, model and fuel type report how many ads there are, the median euro price, median age, median kilometres and euros per thousand km. The result also lands in `bulgaria_models.csv`.

```python
import json
from datetime import date

import pandas as pd

cars = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
fuel_en = {"Дизелов": "diesel", "Бензинов": "petrol", "Електрически": "electric", "Хибриден": "hybrid"}
cars["fuel"] = cars["fuel_type"].map(fuel_en).fillna(cars["fuel_type"])

usable = cars.dropna(subset=["price", "year", "mileage_km"]).copy()
usable["age"] = date.today().year - usable["year"]
usable["eur_per_1000_km"] = usable["price"] / (usable["mileage_km"] / 1000).clip(lower=1)

table = (
    usable.groupby(["make", "model", "fuel"])
    .agg(
        listings=("listing_id", "nunique"),
        median_eur=("price", "median"),
        median_age=("age", "median"),
        median_km=("mileage_km", "median"),
        eur_per_1000_km=("eur_per_1000_km", "median"),
    )
    .sort_values("listings", ascending=False)
)
table.to_csv("bulgaria_models.csv")
print(table.round(1).head(15).to_string())
```

## ❓ FAQ

### Does Mobile.bg have an official API I could call instead?

Not a public one. The Actor reads the same pages a visitor sees, and the only key in this repository is your Apify token.

### Why does the run fail straight away without a proxy group?

Mobile.bg answers with 403 to most non-residential traffic, so the Actor insists on the Apify Proxy `RESIDENTIAL` group and stops with a clear message if it is switched off or not available on your plan. Check your plan before you schedule anything.

### Why are `region` and `seller_name` always empty?

They exist in the schema but the current version never fills them. The detail pass gives you `location` (for example `гр. Варна`) and `seller_type` instead, so group by town rather than region for now.

### How many ads can I collect in one run?

`maxResults` allows up to 1,000, and the Actor walks at most 60 result pages. Very broad searches can also be cut short by the site's own paging, so split big pulls by make or price band.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Mobile.bg Bulgaria Car Scraper on Apify Store](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars)
- [JavaScript version of this example](https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [AutoScout24 Germany Car Scraper](https://apify.com/DevilScrapes/autoscout24-germany-cars) — German used-car ads, handy as a Western price benchmark.
- [Njuškalo Car Scraper (Croatia)](https://apify.com/DevilScrapes/njuskalo-croatia-cars) — car classifieds from Croatia, another Balkan market.
- [Blocket Car Scraper (Sweden)](https://apify.com/DevilScrapes/blocket-sweden-cars) — Swedish car ads with SEK prices for a Nordic comparison.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Mobile.bg.
