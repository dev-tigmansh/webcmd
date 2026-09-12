"""
Fashion Price Comparison Agent — skeleton

Compares a product query across Flipkart, Amazon, Myntra, Nykaa Fashion,
Ajio, Shein, Snitch, H&M, and Zara, then returns the best option.

IMPORTANT before you run this for real:
  1. CSS selectors below are PLACEHOLDERS. Every site's DOM changes
     often and is obfuscated/hashed on purpose to break scrapers.
     Inspect each site yourself (DevTools) and fill in real selectors.
  2. Several of these sites run active bot-detection (Akamai, Cloudflare,
     PerimeterX). Plain Playwright will get blocked. You'll likely need:
       - playwright-stealth (pip install playwright-stealth)
       - randomized delays / human-like scrolling
       - residential proxies for sites that hard-block datacenter IPs
       - retry/backoff logic
  3. Respect robots.txt and rate limits. Scraping at scale or for a
     commercial product likely violates these sites' ToS — get legal
     advice before doing anything beyond personal/learning use.
  4. Some sites (Amazon, Flipkart) have affiliate APIs — prefer those
     over scraping wherever eligible; they're stable and legal.

Install:
    pip install playwright rapidfuzz --break-system-packages
    playwright install chromium
"""

import asyncio
import csv
import os
import random
import re
from dataclasses import dataclass, field
from typing import Optional
from urllib.parse import urljoin, urlparse
from playwright.async_api import async_playwright, Page, BrowserContext
from playwright_stealth import Stealth
from rapidfuzz import fuzz
from tabulate import tabulate

# Minimum plausible price in INR — filters out mis-scraped values like
# ratings ("4.1") or review counts that accidentally match the price selector.
MIN_PLAUSIBLE_PRICE = 50.0

# Map a product URL's domain to the site name used in SITE_CONFIG,
# so we know which site a pasted link belongs to (and can skip re-scraping it
# in the comparison, since we already have its price directly).
DOMAIN_TO_SITE = {
    "flipkart.com": "Flipkart",
    "amazon.in": "Amazon",
    "myntra.com": "Myntra",
    "nykaafashion.com": "Nykaa Fashion",
    "ajio.com": "Ajio",
    "shein.in": "Shein",
    "shein.com": "Shein",
    "snitch.co.in": "Snitch",
    "zara.com": "Zara",
}


@dataclass
class Product:
    site: str
    title: str
    price: Optional[float]
    url: str
    image: Optional[str] = None
    currency: str = "INR"
    match_score: float = field(default=0.0)


# ---------------------------------------------------------------------------
# Per-site config. Fill in real selectors after inspecting each site's
# search results page in DevTools (Elements tab -> right-click -> Copy selector).
# ---------------------------------------------------------------------------
SITE_CONFIG = {
    "Flipkart": {
        "search_url": "https://www.flipkart.com/search?q={query}",
        "card_selector": "div._1AtVbE",          # PLACEHOLDER
        "title_selector": "div._4rR01T",          # PLACEHOLDER
        "price_selector": "div._30jeq3",          # PLACEHOLDER
        "link_selector": "a._1fQZEK",             # PLACEHOLDER
    },
    "Amazon": {
        "search_url": "https://www.amazon.in/s?k={query}",
        "card_selector": "div[data-component-type='s-search-result']",
        "title_selector": "span.a-text-normal",
        "price_selector": "span.a-price-whole",
        "link_selector": "a.a-link-normal",
    },
    "Myntra": {
        "search_url": "https://www.myntra.com/{query}",
        "card_selector": "li.product-base",
        "title_selector": "h4.product-product",
        "price_selector": "div.product-discountedPrice, span.product-discountedPrice",
        "link_selector": "a",
    },
    "Nykaa Fashion": {
        "search_url": "https://www.nykaafashion.com/search/?q={query}",
        "card_selector": "div.product-card",       # PLACEHOLDER
        "title_selector": "div.product-title",     # PLACEHOLDER
        "price_selector": "span.product-price",    # PLACEHOLDER
        "link_selector": "a",
    },
    "Ajio": {
        "search_url": "https://www.ajio.com/search/?text={query}",
        "card_selector": "div.item",                # PLACEHOLDER
        "title_selector": "div.nameCls",             # PLACEHOLDER
        "price_selector": "span.price",              # PLACEHOLDER
        "link_selector": "a",
    },
    "Shein": {
        "search_url": "https://www.shein.in/pdsearch/{query}/",
        "card_selector": "section.product-card",    # PLACEHOLDER
        "title_selector": "a.goods-title-link",      # PLACEHOLDER
        "price_selector": "div[data-price]",         # PLACEHOLDER
        "link_selector": "a.goods-title-link",
    },
    "Snitch": {
        "search_url": "https://www.snitch.co.in/search?q={query}",
        "card_selector": "div.product-card",         # PLACEHOLDER
        "title_selector": "div.product-title",       # PLACEHOLDER
        "price_selector": "span.price",               # PLACEHOLDER
        "link_selector": "a",
    },
    "Zara": {
        "search_url": "https://www.zara.com/in/en/search?searchTerm={query}",
        "card_selector": "li.product-grid-product",   # PLACEHOLDER
        "title_selector": "h3",                         # PLACEHOLDER
        "price_selector": "span.money-amount__main",   # PLACEHOLDER
        "link_selector": "a",
    },
}


# A small rotation of realistic desktop user-agents — using a single fixed
# UA for every request is itself a detectable pattern.
USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
    "(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
    "(KHTML, like Gecko) Chrome/123.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 "
    "(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
]

# Launch flags that reduce the most common automation fingerprints.
STEALTH_LAUNCH_ARGS = [
    "--disable-http2",
    "--disable-blink-features=AutomationControlled",
    "--disable-features=IsolateOrigins,site-per-process",
    "--no-sandbox",
    "--disable-dev-shm-usage",
]


async def new_stealth_context(browser) -> BrowserContext:
    """Create a browser context configured to look less like automation:
    randomized UA/viewport, realistic headers, and JS-level patches
    (navigator.webdriver, plugins, etc.) via playwright-stealth."""
    context = await browser.new_context(
        user_agent=random.choice(USER_AGENTS),
        locale="en-IN",
        timezone_id="Asia/Kolkata",
        viewport={
            "width": random.randint(1280, 1920),
            "height": random.randint(800, 1080),
        },
        extra_http_headers={
            "Accept-Language": "en-IN,en;q=0.9",
        },
    )
    return context


async def new_stealth_page(context: BrowserContext) -> Page:
    return await context.new_page()


async def human_pause(min_ms: int = 400, max_ms: int = 1200):
    """Random delay to avoid the perfectly-uniform timing that flags bots."""
    await asyncio.sleep(random.uniform(min_ms, max_ms) / 1000)


FASHION_CATEGORIES = [
    # multi-word first so they match before their single-word substrings
    "kurta set", "anarkali kurta", "t shirt", "t-shirt", "sweatshirt", "hoodie",
    "jacket", "kurta", "saree", "dress", "jeans", "trousers", "shorts", "skirt",
    "top", "shirt", "coat", "blazer", "leggings", "joggers", "co-ord set",
    "jumpsuit", "sneakers", "shoes", "sandals", "heels", "bag", "watch",
]

FASHION_COLORS = [
    "black", "white", "red", "blue", "green", "yellow", "pink", "purple",
    "orange", "brown", "grey", "gray", "maroon", "navy", "beige", "olive",
    "cream", "gold", "silver", "multicolor", "printed", "floral",
]

FASHION_GENDERS = ["men", "women", "boys", "girls", "unisex", "kids"]


def _find_keyword(lower_text: str, keywords: list[str]) -> Optional[str]:
    """Find the first keyword present as a whole word/phrase — NOT as a
    substring, since plain 'in' checks misfire (e.g. 'men' matching inside
    'women', or 'top' matching inside 'laptop')."""
    for kw in keywords:
        if re.search(r"\b" + re.escape(kw) + r"\b", lower_text):
            return kw
    return None


def simplify_query(title: str) -> str:
    """Reduce a full scraped product title (often brand + seller + fabric +
    fit descriptors) down to a short generic query like 'women printed kurta',
    so searching OTHER sites isn't crippled by a brand name that only exists
    on the source site. Falls back to the last few words if no keywords match."""
    lower = title.lower()

    gender = _find_keyword(lower, FASHION_GENDERS)
    color = _find_keyword(lower, FASHION_COLORS)
    category = _find_keyword(lower, FASHION_CATEGORIES)

    parts = [p for p in [gender, color, category] if p]
    if category:  # only trust the simplified version if we actually found a product type
        return " ".join(parts)

    # Fallback: last 4 words, which usually drops the leading brand name
    words = title.split()
    return " ".join(words[-4:]) if len(words) > 4 else title


def parse_price(raw: str) -> Optional[float]:
    """Extract a float price from messy scraped text like '₹1,299' or 'Rs. 999.00'."""
    if not raw:
        return None
    cleaned = re.sub(r"[^\d.]", "", raw)
    try:
        return float(cleaned) if cleaned else None
    except ValueError:
        return None


async def scrape_site(page: Page, site_name: str, query: str, limit: int = 8, retries: int = 2) -> list[Product]:
    cfg = SITE_CONFIG[site_name]
    url = cfg["search_url"].format(query=query.replace(" ", "-" if site_name == "Myntra" else "+"))

    results: list[Product] = []
    last_error = None

    for attempt in range(1, retries + 1):
        try:
            await human_pause()  # stagger navigation so requests don't fire in lockstep
            await page.goto(url, timeout=40000, wait_until="domcontentloaded")
            await page.wait_for_timeout(random.randint(1800, 3000))  # let JS-rendered content settle

            cards = await page.query_selector_all(cfg["card_selector"])
            skipped_missing_title = skipped_missing_price = skipped_missing_link = 0

            for card in cards[:limit]:
                title_el = await card.query_selector(cfg["title_selector"])
                price_el = await card.query_selector(cfg["price_selector"])
                link_el = await card.query_selector(cfg["link_selector"])

                title = (await title_el.inner_text()).strip() if title_el else None
                price_raw = (await price_el.inner_text()).strip() if price_el else None
                href = (await link_el.get_attribute("href")) if link_el else None

                if not title:
                    skipped_missing_title += 1
                if not price_raw:
                    skipped_missing_price += 1
                if not href:
                    skipped_missing_link += 1
                if not title or not price_raw or not href:
                    continue

                href = urljoin(url, href)  # handles absolute, //, /path, and relative hrefs

                results.append(Product(
                    site=site_name,
                    title=title,
                    price=parse_price(price_raw),
                    url=href,
                ))

            # Always report what happened — silent zero-result cases are the
            # hardest to debug, so make every site's outcome visible.
            print(f"[{site_name}] found {len(cards)} card(s), "
                  f"extracted {len(results)} valid product(s)"
                  + (f"  — skipped: {skipped_missing_title} missing title, "
                     f"{skipped_missing_price} missing price, "
                     f"{skipped_missing_link} missing link"
                     if len(cards) and not results else ""))
            return results  # success — no need to retry

        except Exception as e:
            last_error = e
            if attempt < retries:
                await human_pause(1000, 2000)  # back off before retrying
                continue

    print(f"[{site_name}] scrape failed after {retries} attempts: {last_error}")
    return results


def rank_matches(match_title: str, products: list[Product]) -> list[Product]:
    """Score each scraped product against the reference title (the original
    full product name, NOT the short search query) using fuzzy matching."""
    for p in products:
        p.match_score = fuzz.token_set_ratio(match_title.lower(), p.title.lower())

    # Keep only plausible matches, then sort by price ascending
    plausible = [
        p for p in products
        if p.match_score >= 45 and p.price and p.price >= MIN_PLAUSIBLE_PRICE
    ]

    # Visibility: show per-site how many extracted products survived filtering,
    # since a site can extract results fine and still lose all of them here
    # (low match score, or an implausible/mis-scraped price).
    sites_seen = {p.site for p in products}
    for site in sites_seen:
        extracted = [p for p in products if p.site == site]
        kept = [p for p in plausible if p.site == site]
        if extracted and not kept:
            best_score = max(p.match_score for p in extracted)
            print(f"[{site}] {len(extracted)} product(s) extracted but 0 kept "
                  f"after filtering (best match score was {best_score:.0f}, "
                  f"threshold is 45)")

    return sorted(plausible, key=lambda p: p.price)


async def extract_product_info(page: Page, product_url: str) -> dict:
    """
    Given a direct product-page URL from any of the supported sites, extract
    the product name (and price, if available) using metadata that most
    e-commerce sites embed regardless of their visible page structure:
      1. JSON-LD structured data (schema.org Product) — most reliable
      2. Open Graph meta tags (og:title, product:price:amount)
      3. <title> tag — last-resort fallback
    This avoids needing hand-picked CSS selectors for every product page.
    """
    await human_pause()
    await page.goto(product_url, timeout=40000, wait_until="domcontentloaded")
    await page.wait_for_timeout(random.randint(1200, 2200))

    info = await page.evaluate("""
        () => {
            let name = null, price = null;

            // 1. JSON-LD Product schema
            const scripts = document.querySelectorAll('script[type="application/ld+json"]');
            for (const s of scripts) {
                try {
                    const data = JSON.parse(s.textContent);
                    const items = Array.isArray(data) ? data : [data];
                    for (const item of items) {
                        if (item['@type'] === 'Product') {
                            if (item.name) name = item.name;
                            const offers = Array.isArray(item.offers) ? item.offers[0] : item.offers;
                            if (offers && offers.price) price = offers.price;
                        }
                    }
                } catch (e) {}
            }

            // 2. Open Graph fallback
            if (!name) {
                const og = document.querySelector('meta[property="og:title"]');
                if (og) name = og.content;
            }
            if (!price) {
                const ogPrice = document.querySelector('meta[property="product:price:amount"], meta[property="og:price:amount"]');
                if (ogPrice) price = ogPrice.content;
            }

            // 3. Last resort
            if (!name) name = document.title;

            return { name, price };
        }
    """)
    return info


def detect_site_from_url(product_url: str) -> Optional[str]:
    netloc = urlparse(product_url).netloc.lower().replace("www.", "")
    for domain, site_name in DOMAIN_TO_SITE.items():
        if netloc.endswith(domain):
            return site_name
    return None


def clean_extracted_title(raw_title: str, site_name: Optional[str]) -> str:
    """Product-page <title> tags often look like 'Product Name | Brand - Site.com'.
    Strip the trailing site/brand noise so it works better as a search query."""
    title = raw_title
    for sep in [" | ", " - ", " – ", " :: "]:
        if sep in title:
            title = title.split(sep)[0]
    return title.strip()


async def compare_price(search_query: str, match_title: Optional[str] = None,
                         exclude_site: Optional[str] = None) -> list[Product]:
    """search_query: short generic terms used to actually search each site.
    match_title: the full original product name used only for scoring how
    close each result is. Defaults to search_query when not provided
    (e.g. when the user typed a plain search query rather than a link)."""
    match_title = match_title or search_query
    all_results: list[Product] = []

    async with Stealth().use_async(async_playwright()) as pw:
        browser = await pw.chromium.launch(
            headless=False,     # visible window so you can see what each site returns
            slow_mo=150,
            args=STEALTH_LAUNCH_ARGS,
        )
        context = await new_stealth_context(browser)

        pages_and_sites = []
        for site_name in SITE_CONFIG:
            if site_name == exclude_site:
                continue
            page = await new_stealth_page(context)
            pages_and_sites.append((page, site_name))

        # Run all site scrapes concurrently
        tasks = [scrape_site(page, site, search_query) for page, site in pages_and_sites]
        per_site_results = await asyncio.gather(*tasks)

        for r in per_site_results:
            all_results.extend(r)

        await browser.close()

    return rank_matches(match_title, all_results)


async def main():
    user_input = input(
        "Paste a product link (Flipkart/Amazon/Myntra/etc.) OR type a search query: "
    ).strip()

    source_site = None
    source_price = None
    full_title = user_input      # full title, used for scoring matches
    search_query = user_input    # short generic terms, used to search other sites

    if user_input.lower().startswith("http"):
        source_site = detect_site_from_url(user_input)
        if not source_site:
            print("Couldn't recognize that domain as one of the supported sites — "
                  "treating your input as plain text instead.")
        else:
            print(f"Detected source site: {source_site}. Reading product info...")
            async with Stealth().use_async(async_playwright()) as pw:
                browser = await pw.chromium.launch(
                    headless=False,
                    slow_mo=150,
                    args=STEALTH_LAUNCH_ARGS,
                )
                context = await new_stealth_context(browser)
                page = await new_stealth_page(context)
                try:
                    info = await extract_product_info(page, user_input)
                except Exception as e:
                    print(f"Couldn't load that product page: {e}")
                    print("This site may be blocking automated access. "
                          "Try again in a moment, or paste a link from a different site.")
                    await browser.close()
                    return
                finally:
                    await browser.close()

            if not info.get("name"):
                print("Couldn't extract a product name from that link. "
                      "Try pasting a different product page, or type a search query instead.")
                return

            full_title = clean_extracted_title(info["name"], source_site)
            search_query = simplify_query(full_title)
            source_price = parse_price(str(info["price"])) if info.get("price") else None

            print(f"Extracted product: \"{full_title}\"")
            print(f"Searching other sites with simplified terms: \"{search_query}\"")
            if source_price:
                print(f"Price on {source_site}: ₹{source_price:.0f}")

    print(f"\nSearching other sites for: \"{search_query}\" ...\n")
    matches = await compare_price(search_query, match_title=full_title, exclude_site=source_site)

    # Fold the source site's own price back in as a baseline comparison point
    if source_site and source_price:
        matches.append(Product(
            site=source_site, title=full_title, price=source_price,
            url=user_input, match_score=100.0,
        ))
        matches.sort(key=lambda p: p.price)

    if not matches:
        print("No confident matches found elsewhere. Try a more specific query.")
        return

    # --- Terminal table: readable, titles/links truncated to fit ---
    def truncate(text: str, n: int) -> str:
        return text if len(text) <= n else text[: n - 1] + "…"

    table_rows = [
        [
            p.site,
            f"₹{p.price:.0f}",
            f"{p.match_score:.0f}",
            truncate(p.title, 40),
            truncate(p.url, 45),
            "source" if p.site == source_site else "",
        ]
        for p in matches
    ]
    print(f"\nResults for '{full_title}', cheapest first:\n")
    print(tabulate(
        table_rows,
        headers=["Site", "Price", "Match", "Product", "Link", ""],
        tablefmt="fancy_grid",
    ))

    # --- CSV export: full titles and full links, nothing truncated ---
    # Save next to this script file, regardless of what folder the terminal
    # was launched from (VS Code's debugger often runs from a different cwd).
    script_dir = os.path.dirname(os.path.abspath(__file__))
    csv_path = os.path.join(script_dir, "price_comparison_results.csv")
    with open(csv_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        writer.writerow(["Site", "Price (INR)", "Match Score", "Product Name", "Link", "Is Source"])
        for p in matches:
            writer.writerow([p.site, p.price, p.match_score, p.title, p.url,
                              "yes" if p.site == source_site else ""])
    print(f"\nFull results (untruncated links) saved to: {csv_path}")

    best = matches[0]
    print(f"\nBEST OPTION: {best.title} on {best.site} for ₹{best.price:.0f}")
    print(f"Link: {best.url}")


if __name__ == "__main__":
    asyncio.run(main())
