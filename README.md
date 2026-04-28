# ERP-to-Shopify Automated Sync

**Real-world integration** - Daily automated inventory sync from TeamSystem Alyante to Shopify.

## Overview

Automated workflow that:
1. Exports daily CSV from TeamSystem Alyante ERP (00:30)
2. Reads file from server (SFTP/HTTP)
3. Filters only "IN USO", "IN ESAURIMENTO", "ESAURITO" status
4. Skips "CODIFICATO" (dirty data)
5. Maps ERP fields to Shopify product fields
6. Updates products if SKU exists
7. Logs all changes to Google Sheets

- **Status:** 🔄 Design Phase (Ready to build May 2026)
- **Tech Stack:** Zapier, TeamSystem Alyante (CSV), Shopify API, Google Sheets
- **Trigger:** Scheduled daily (00:35 - 5 min after Alyante export)
- **Frequency:** Once per 24 hours
- **Products Affected:** ~500 SKUs (only IN USO + IN ESAURIMENTO + ESAURITO)
- **Expected Accuracy:** 99.8% (CSV parsing + validation)

## The Problem (Before)

**Manual inventory sync:**
- Time spent: 2 hours per week
- Process: Download CSV from Alyante → manually check SKU → manually update Shopify
- Error rate: ~5% (forgotten updates, wrong prices)
- Visibility: No audit trail (who updated what when?)
- Workflow: Reactive (discover discrepancies after complaints)

**Result:** Price inconsistencies, wrong inventory, missed reorders, unhappy customers

## The Solution (After)

**Automated daily sync:**
- Time spent: 0 minutes (fully automated)
- Process: Alyante → CSV → Zapier → Shopify (automatic)
- Error rate: 0.2% (only system failures)
- Visibility: Complete audit trail in Google Sheets
- Workflow: Proactive (sync happens before anyone notices issues)

**Result:** Consistent data, accurate inventory, better customer experience

## Architecture

ALYANTE ERP (Source of Truth)  
↓ (export at 00:30)  
CSV FILE on Server  
↓ (read at 00:35)  
ZAPIER SCHEDULE TRIGGER  
↓  
PARSE CSV  
↓  
FILTER: Only status IN USO, IN ESAURIMENTO, ESAURITO  
Skip: CODIFICATO (dirty data)  
↓  
FOR EACH PRODUCT:  
├─ Get SKU from CSV  
├─ Search Shopify for product with this SKU  
├─ IF FOUND:  
│  ├─ Update: Prezzo (Listino IVA) → Shopify "Prezzo"  
│  ├─ Update: Costo → Shopify "Costo"  
│  ├─ Update: EAN → Shopify "Codice a barre"  
│  ├─ Update: Qty → Shopify "Disponibile"  
│  ├─ Update: Peso → Shopify "Peso del prodotto"  
│  └─ Update: Status → Shopify custom field "Status Alyante"  
├─ IF NOT FOUND:  
│  └─ Skip (product not yet created in Shopify)  
↓  
LOG ALL CHANGES:  
├─ Timestamp  
├─ SKU  
├─ Fields updated  
├─ Old value → New value  
└─ Status (SUCCESS / ERROR / SKIPPED)

## Data Mapping (CSV → Shopify)

| Alyante Column | Shopify Field | Type | Update Rule |
|---|---|---|---|
| CODICE ARTICOLO | Product SKU | text | NO (skip product) |
| NOME ARTICOLO | Product Title | text | NO (keep manual) |
| PESO | Peso del prodotto | number | YES (overwrite daily) |
| PREZZO LISTINO CON IVA | Prezzo | currency | YES (overwrite daily) |
| COSTO | Costo | currency | YES (overwrite daily) |
| EAN ARTICOLO | Codice a barre | text | YES (overwrite daily) |
| QUANTITA' MAGAZZINO | Disponibile | number | YES (overwrite daily) |
| STATUS ARTICOLO | Status Alyante* | dropdown | YES (see below) |

*Custom field in Shopify (must create)

## Status Logic (CRITICAL)

**Alyante Status → Shopify Behavior:**

### 1. CODIFICATO

Action: SKIP ENTIRE PRODUCT
Reason: Dirty data (not ready for sale)
Shopify: Keep as is (don't touch)
Note: Tell Alyante team to clean data

### 2. IN USO

Action: UPDATE all fields, make VISIBLE
Shopify Visibility: Show product
Shopify "Status Alyante" field: "IN USO"
Tag: NONE
Behavior: Normal product (for sale)

### 3. IN ESAURIMENTO

Action: UPDATE all fields, make VISIBLE
Shopify Visibility: Show product
Shopify "Status Alyante" field: "IN ESAURIMENTO"
Tag: LAST_PIECES (or LOW_STOCK)
Behavior: Warning banner "Ultimi pezzi disponibili"
Customer sees: "Quantità limitata"

### 4. ESAURITO

Action: Two options (see "Inventory Strategy" below)  

Option A: Hide product + Tag TEMPORARILY_OUT

- Shopify Visibility: Hide product
- Tag: TEMPORARILY_OUT
- Meaning: Will restock soon

Option B: Show product + Tag PERMANENTLY_OUT

- Shopify Visibility: Show product
- Availability: "Out of Stock"
- Tag: PERMANENTLY_OUT
- Meaning: Won't restock (discontinued)
- Button: "Notify me" instead of "Add to cart"

Use Option A or B based on:

- Is product in Alyante "ESAURITO" temporarily? → Option A (will restock soon)
- Is product discontinued/phased out? → Option B (won't come back)

## Shopify Configuration Required

### 1. Create Custom Field "Status Alyante"

Settings → Custom Fields → Create  
Name: Status Alyante  
Type: Dropdown  
Options:
- IN USO
- IN ESAURIMENTO
- ESAURITO_TEMPORARILY
- ESAURITO_PERMANENTLY
- UNKNOWN

### 2. Create Tags for Filtering

Recommended tags:  

- LAST_PIECES (for IN ESAURIMENTO)
- TEMPORARILY_OUT (for ESAURITO restock soon)
- PERMANENTLY_OUT (for ESAURITO no restock)
- DELISTED (alternative if you prefer)
- SYNC_ERROR (if sync failed for this product)

### 3. Configure Shopify "Availability"

For each product, set:

- Quantity (from Alyante)
- Track inventory: YES
- Deactivate when out of stock: YES (optional)

## Implementation Steps (High Level)

1. **Setup Alyante export**
   - Configure daily CSV export (00:30)
   - File location: /server/exports/alyante-daily.csv
   - Format: UTF-8, semicolon-delimited

2. **Create Shopify custom field**
   - Status Alyante (dropdown)
   - Add to all products

3. **Create tags in Shopify**
   - LAST_PIECES, TEMPORARILY_OUT, PERMANENTLY_OUT, etc

4. **Build Zapier workflow**
   - Trigger: Schedule daily (00:35)
   - Read CSV from server
   - Parse & filter
   - For each row: lookup SKU in Shopify
   - Update fields + tags + visibility
   - Log to Google Sheets

5. **Test & Monitor**
   - Test with 5-10 products first
   - Verify fields update correctly
   - Check tags apply
   - Monitor first week for errors

## Error Handling

**What if:**
- ❌ CSV file missing: Alert + skip run
- ❌ SKU not found in Shopify: Log to "Skipped" sheet, don't error
- ❌ API rate limit: Retry with backoff
- ❌ Bad data in CSV (price = null): Skip field, log warning

## Monitoring & Audit Trail

All syncs logged to Google Sheets:

| Timestamp | SKU | Product | Fields Updated | Old Price | New Price | Status |
|---|---|---|---|---|---|---|
| 2026-05-01 00:35:12 | 12345 | Happy Cat 10kg | Prezzo, Qty, Status | €45.00 | €46.50 | SUCCESS |
| 2026-05-01 00:35:15 | 12346 | Dog Food 5kg | Qty, Status | 50 pcs | 12 pcs | SUCCESS |
| 2026-05-01 00:35:18 | 99999 | Unknown SKU | - | - | - | SKIPPED (not in Shopify) |

## Cost & ROI

- **Development time:** 25-35 hours
- **Shopify custom field:** Free
- **Zapier:** Free tier (if ≤2 steps) or Pro (€25-30/month if complex)
- **Server/CSV hosting:** Depends (your IT)
- **Ongoing maintenance:** 30 min/month

**Time saved:** 2 hours/week × 52 weeks = 104 hours/year
**Value:** ~€2,600/year (at €25/hour) + better customer experience

**ROI:** Breaks even in month 1, positive ROI thereafter

## Challenges & Solutions

| Challenge | Solution |
|---|---|
| Dirty data in Alyante (CODIFICATO products) | Filter them out (skip in Zapier) |
| SKU mismatch (Alyante SKU ≠ Shopify SKU) | Manual SKU mapping in Shopify (do once) |
| Concurrent edits (manual change + auto sync conflict) | Sync is scheduled (00:35), avoid manual edits at that time |
| Product doesn't exist in Shopify yet | Log as SKIPPED, re-run daily until product created |
| Shopify API rate limit | Batch requests, implement backoff |

## Inventory Strategy (IMPORTANT)

**Decision: How to handle ESAURITO status?**

### Strategy A: Always Hide When ESAURITO

Pro: Clean storefront (no dead listings)  
Con: Customers can't find product when restock  
Use: If products restock quickly (< 2 weeks)

### Strategy B: Always Show When ESAURITO

Pro: Customers see product, "Notify me"  
Con: Messy storefront with many "out of stock"  
Use: If products valuable/searchable

### Strategy C: Smart Toggle (RECOMMENDED)

Use custom field "Restock Date" from Alyante  
IF restock_date < TODAY + 30 days:  
→ Show with "Notify me"  
ELSE:  
→ Hide product  
Benefit: Best of both worlds

## Status

🔄 **Design Phase** - Ready to build when Alyante export configured  
Estimated build: May 2026 (20-30 hours)  
Current blocker: Alyante export needs setup

Last updated: April 27, 2026
