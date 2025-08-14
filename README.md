
import os
import time
import json
import logging
import tempfile
from pathlib import Path
from datetime import datetime, timezone, timedelta

import requests
import keepa
import matplotlib.pyplot as plt
import numpy as np
from dotenv import load_dotenv

# ---- Load env ----
load_dotenv()

KEEPA_KEY = os.getenv("KEEPA_KEY")
WEBHOOK_90_100 = os.getenv("DISCORD_WEBHOOK_90_100")
WEBHOOK_80_90 = os.getenv("DISCORD_WEBHOOK_80_90")
WEBHOOK_70_80 = os.getenv("DISCORD_WEBHOOK_70_80")
WEBHOOK_20_70 = os.getenv("DISCORD_WEBHOOK_20_70")
POLL_INTERVAL = int(os.getenv("POLL_INTERVAL", "60"))

if not KEEPA_KEY:
    raise RuntimeError("KEEPA_KEY not set in .env")
if not (WEBHOOK_90_100 and WEBHOOK_80_90 and WEBHOOK_70_80):
    raise RuntimeError("One or more DISCORD_WEBHOOK_* missing in .env")

# ---- Logging ----
logging.basicConfig(level=logging.DEBUG, format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger("keepa-discord")

# ---- Keepa client ----
api = keepa.Keepa(KEEPA_KEY)

# ---- Persistent seen file ----
SEEN_FILE = Path("seen_deals.json")
if SEEN_FILE.exists():
    try:
        seen = json.loads(SEEN_FILE.read_text())
    except Exception:
        seen = {}
else:
    seen = {}

def save_seen():
    SEEN_FILE.write_text(json.dumps(seen, indent=2))

def safe_get(d, *keys, default=None):
    for k in keys:
        if isinstance(d, dict) and k in d and d[k] is not None:
            return d[k]
    return default

def format_price(value):
    if value is None:
        return "N/A"
    try:
        if isinstance(value, (int, np.integer)):
            if value < 0:
                return "N/A"
            return f"{value/100:.2f}"
        elif isinstance(value, float):
            return f"{value:.2f}"
        else:
            return str(value)
    except Exception:
        return str(value)

def plot_product_price_history(product, out_path):
    try:
        data = product.get("data") or {}
        price_keys = ["AMAZON", "NEW", "NEW_f", "NEW", "AMAZON_new"]
        key = None
        for k in price_keys:
            if k in data and data[k] is not None:
                key = k
                break
        time_key = None
        if key:
            if f"{key}_time" in data:
                time_key = f"{key}_time"
            elif f"{key}Time" in data:
                time_key = f"{key}Time"
            else:
                for cand in data:
                    if cand.endswith("_time") and isinstance(data[cand], list):
                        time_key = cand
                        break
        times = None
        prices = None
        if key and time_key and data.get(key) and data.get(time_key):
            prices_raw = data.get(key, [])
            times_raw = data.get(time_key, [])
            times = keepa.keepa_minutes_to_time_array(times_raw)
            prices = np.array(prices_raw, dtype=float) / 100.0
        else:
            try:
                import matplotlib
                matplotlib.use("Agg")
                keepa.plot_product(product, show=False)
                plt.savefig(out_path, bbox_inches="tight")
                plt.close()
                return True
            except Exception as e:
                logger.debug("plot_product fallback failed: %s", e)
                return False

        if times is None or prices is None or len(times) == 0:
            logger.debug("No price series found for plotting")
            return False

        plt.figure(figsize=(8, 3))
        plt.plot(times, prices, linewidth=1.2)
        plt.fill_between(times, prices, alpha=0.12)
        plt.title(safe_get(product, "title", default="Price history"))
        plt.xlabel("Date")
        plt.ylabel("Price (USD)")
        plt.tight_layout()
        plt.gcf().autofmt_xdate()
        plt.savefig(out_path, bbox_inches="tight", dpi=150)
        plt.close()
        return True
    except Exception as exc:
        logger.exception("Error plotting product history: %s", exc)
        return False

def build_embed_for_deal(deal):
    asin = safe_get(deal, "asin")
    title = safe_get(deal, "title") or safe_get(deal, "productTitle") or "No title"
    discount = safe_get(deal, "percentOff", "discountPercent", "discount", default=0)
    current_price = safe_get(deal, "currentPrice", "price", default=None)
    avg_price = safe_get(deal, "avgPrice", "averagePrice", default=None)
    sales = safe_get(deal, "orderCount", "sales", default=None) or safe_get(deal, "sales", default=None)
    order_limit = safe_get(deal, "orderLimit", default=None)

    current_price_str = format_price(current_price)
    avg_price_str = format_price(avg_price)
    discount_str = f"{discount}%"
    sales_str = str(sales) if sales is not None else "N/A"
    order_limit_str = str(order_limit) if order_limit is not None else "N/A"

    embed = {
        "title": f"{title}",
        "url": f"https://www.amazon.com/dp/{asin}" if asin else None,
        "description": f"Current: **${current_price_str}**  •  Average Price: **${avg_price_str}**  •  Discount: **{discount_str}**",
        "color": 0xE67E22,
        "fields": [
            {"name": "ASIN", "value": asin or "N/A", "inline": True},
            {"name": "Sales", "value": sales_str, "inline": True},
            {"name": "Order Limit", "value": order_limit_str, "inline": True},
        ],
        "footer": {"text": "Data from Keepa"},
    }

    image_path = None
    try:
        product_resp = api.query(asin, history=True)
        if isinstance(product_resp, list) and len(product_resp) > 0:
            product_obj = product_resp[0]
            tmpfile = Path(tempfile.gettempdir()) / f"keepa_chart_{asin}.png"
            ok = plot_product_price_history(product_obj, str(tmpfile))
            if ok and tmpfile.exists():
                image_path = str(tmpfile)
            images = safe_get(product_obj, "imagesCSV", default=None)
            if images:
                try:
                    first = images.split(",")[0]
                    thumb_url = f"https://images-na.ssl-images-amazon.com/images/I/{first}.jpg"
                    embed["thumbnail"] = {"url": thumb_url}
                except Exception:
                    pass
    except Exception as e:
        logger.debug("Could not query product for ASIN %s: %s", asin, e)

    return embed, image_path

def post_embed_with_image(webhook_url, embed, image_path=None, username="AmazonBot"):
    payload = {"username": username, "embeds": [embed]}
    try:
        if image_path and Path(image_path).exists():
            filename = Path(image_path).name
            embed["image"] = {"url": f"attachment://{filename}"}
            data = {"payload_json": json.dumps(payload)}
            with open(image_path, "rb") as fh:
                files = {"file": (filename, fh, "image/png")}
                r = requests.post(webhook_url, data=data, files=files, timeout=30)
        else:
            r = requests.post(webhook_url, json=payload, timeout=10)
        if r.status_code not in (200, 204):
            logger.error("Webhook failed (%s): %s", r.status_code, r.text)
            return False
        logger.info("Posted to webhook OK (status %s).", r.status_code)
        return True
    except Exception as e:
        logger.exception("Error posting webhook: %s", e)
        return False

def get_webhook_for_discount(deal):
    """
    Determines which webhook a deal should be sent to based on its discount percentage.
    Always uses the actual discount value from Keepa, even if it is None.
    """
    # Pull discount from the deal
    pct_val = safe_get(deal, "percentOff", "discountPercent", "discount")
    
    if pct_val is None:
        logger.debug("Deal has no discount value, skipping.")
        return None

    try:
        pct_val = int(pct_val)
    except Exception:
        logger.debug("Deal discount value is not an integer, skipping: %s", pct_val)
        return None

    if 90 <= pct_val <= 100:
        return WEBHOOK_90_100
    elif 80 <= pct_val < 90:
        return WEBHOOK_80_90
    elif 70 <= pct_val < 80:
        return WEBHOOK_70_80
    elif 20 <= pct_val < 70:
        return WEBHOOK_20_70
    else:
        # Deals below 20% are ignored
        logger.debug("ASIN %s has discount %s%%, below 20%%, skipping", safe_get(deal, "asin"), pct_val)
        return None




def poll_and_dispatch():
    logger.info("Polling Keepa deals...")

    deal_params = {
        "page": 0,
        "domainId": 1,
        "excludeCategories": [283155, 133140011, 5174, 18145289011, 2625373011],
        "priceTypes": [1],
        "deltaPercentRange": [70, 2147483647],
        "currentRange": [0, 29700],  # 0 to $297
        "isRangeEnabled": True,
        "isFilterEnabled": True,
        "filterErotic": True,
        "singleVariation": True,
        "sortType": 1,
        "dateRange": "0",
        "warehouseConditions": [1, 2, 3, 4, 5],
    }

    try:
        resp = api.deals(deal_params, domain="US")
    except Exception as e:
        logger.error("Error calling Keepa deals(): %s", e)
        return

    deals = resp.get("dr") or resp.get("deals") or []
    logger.info("Keepa returned %d deals (raw).", len(deals))

    now_utc = datetime.now(timezone.utc)



    for d in deals:
        asin = safe_get(d, "asin")
        title = safe_get(d, "title") or safe_get(d, "productTitle") or "No title"

        # --- Query full product for accurate Buy Box and average price ---
        try:
            products_resp = api.query(asin, offers=20, stats=30, history=0)
            if not products_resp:
                logger.debug(f"Skipping ASIN {asin}: no product data returned")
                continue
            product = products_resp[0]
        except Exception as e:
            logger.debug(f"Skipping ASIN {asin}: product query failed ({e})")
            continue

        # --- Buy Box price ---
        product_parsed = product.get("stats_parsed", {})
        buy_box_price = product_parsed.get("buyBoxPrice")
        if buy_box_price not in (None, -1):
            buy_box_price = buy_box_price / 100
        else:
            # fallback to first live offer
            offers = product.get("offers", [])
            buy_box_price = None
            for offer in offers:
                price = offer.get("price")
                if price not in (None, -1):
                    buy_box_price = price / 100
                    break

        if buy_box_price is None:
            logger.debug(f"Skipping ASIN {asin}: no valid buy box or live offer")
            continue


        
        # --- Average price ---
        avg_stats = product_parsed.get("avg", {})
        if isinstance(avg_stats, list):
            logger.debug(f"ASIN {asin}: avg_stats is a list, skipping")
        
            continue


        print("stats",avg_stats)


        avg_price = avg_stats.get("NEW") or avg_stats.get("NEW_FBA") or avg_stats.get("NEW_FBM")
        if avg_price is None:
            logger.debug(f"Skipping ASIN {asin}: no valid average price")
            continue
    


        print("avg price", avg_price)
        # --- Discount calculation ---
        try:
            discount_pct = int(round(((avg_price - buy_box_price) / avg_price) * 100))
        except Exception as e:
            logger.debug(f"Skipping ASIN {asin}: could not calculate discount ({e})")
            continue

        if discount_pct < 20:
            logger.debug(f"ASIN {asin} has discount {discount_pct}%, below 20%, skipping")
            continue

        # Select webhook based on calculated discount, NOT the Keepa deal percentOff
        if 90 <= discount_pct <= 100:
            webhook = WEBHOOK_90_100
        elif 80 <= discount_pct < 90:
            webhook = WEBHOOK_80_90
        elif 70 <= discount_pct < 80:
            webhook = WEBHOOK_70_80
        else:  # 20-70%
            webhook = WEBHOOK_20_70

        # For embed: use buy_box_price as currentPrice, avg_price as avgPrice
        deal_for_embed = {
            "asin": asin,
            "title": title,
            "currentPrice": buy_box_price,
            "avgPrice": avg_price,
            "percentOff": discount_pct,
            "orderCount": safe_get(d, "orderCount") or safe_get(d, "sales"),
            "orderLimit": safe_get(d, "orderLimit"),
        }

        embed, image_path = build_embed_for_deal(deal_for_embed)
        posted = post_embed_with_image(webhook, embed, image_path=image_path, username="AmazonBot")
        if posted:
            seen[asin] = buy_box_price
            save_seen()



def main_loop():
    logger.info(f"Starting Keepa -> Discord relay. Poll interval: {POLL_INTERVAL}s")
    while True:
        try:
            poll_and_dispatch()
        except Exception as exc:
            logger.exception("Unexpected loop error: %s", exc)
        logger.info(f"Sleeping for {POLL_INTERVAL} seconds...")
        time.sleep(POLL_INTERVAL)

if __name__ == "__main__":
    main_loop()
