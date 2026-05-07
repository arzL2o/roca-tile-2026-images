# Roca Tile e-commerce UI — data brief

You are building a product browse / detail UI for a kitchen & bath remodeling business that resells **Roca Tile** products. The product catalog has already been extracted from the Roca Tile Book 2026 PDF into a CSV; all images are hosted publicly on GitHub. This brief tells you exactly what's in the data, the conventions used, and the dimensions you'll want to expose in the UI.

## 1. Where the data lives

- **CSV:** `https://raw.githubusercontent.com/arzL2o/roca-tile-2026-images/main/ROCA_Products_2026.csv`
- **Image host (GitHub raw):** `https://raw.githubusercontent.com/arzL2o/roca-tile-2026-images/main/`
  - `lifestyle/<slug>__lifestyle__<n>.jpg` — installation/room photos (kitchens, bathrooms, offices). Show these as the hero / "in‑situ" image on a product page.
  - `swatches/<slug>__<variant‑slug>__<n>.png` — color/finish swatches. White background has been removed so they composite cleanly on any UI surface (use them with `object-fit: contain` against a neutral page background).

There are **108 products**, **361 named color/finish variants**, **728 swatch images**, and **156 lifestyle images**.

## 2. CSV columns (12 total)

| Column | Description |
|---|---|
| `Product Name` | The shopper-facing product name (e.g. "Bianco Reale", "Color Collection", "Pavers"). |
| `Category` | One of `XL Slabs`, `Slabs`, or empty. The two filled values are *category* groupings used by Roca for large-format slab products; empty means the product stands on its own. |
| `Material Type` | Construction/finish family (e.g. `Glazed Porcelain`, `Ceramic Wall`, `Polished Porcelain`, `Color Body Porcelain`, `Glazed Porcelain Mosaics`, `Quarry Tile`, `Natural Stone & Glass Mosaics`). 15 distinct values. |
| `Style` | Visual look (e.g. `MARBLES`, `STONES`, `CONCRETES`, `WOODS`, `HANDMADE`, `MOSAICS`, `INDUSTRIAL`, `SOLID COLORS`, `PATTERNED & DECO TILES`, `3D - TEXTURES`, plus combo `MARBLES - STONES - CONCRETES` for slab products). 10 distinct values. |
| `Description` | 1–3 sentence marketing description from the catalog. May be empty for sub-products of XL Slabs / Slabs that share the section description. |
| `Available Sizes` | `;`-separated list of sizes in inches. Each entry can include a finish suffix — see §3 below. Example: `63"x126" R; 63"x63" R`. |
| `Color Variants Count` | Integer. How many named color/finish variants this product has. `0` means the product page didn't expose individually selectable variants in the catalog. |
| `Color Variants` | ` \| `-separated names of the color variants (e.g. `Opal \| Sandstone \| Copper \| Burgundy \| Emerald \| Sapphire`). |
| `Color Variants Detailed` | Same names, but each is followed by `[<comma-separated swatch URLs for that variant>]`. Use this column when you need the variant→image mapping; it's the canonical join. |
| `Swatch Image URLs` | All swatch URLs for the product, `;`-separated, in the same order as variants. Useful for a quick "all swatches" gallery. |
| `Lifestyle / Room Image URLs` | All installation photos for the product, `;`-separated. There is at least one per product. |
| `Price` | Empty in this dataset — the catalog does not publish prices. Plug your own pricing layer in (e.g. per‑sqft × tile area). |

## 3. Conventions you must understand

### Size notation
Sizes are written with imperial double-quote inches and an optional one- or two-letter finish suffix:

```
63"x126" R       → 63 × 126 inches, Rectified edge
12"x24" Hexagon  → 12 × 12 inch hexagonal tile (the second number is the bounding box)
4 1/4"x10"       → fractional, no finish suffix
```

Within a single product, multiple sizes appear separated by `;`. **Treat each "size + finish" combo as a SKU-level option.** Don't dedupe on size alone.

### Finish suffixes
| Code | Meaning |
|---|---|
| `R`     | Rectified edges (clean, minimal grout joint, modern look) |
| `PO`    | Polished |
| `MT`    | Matte |
| `ST`    | Structured / textured |
| `UP`    | Unpolished / honed |
| `ABS`   | Anti‑slip (typically for floor / wet area) |
| `Hexagon`, `Mesh` | Mosaic shape descriptors (not a finish per se — the *tile geometry*) |

A size string like `63"x126" R PO` means rectified, polished. `63"x126" R MT` is the matte sibling. `63"x126" R PO & ST` means available in either polished or structured at that size.

### Variant naming
Variant names always pair the **collection** with a **color or look modifier**:
- `Bianco Reale` (under category `XL Slabs`) — Bianco Reale is itself the variant; one slab look.
- `Riviere Beige`, `Riviere Gris`, `Riviere Cacao` — three colorways of Riviere.
- `Opal`, `Sandstone`, `Copper` — pure color names within the `Amber` collection.
- `Snow White Bright`, `Snow White Matte`, `Tender Gray Bright`, `Tender Gray Matte` — color × finish, where `Bright` and `Matte` are finish modifiers in the Color Collection naming.

Don't try to parse the variant name into "color" + "finish" automatically — Roca isn't consistent enough. Treat each variant as an opaque label with one or more swatch images attached.

### Slugs in image URLs
URL slugs are lowercase-kebab-case of the product name + `__` + variant name. Examples:

```
swatches/bianco-reale__bianco-reale__1.png
swatches/amber__copper__1.png
swatches/color-collection__snow-white-bright__1.png
lifestyle/pavers__lifestyle__1.jpg
```

You can derive a slug from `Product Name` and a variant name by:
```
slug(s) = re.sub(r'[^a-z0-9]+', '-', s.lower()).strip('-')
```

### Catalog → e‑commerce mental model
| Catalog level | E‑commerce mapping |
|---|---|
| Section (XL Slabs, Wall Collections, Mosaics, Wall & Floor) | Top-level category nav |
| `Style` (MARBLES, WOODS, …) | Visual filter / "shop the look" |
| `Material Type` | Technical filter |
| Product (a row in the CSV) | Product page |
| Color variant | Selectable option on the product page |
| Size + finish | SKU |

## 4. Selection axes the UI should expose

These are the dimensions a shopper actually filters on. List them here with the dominant values so you can build the facets without re-querying the CSV.

1. **Style** (10 values — the primary "look" filter):
   - MARBLES (16), MARBLES - STONES - CONCRETES (33 slab products), STONES (11), CONCRETES, WOODS (9), HANDMADE (14), MOSAICS (5), INDUSTRIAL (9), SOLID COLORS (3), 3D - TEXTURES (6), PATTERNED & DECO TILES (2).

2. **Material Type** (15 values — the "what is it made of" filter):
   - Glazed Porcelain (31), Ceramic Wall (24), Matte & Polished Porcelain (21), Matte Porcelain (8), Polished Porcelain (7), Color Body Porcelain (4), and various mosaic/specialty types.

3. **Category** (`XL Slabs` / `Slabs` / general). Useful as a top-level toggle: large‑format slabs vs. everything else.

4. **Size class**, derived from `Available Sizes`. Suggested buckets:
   - **XL slabs**: any size where either dimension ≥ 48"
   - **Large format**: 24"–47"
   - **Standard**: 8"–23"
   - **Mosaic / small**: < 8" or contains "Hexagon"/"Mesh"

5. **Finish**, derived from size suffixes: `Polished`, `Matte`, `Rectified`, `Structured`, `Honed`, `Anti‑slip`. A product may belong to multiple.

6. **Application**, inferable from `Material Type`:
   - "Wall only" → `Ceramic Wall`, `Ceramic Wall & Gres Stoneware`
   - "Floor or wall" → all porcelain types
   - "Mosaic accent" → anything with `Mosaics` in the type
   - "Heavy traffic / outdoor" → `Color Body Porcelain`, `Quarry Tile`, products with `ABS` finish

7. **Color family**, inferred from variant name keywords (white, gray, beige, blue, green, etc.). Optional but high-value for shoppers who search by color.

## 5. Data quality notes (so you don't trip on them)

- **5 products have no swatch images** (`CC Mosaics`, `CC Mosaics +`, `CC Porcelain`, `Decorative Accents`, `Rockart`) — their per-tile imagery in the catalog is baked into composite mosaic photos. Those photos *are* present, classified as lifestyle images. Treat these products as "lifestyle‑only" in the UI: skip the swatch picker, show the room shots.
- **Some swatches are very pale** (e.g. Amber → Opal, Color Collection → Snow White Bright). The white background was removed, but the tile color itself is near‑white. Always render swatches against a soft gray (e.g. `#F2F2F4`) so light tiles are visible.
- **Description is empty for sub-products of XL Slabs / Slabs** (Bianco Reale, Gran Calacata, Nuage, …). Inherit the parent category's description on the UI side, or write a short fallback like *"From the XL Slabs collection — large-format porcelain slabs inspired by natural marble."*
- **`Price` is empty for everything.** Don't render a `$0` placeholder; either hide the price field or show "Request a quote" until a pricing source is wired up.
- **Lifestyle images are shared** across multiple products on the same catalog spread (e.g. all Riviere colorways share a bathroom photo). Don't be surprised when the same URL appears on several products — that's intentional, not a duplicate.

## 6. UI design suggestions

This is a *visual* category. The home page and category pages should be image-led, not list-led.

### Home / landing
- Hero with one rotating lifestyle photo (pick from the 156 we have — large XL Slabs kitchens look the most premium).
- A "Shop by style" grid: 10 tiles, one per `Style`, each backed by a representative lifestyle image.
- A "Shop by room" section: Kitchen / Bathroom / Office / Living — these aren't tagged in the CSV, so you'll need a curator step (or use the lifestyle filename slug heuristically: collections like `panama`, `serena`, `nordico` skew bathroom; `xl-slabs/*`, `lithology` skew kitchen/living).

### Category / browse page
- Left rail facets: Style → Material Type → Size class → Finish → Application → Color family.
- Card grid: each card shows the **first lifestyle image**, the product name, the count of color variants ("6 colors"), and a small inline strip of swatch dots.
- Sort: featured, A‑Z, "most colors", "largest format".

### Product detail page
- Big lifestyle image carousel (use *all* `Lifestyle / Room Image URLs` for that product).
- Color/finish picker showing every variant as a swatch tile. Selecting one just swaps the inline swatch preview (since variants here typically share the same lifestyle imagery — the catalog photographs one colorway and trusts the swatch to convey the others).
- Size selector built from `Available Sizes`. Each entry gets a chip with the size and a small finish badge (R / PO / MT / ST / UP / ABS).
- Specs panel: Material Type, Style, Category (if filled), all finishes available, plain‑text size list.
- Description, then a "Request a quote" CTA (since `Price` is empty).

### Search
- Full-text over `Product Name`, `Color Variants`, `Description`.
- Synonyms: `marble` ↔ Style `MARBLES`; `wood look`, `oak` ↔ Style `WOODS`; `subway`, `handmade look` ↔ Style `HANDMADE`; `concrete look` ↔ Style `CONCRETES`.

### Image rendering
- Swatches: serve as PNG with transparency, render against `#EFEFF1` in a card with subtle inset shadow so the tile texture pops.
- Lifestyle: serve as JPG, lazy‑load, use a 16:9 or 3:2 crop in cards. The originals are wider (≈1500×750), so they downscale fine.

## 7. Quick code starters

Parse the CSV (Python):
```python
import csv, re
def slug(s): return re.sub(r"[^a-z0-9]+", "-", s.lower()).strip("-")

products = []
with open("ROCA_Products_2026.csv") as f:
    for row in csv.DictReader(f):
        sizes = [s.strip() for s in row["Available Sizes"].split(";") if s.strip()]
        variants = [v.strip() for v in row["Color Variants"].split("|") if v.strip()]
        swatches = [u.strip() for u in row["Swatch Image URLs"].split(";") if u.strip()]
        lifestyles = [u.strip() for u in row["Lifestyle / Room Image URLs"].split(";") if u.strip()]
        products.append({**row, "_sizes": sizes, "_variants": variants,
                         "_swatches": swatches, "_lifestyles": lifestyles})
```

Detected size class:
```python
def size_class(size_strs):
    nums = []
    for s in size_strs:
        for n in re.findall(r"\d+(?:\s+\d+/\d+)?", s):
            nums.append(int(re.split(r"\s+", n)[0]))
    if not nums: return "unknown"
    m = max(nums)
    if m >= 48: return "xl-slab"
    if m >= 24: return "large-format"
    if m >= 8:  return "standard"
    return "mosaic"
```

---

**Bottom line for the implementer:** the CSV is the source of truth. The UI's job is to expose 6–7 facets (Style, Material, Size class, Finish, Application, Color family, Category) over 108 visual products, with rich lifestyle photography on every product page. The data is consistent enough to drive a faceted browse without any further cleanup; the only fixups needed are inheriting descriptions for XL Slabs / Slabs sub‑products and hiding the empty Price column until pricing is wired up.
