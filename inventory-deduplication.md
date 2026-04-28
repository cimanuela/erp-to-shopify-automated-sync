# Inventory Deduplication & Conflict Resolution

## Problem: What If Product Changes While Sync Runs?

**Scenario:**

00:30 - Alyante exports CSV  
Alyante DB: Product A = 50 pcs, Price €50  
00:35 - Zapier reads CSV  
CSV says: Product A = 50 pcs, Price €50  
00:35-00:40 - Customer buys 1 product  
Shopify now: Product A = 49 pcs  
00:35-00:41 - Zapier processes and updates  
Updates: Product A = 50 pcs (OVERWRITES customer purchase!)  
PROBLEM: Customer's purchase is lost in inventory!

---

## Solution: Conflict Detection

### Method 1: Timestamp Comparison (RECOMMENDED)

**Approach:** Only update if CSV is newer than last update

**Zapier logic:**

IF csv_update_time > shopify_last_updated_time THEN  
→ Apply update  
ELSE  
→ Skip (Shopify was modified more recently)  
→ Log to "Conflict" sheet

**Example:**

CSV data: "Updated at 00:30:15 in Alyante"  
Shopify: "Last updated at 00:35:30 by manual edit"  
00:35:30 > 00:30:15? YES  
→ Alyante is newer, so update  
BUT:  
CSV data: "Updated at 00:30:15"  
Shopify: "Last updated at 00:36:00 by inventory sync"  
00:30:15 < 00:36:00? NO  
→ Shopify is newer, so SKIP (don't overwrite)

---

### Method 2: Quantity Change Detection

**Approach:** If Shopify qty ≠ CSV qty, don't overwrite

**Zapier logic:**

IF shopify_qty == csv_qty THEN  
→ Update all fields normally  
ELSE IF shopify_qty < csv_qty THEN  
→ Qty decreased in Shopify (customer bought?)  
→ Keep Shopify qty (don't overwrite)  
→ Log to "Manual Change" sheet  
→ Alert: "Product X qty changed manually"  
ELSE IF shopify_qty > csv_qty THEN  
→ Qty increased in Shopify (inventory added?)  
→ Keep Shopify qty (don't overwrite)  
→ Log to "Manual Change" sheet  
→ Alert: "Product X qty increased manually"

---

### Method 3: User Notification

**Approach:** If conflict detected, ask human before updating

**Zapier logic:**

IF conflict_detected THEN  
→ Email to admin:  
Subject: "Inventory sync conflict for Product X"  
Body: "CSV says 50, Shopify says 45.  
Manually changed?  
Reply YES to keep Shopify, NO to use CSV"  
→ Wait for response  
→ Update based on response

---

## Recommended: Combination Approach

**Use both Method 1 + Method 2:**

Step 1: Get current Shopify data  
shopify_qty = 45  
shopify_price = €46.50  
shopify_last_updated = "2026-05-01 00:35:30"  
Step 2: Get CSV data  
csv_qty = 50  
csv_price = €46.50  
csv_updated = "2026-05-01 00:30:15"  
Step 3: Check timestamp  
csv_updated (00:30:15) < shopify_last_updated (00:35:30)?  
IF YES (Shopify is newer):  
→ Someone modified Shopify after CSV export  
→ SKIP update (keep Shopify as is)  
→ Log to "Skipped (newer data)" sheet  
IF NO (CSV is newer):  
→ CSV is current  
→ Continue to Step 4  
Step 4: Check quantity conflict  
shopify_qty (45) == csv_qty (50)?  
IF NOT EQUAL:  
→ Qty changed in Shopify since export  
→ SKIP qty update (keep Shopify qty)  
→ DO update price/weight/other fields  
→ Log: "Qty skipped (manual change detected)"  
IF EQUAL:  
→ No conflict  
→ Update all fields normally  
Step 5: Apply updates  
Update in Shopify:  
- Price: €46.50 (from CSV)  
- Qty: 45 (from Shopify, not CSV)  
- Weight: 10.5 (from CSV)  
- etc

---

## Implementation in Zapier

Zapier Step: Conflict Detection  
Input from Step 5 (Search Shopify):  
shopify_product_id: "12345"  
shopify_qty: 45  
shopify_price: "46.50"  
shopify_last_modified: "2026-05-01T00:35:30Z"  
Input from CSV:  
csv_qty: 50  
csv_price: "46.50"  
csv_updated: "2026-05-01T00:30:15Z"  
Logic:  
IF csv_updated < shopify_last_modified:  
timestamp_conflict = true  
ELSE:  
timestamp_conflict = false  
IF shopify_qty != csv_qty:  
qty_conflict = true  
ELSE:  
qty_conflict = false  
Output:  
{  
"timestamp_conflict": false,  
"qty_conflict": true,  
"update_qty": false,  
"update_other_fields": true,  
"conflict_reason": "Qty modified in Shopify after CSV export"  
}  
Zapier Step: Apply Conditional Update  
IF update_qty == true:  
→ Include qty in update payload  
ELSE:  
→ Exclude qty from update payload (keep existing)  
IF update_other_fields == true:  
→ Include price, weight, etc in update payload  
ELSE:  
→ Skip all updates (don't modify)  
Result: Only fields without conflict are updated

---

## Log to Google Sheets

Every sync run, log conflicts:

Sync Log - Conflicts Sheet  
Date | SKU | Product | Conflict Type | Action Taken | Notes  
2026-05-01 | 12345 | Happy Cat | qty_mismatch | qty_skipped | Shopify qty=45, CSV qty=50. Kept Shopify qty.  
2026-05-01 | 12346 | Dog Food | timestamp | update_skipped | Shopify modified at 00:35:30, CSV at 00:30:15. Kept Shopify.

---

## Manual Resolution Process

**If conflict detected:**

1. Admin receives email alert
2. Admin checks Google Sheets for conflict log
3. Admin manually compares:
   - What's in Alyante (source of truth)?
   - What's in Shopify (current)?
4. Decide: Keep Shopify or use Alyante?
5. If Alyante is right: manually update Shopify (and Zapier will respect it next sync)
6. If Shopify is right: manually update Alyante (CSV will sync it next day)

---

## Best Practice: Avoid Conflicts

**Recommendation:**

- Don't manually edit inventory in Shopify during sync window (00:35 - 00:50)
- All inventory changes should flow through Alyante → Zapier → Shopify
- If need to make emergency edit in Shopify: do it AFTER 01:00 (well after sync ends)
- Keep Alyante as single source of truth
