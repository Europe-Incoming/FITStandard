"""
rebuild_site.py
Scans all PDF brochures, extracts key info, rebuilds all index.html files
and updates packages.json. Run via GitHub Actions on every push.
"""

import os
import re
import json
import fitz  # PyMuPDF

# ── CONFIG ────────────────────────────────────────────────────────────────────

REPO_ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

# Folder → display name + region mapping
FOLDER_CONFIG = {
    "city-break": {
        "title": "City Breaks Packages",
        "breadcrumb": "City Breaks",
        "region": "City Break",
        "type": "city-break",
        "parent": "../"
    },
    "multi-country/italy": {
        "title": "Italy",
        "breadcrumb": "Italy",
        "region": "Italy",
        "type": "region",
        "parent": "../"
    },
    "multi-country/eastern-europe": {
        "title": "Eastern Europe",
        "breadcrumb": "Eastern Europe",
        "region": "Eastern Europe",
        "type": "region",
        "parent": "../"
    },
    "multi-country/france": {
        "title": "France",
        "breadcrumb": "France",
        "region": "France",
        "type": "region",
        "parent": "../"
    },
    "multi-country/scandinavia-iceland": {
        "title": "Scandinavia & Iceland",
        "breadcrumb": "Scandinavia & Iceland",
        "region": "Scandinavia & Iceland",
        "type": "region",
        "parent": "../"
    },
    "multi-country/spain-portugal": {
        "title": "Spain & Portugal",
        "breadcrumb": "Spain & Portugal",
        "region": "Spain & Portugal",
        "type": "region",
        "parent": "../"
    },
    "multi-country/switzerland": {
        "title": "Switzerland",
        "breadcrumb": "Switzerland",
        "region": "Switzerland",
        "type": "region",
        "parent": "../"
    },
    "multi-country/uk-ireland": {
        "title": "UK & Ireland",
        "breadcrumb": "UK & Ireland",
        "region": "UK & Ireland",
        "type": "region",
        "parent": "../"
    },
    "multi-country/western-central-europe": {
        "title": "Western & Central Europe",
        "breadcrumb": "Western & Central Europe",
        "region": "Western & Central Europe",
        "type": "region",
        "parent": "../"
    },
}

GEO_BLOCK_SCRIPT = """<script>
(async function() {
    try {
        const response = await fetch('https://api.country.is/');
        const data = await response.json();
        const blockedCountries = ['US', 'CA', 'AU', 'NZ'];
        if (blockedCountries.includes(data.country)) {
            document.body.innerHTML = `
                <div style="display:flex;flex-direction:column;align-items:center;justify-content:center;
                height:100vh;font-family:-apple-system,sans-serif;background:#f5f5f5;text-align:center;padding:20px;">
                    <h1 style="font-size:48px;margin-bottom:16px;">🌍</h1>
                    <h2 style="font-size:32px;font-weight:600;margin-bottom:12px;">Service Not Available</h2>
                    <p style="font-size:18px;color:#757575;">This site is not available in your region.</p>
                </div>`;
        }
    } catch (e) { console.log('Geolocation check failed, allowing access'); }
})();
</script>"""

GA_SCRIPT = """<script async src="https://www.googletagmanager.com/gtag/js?id=G-04BZKH6574"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-04BZKH6574');
</script>"""

NAV_TEMPLATE = """    <nav class="top-nav">
        <div class="nav-container">
            <a href="{logo_href}">
                <img src="{logo_src}" alt="Europe Incoming" class="logo">
            </a>
            <input type="text" class="search-box" placeholder="Search packages - type city or country" id="searchBox">
            <div class="header-right">
                <div class="site-title-group">
                    <div class="site-title-main">Europe Incoming</div>
                    <div class="site-title-sub">FIT Packages</div>
                </div>
                <div class="contact-info">
                    <div class="contact-prompt">Can't find what you're looking for? Email us at:</div>
                    <a href="mailto:fitsales@europeincoming.com" class="contact-email">fitsales@europeincoming.com</a>
                </div>
            </div>
        </div>
    </nav>"""

CSS = """        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Arial, sans-serif;
            background: #f5f5f5; color: #212121; line-height: 1.6; padding-top: 80px;
        }
        .top-nav {
            position: fixed; top: 0; left: 0; right: 0;
            background: white; box-shadow: 0 1px 3px rgba(0,0,0,0.08); z-index: 1000; padding: 16px 0;
        }
        .nav-container {
            max-width: 1200px; margin: 0 auto; padding: 0 24px;
            display: flex; align-items: center; gap: 32px;
        }
        .logo { height: 48px; width: auto; }
        .logo:hover { opacity: 0.8; }
        .header-right { display: flex; align-items: center; gap: 32px; flex: 1; justify-content: flex-end; }
        .site-title-group { text-align: right; }
        .site-title-main { font-size: 1.1em; font-weight: 600; color: #212121; line-height: 1.3; }
        .site-title-sub { font-size: 0.9em; font-weight: 400; color: #757575; }
        .contact-info { text-align: right; padding-left: 32px; border-left: 1px solid #e0e0e0; }
        .contact-prompt { font-size: 0.85em; color: #757575; margin-bottom: 4px; }
        .contact-email { font-size: 0.9em; color: #2196F3; text-decoration: none; font-weight: 500; }
        .contact-email:hover { text-decoration: underline; }
        .search-box {
            width: 100%; max-width: 350px; padding: 10px 18px; font-size: 0.95em;
            border: 1px solid #e0e0e0; border-radius: 24px; transition: all 0.2s; background: #fafafa;
        }
        .search-box::placeholder { text-align: center; }
        .search-box:focus {
            outline: none; border-color: #2196F3; background: white;
            box-shadow: 0 2px 8px rgba(33,150,243,0.15);
        }
        .breadcrumb {
            max-width: 1200px; margin: 0 auto; padding: 24px 24px 0;
            font-size: 0.9em; color: #757575;
        }
        .breadcrumb a { color: #2196F3; text-decoration: none; }
        .breadcrumb a:hover { text-decoration: underline; }
        .container { max-width: 1200px; margin: 0 auto; padding: 32px 24px 48px; }
        h1 { font-size: 2.2em; font-weight: 600; color: #212121; margin-bottom: 40px; letter-spacing: -0.5px; }
        .brochures { display: grid; grid-template-columns: repeat(2, 1fr); gap: 20px; }
        .brochure-card {
            background: white; padding: 28px; border-radius: 12px;
            box-shadow: 0 1px 3px rgba(0,0,0,0.08);
            transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
            text-decoration: none; color: inherit; display: block;
            border: 1px solid #f5f5f5;
        }
        .brochure-card:hover {
            transform: translateY(-4px); box-shadow: 0 8px 24px rgba(0,0,0,0.12);
            border-color: #e0e0e0;
        }
        .brochure-card h3 { font-size: 1.15em; color: #212121; margin-bottom: 10px; font-weight: 600; }
        .tour-type {
            color: #c62828; font-size: 0.8em; font-weight: 700;
            text-transform: uppercase; letter-spacing: 0.8px; margin-bottom: 12px;
        }
        .card-meta { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 10px; }
        .meta-tag {
            background: #f0f4ff; color: #1565c0; font-size: 0.78em;
            padding: 3px 10px; border-radius: 12px; font-weight: 500;
        }
        .meta-tag.duration { background: #f3f3f3; color: #555; }
        .meta-tag.stars { background: #fff8e1; color: #f57f17; }
        .cities-list { font-size: 0.85em; color: #757575; margin-bottom: 10px; }
        .price-tag { font-size: 0.9em; color: #2e7d32; font-weight: 600; margin-top: 8px; }
        .pdf-icon {
            color: #d32f2f; font-size: 0.78em; font-weight: 700;
            text-transform: uppercase; letter-spacing: 0.5px; float: right;
        }
        footer {
            text-align: center; margin-top: 60px; padding: 32px 0;
            color: #9e9e9e; font-size: 0.9em; border-top: 1px solid #f0f0f0;
        }
        @media (max-width: 768px) {
            body { padding-top: 180px; }
            .nav-container { flex-wrap: wrap; gap: 16px; }
            .header-right { width: 100%; flex-direction: column; align-items: center; order: 3; gap: 16px; }
            .site-title-group { text-align: center; }
            .contact-info { text-align: center; border-left: none; border-top: 1px solid #e0e0e0; padding-left: 0; padding-top: 16px; }
            .search-box { max-width: 100%; }
            .brochures { grid-template-columns: 1fr; }
        }"""


# ── PDF EXTRACTION ─────────────────────────────────────────────────────────────

def extract_pdf_data(pdf_path, filename):
    """Extract key info from a brochure PDF using PyMuPDF."""
    result = {
        "duration": None,
        "tour_type": None,
        "cities": [],
        "price_twin": None,
        "stars": None,
        "includes": []
    }

    # Duration from filename
    name = filename.replace('_', ' ')
    dur = re.search(r'(\d+)\s*nights?\s*/?\s*(\d+)\s*days?', name, re.IGNORECASE)
    if dur:
        result["duration"] = f"{dur.group(1)} nights / {dur.group(2)} days"
    else:
        days = re.search(r'(\d+)\s*days?', name, re.IGNORECASE)
        if days:
            result["duration"] = f"{days.group(1)} days"

    # Tour type from filename
    tour = re.search(r'(Self.?[Dd]rive|Private|Regular)', name)
    if tour:
        result["tour_type"] = tour.group(1).replace('-', ' ').title()

    try:
        doc = fitz.open(pdf_path)
        full_text = "\n".join(page.get_text() for page in doc)
        lines = [l.strip() for l in full_text.split('\n')]

        # Cities from "Overnight in X"
        overnight = re.findall(r'Overnight in ([A-Z][a-zA-Z\s]+?)[\.\n,]', full_text)
        result["cities"] = list(dict.fromkeys([c.strip() for c in overnight]))[:6]

        # Star rating
        stars = re.search(r'(\d)\s*Star', full_text)
        if stars:
            result["stars"] = int(stars.group(1))

        # Twin price
        twin_idx = next((i for i, l in enumerate(lines) if 'Twin' in l and 'Do' in l), None)
        if twin_idx:
            euro_prices = []
            for l in lines[twin_idx:twin_idx + 15]:
                m = re.match(r'€\s*([\d,]+)', l)
                if m:
                    euro_prices.append(int(m.group(1).replace(',', '')))
            if len(euro_prices) >= 2:
                twins = euro_prices[1::3] if len(euro_prices) >= 3 else [euro_prices[1]]
                result["price_twin"] = min(twins)

        # Includes
        inc_m = re.search(r'price includes:(.*?)(?:Sample Tours|Terms|Sample Hotels)', full_text, re.DOTALL | re.IGNORECASE)
        if inc_m:
            inc_lines = [l.strip().lstrip('•').strip() for l in inc_m.group(1).split('\n')
                         if l.strip() and not l.strip().startswith('**') and len(l.strip()) > 5]
            result["includes"] = inc_lines[:3]

    except Exception as e:
        print(f"  ⚠ Could not read {filename}: {e}")

    return result


# ── CARD HTML ──────────────────────────────────────────────────────────────────

def make_card(pdf_filename, pdf_data, existing_pkg=None):
    """Generate a brochure card HTML block."""
    name = pdf_data.get("name") or (existing_pkg or {}).get("name") or clean_name(pdf_filename)
    tour_type = pdf_data.get("tour_type") or (existing_pkg or {}).get("type", "")
    duration = pdf_data.get("duration") or (existing_pkg or {}).get("duration", "")
    cities = pdf_data.get("cities") or (existing_pkg or {}).get("cities", [])
    price = pdf_data.get("price_twin")
    stars = pdf_data.get("stars")

    tour_html = f'<div class="tour-type">{tour_type}</div>' if tour_type else ''

    meta_tags = ""
    if duration:
        meta_tags += f'<span class="meta-tag duration">🕐 {duration}</span>'
    if stars:
        meta_tags += f'<span class="meta-tag stars">{"⭐" * stars} {stars} Star</span>'
    meta_html = f'<div class="card-meta">{meta_tags}</div>' if meta_tags else ''

    cities_html = ""
    if cities:
        cities_html = f'<div class="cities-list">📍 {" · ".join(cities)}</div>'

    price_html = ""
    if price:
        price_html = f'<div class="price-tag">From €{price:,} pp (twin)</div>'

    return f'''    <a href="{pdf_filename}" class="brochure-card" target="_blank">
        <span class="pdf-icon">PDF</span>
        <h3>{name}</h3>
        {tour_html}
        {meta_html}
        {cities_html}
        {price_html}
    </a>'''


def clean_name(filename):
    """Convert filename to display name."""
    name = filename.replace('.pdf', '').replace('_', ' ')
    name = re.sub(r'\d{4}-\d{2,4}', '', name)
    name = re.sub(r'Europe Incoming', '', name, flags=re.IGNORECASE)
    name = re.sub(r'\s+', ' ', name).strip()
    return name


# ── INDEX HTML BUILDERS ────────────────────────────────────────────────────────

def build_subpage_html(title, breadcrumb_html, cards_html, logo_src, logo_href, search_js_src, parent):
    nav = NAV_TEMPLATE.format(logo_href=logo_href, logo_src=logo_src)
    return f"""<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{title}</title>
    <style>
{CSS}
    </style>
{GA_SCRIPT}
</head>
<body>
{GEO_BLOCK_SCRIPT}
{nav}
    <div class="breadcrumb">{breadcrumb_html}</div>
    <div class="container">
        <h1>{title}</h1>
        <div class="brochures" id="brochuresList">
{cards_html}
        </div>
        <footer><p>All packages are available for download in PDF format</p></footer>
    </div>
    <script src="{search_js_src}"></script>
</body>
</html>"""


# ── PACKAGES.JSON UPDATE ───────────────────────────────────────────────────────

def update_packages_json(packages_path, all_found):
    """
    Merge found PDFs with existing packages.json.
    - New PDFs: add with extracted data
    - Existing PDFs: preserve existing entry (keep manually added tags/cities)
    - Removed PDFs: delete from json
    """
    existing = {}
    if os.path.exists(packages_path):
        with open(packages_path, 'r') as f:
            data = json.load(f)
        for pkg in data.get("packages", []):
            key = pkg.get("folder", "") + "/" + pkg.get("filename", "")
            existing[key] = pkg

    new_packages = []
    for item in all_found:
        key = item["folder"] + "/" + item["filename"]
        if key in existing:
            # Keep existing entry — user may have added tags/cities manually
            new_packages.append(existing[key])
        else:
            # New PDF — create entry from extracted data
            pkg_id = re.sub(r'[^a-z0-9]', '-', item["filename"].lower().replace('.pdf', ''))[:30]
            new_packages.append({
                "id": pkg_id,
                "name": item["name"],
                "filename": item["filename"],
                "region": item["region"],
                "folder": item["folder"],
                "cities": item["pdf_data"].get("cities", []),
                "duration": item["pdf_data"].get("duration", ""),
                "type": item["pdf_data"].get("tour_type", ""),
                "tags": item["pdf_data"].get("cities", [])
            })

    with open(packages_path, 'w') as f:
        json.dump({"packages": new_packages}, f, indent=2)

    print(f"  ✓ packages.json updated: {len(new_packages)} packages")


# ── MAIN ──────────────────────────────────────────────────────────────────────

def main():
    packages_path = os.path.join(REPO_ROOT, "packages.json")
    all_found = []

    for folder_rel, config in FOLDER_CONFIG.items():
        folder_abs = os.path.join(REPO_ROOT, folder_rel)
        if not os.path.isdir(folder_abs):
            print(f"⚠ Folder not found, skipping: {folder_rel}")
            continue

        pdfs = sorted([f for f in os.listdir(folder_abs) if f.lower().endswith('.pdf')])
        if not pdfs:
            print(f"  (no PDFs in {folder_rel})")
            continue

        print(f"\n📁 {folder_rel} — {len(pdfs)} PDFs")

        # Depth determines relative paths
        depth = folder_rel.count('/') + 1
        logo_src = "../" * depth + "logo.png"
        logo_href = "../" * depth
        search_js = "../" * depth + "global-search.js"

        # Build breadcrumb
        if depth == 1:
            breadcrumb = f'<a href="../">Home</a> > {config["breadcrumb"]}'
        else:
            breadcrumb = f'<a href="../../">Home</a> > <a href="../">Multi-Country</a> > {config["breadcrumb"]}'

        cards = []
        for pdf in pdfs:
            pdf_abs = os.path.join(folder_abs, pdf)
            print(f"  → {pdf}")
            pdf_data = extract_pdf_data(pdf_abs, pdf)
            name = clean_name(pdf)

            all_found.append({
                "filename": pdf,
                "name": name,
                "folder": folder_rel,
                "region": config["region"],
                "pdf_data": pdf_data
            })

            cards.append(make_card(pdf, pdf_data))

        cards_html = "\n".join(cards)
        html = build_subpage_html(
            title=config["title"],
            breadcrumb_html=breadcrumb,
            cards_html=cards_html,
            logo_src=logo_src,
            logo_href=logo_href,
            search_js_src=search_js,
            parent=config["parent"]
        )

        index_path = os.path.join(folder_abs, "index.html")
        with open(index_path, 'w', encoding='utf-8') as f:
            f.write(html)
        print(f"  ✓ Rebuilt {folder_rel}/index.html")

    # Update packages.json
    print(f"\n📦 Updating packages.json...")
    update_packages_json(packages_path, all_found)
    print("\n✅ Done!")


if __name__ == "__main__":
    main()
