# Status Logic & Decision Tree

## Overview

This document explains how to handle each Alyante status value.

## Status Values & Shopify Behavior

### STATUS 1: CODIFICATO (Dirty Data)

**Meaning:** Product is coded/planned but NOT ready for sale yet.

**Alyante:** Internal use only, not for customers.

**Zapier Action:** SKIP ENTIRELY

IF status == "CODIFICATO" THEN  
→ Do not sync to Shopify  
→ Do not update fields  
→ Do not create product  
→ Just skip (no error, no log)

**Shopify:** Leave as is (if exists from before)

**Why?** These are test products, prototypes, or WIP. 
Clean up Alyante first, then enable sync.

**You should:** Ask Alyante team to clean CODIFICATO items.

---

### STATUS 2: IN USO (Active / For Sale)

**Meaning:** Product is ready and actively for sale.

**Alyante:** This is what should be on your store.

**Zapier Action:** FULL SYNC + VISIBLE

IF status == "IN USO" THEN  
→ Update all fields:  
• Price  
• Quantity  
• Cost  
• EAN  
• Weight  
→ Set visibility: VISIBLE (show in storefront)  
→ Set tags: NONE  
→ Update custom field: status_alyante = "IN USO"

**Shopify:**

Visibility: Published (show in storefront)  
Tags: (none)  
Availability: Normal (if qty > 0) or "Low stock" (if qty < 10)  
Button: "Add to Cart"

**Customer Sees:** Product available for purchase (normal behavior)

**Example:**

Product: Happy Cat Minkas 10kg  
Status: IN USO  
Qty: 45 pcs  
Price: €46.50  
Shopify: ✓ Visible, "Add to Cart" enabled

---

### STATUS 3: IN ESAURIMENTO (Low Stock / Selling Out)

**Meaning:** Product is still for sale but running low.
Alyante considers this a warning sign.

**Alyante:** Still active, but reorder needed soon.

**Zapier Action:** FULL SYNC + VISIBLE + WARNING TAG

IF status == "IN ESAURIMENTO" THEN  
→ Update all fields (same as IN USO)  
→ Set visibility: VISIBLE (keep showing)  
→ Set tags: ["LAST_PIECES"]  
→ Update custom field: status_alyante = "IN ESAURIMENTO"  
→ (Optional) Add banner: "Ultimi pezzi disponibili"

**Shopify:**

Visibility: Published (show in storefront)  
Tags: LAST_PIECES  
Availability: "Low stock" warning  
Button: "Add to Cart" (still available)  
Banner: "⚠️ Only X pieces left!"

**Customer Sees:** 
- Product is available
- But warning: "Last pieces available"
- Creates urgency → more likely to buy

**Example:**

Product: Dog Food Premium 5kg  
Status: IN ESAURIMENTO  
Qty: 12 pcs  
Price: €21.96  
Shopify: ✓ Visible, "⚠️ Last 12 pieces!" banner

**You should:** Reorder this product from supplier soon.

---

### STATUS 4: ESAURITO (Out of Stock)

**Meaning:** Product qty = 0. No stock available.

**Alyante:** Out of stock.

**Challenge:** Need to distinguish 2 scenarios:

**Scenario A: ESAURITO TEMPORARILY**
- Product will be restocked soon (in < 30 days)
- You plan to sell again
- Example: Seasonal product, incoming shipment

**Scenario B: ESAURITO PERMANENTLY**
- Product discontinued / phased out
- Won't be restocked
- Example: Old model, supplier discontinued

---

## Decision: How to Handle ESAURITO?

### OPTION 1: Always Hide When ESAURITO

**Zapier Action:**

IF status == "ESAURITO" THEN  
→ Update quantity: 0  
→ Set visibility: HIDDEN (remove from storefront)  
→ Set tags: ["TEMPORARILY_OUT"]  
→ Update custom field: status_alyante = "ESAURITO"

**Shopify:**

Visibility: Hidden (not in search/collections)  
Button: "Notify me" (optional)

**Pro:**
- Clean storefront (no dead listings)
- Customer doesn't see unavailable products

**Con:**
- Customer can't find product when restock
- Search engine loses SEO history
- If product restocks, need to un-hide manually

**Best for:** Shops with frequent reorders (< 2 weeks restocks)

---

### OPTION 2: Always Show When ESAURITO

**Zapier Action:**

IF status == "ESAURITO" THEN  
→ Update quantity: 0  
→ Set visibility: VISIBLE (keep in storefront)  
→ Set tags: ["PERMANENTLY_OUT"] (optional)  
→ Update custom field: status_alyante = "ESAURITO"  
→ Availability: "Out of stock" message

**Shopify:**

Visibility: Published (show in storefront)  
Button: "Notify me when available"

**Pro:**
- Customer can still find/view product
- Can pre-order or request notification
- Keep SEO ranking
- Good for high-demand items

**Con:**
- Storefront shows many "out of stock" items
- Might confuse customers
- Harder to distinguish permanent vs temporary

**Best for:** Shops with high-value products (customers want options)

---

### OPTION 3: Smart Toggle (RECOMMENDED) ⭐⭐⭐

**Best approach:** Use custom field in Alyante to decide.

**Setup:** Add custom field to Alyante
- Field: "Restock Date"
- Type: Date
- Meaning: When is product expected back in stock?

**Zapier Logic:**

IF status == "ESAURITO" THEN  
IF restock_date != null AND restock_date <= TODAY + 30 days THEN  
→ Visibility: VISIBLE  
→ Tags: ["TEMPORARILY_OUT"]  
→ Message: "Back in stock on {{restock_date}}"  
ELSE (restock_date > 30 days OR null) THEN  
→ Visibility: HIDDEN  
→ Tags: ["PERMANENTLY_OUT"]  
→ Message: (none, product hidden)

**Shopify - Temporarily Out:**

Visibility: Published  
Button: "Notify me when available"  
Message: "Back in stock on May 15"

**Shopify - Permanently Out:**

Visibility: Hidden (removed from storefront)  
Customer: Can't find it in search

**Pro:**
- Flexible (handles both scenarios)
- Automatic decision based on Alyante data
- Customer sees what's coming back vs what's gone
- Best customer experience

**Con:**
- Requires extra field in Alyante ("Restock Date")
- Slight complexity in Zapier logic

---

## RECOMMENDATION

**Use OPTION 3 (Smart Toggle):**

1. Ask Alyante to add "Restock Date" field to product master
2. When product goes ESAURITO, team fills in restock date
3. Zapier automatically decides: hide or show

**If you can't add field in Alyante:**

Use OPTION 1 (Always Hide)
- Simpler
- Keep storefront clean
- When product back, manually un-hide

---

## Implementation Checklist

- [ ] Decide which approach (Option 1, 2, or 3)
- [ ] Create custom field in Shopify: "Status Alyante"
- [ ] Create tags in Shopify:
  - [ ] LAST_PIECES (for IN ESAURIMENTO)
  - [ ] TEMPORARILY_OUT (for ESAURITO temporary)
  - [ ] PERMANENTLY_OUT (for ESAURITO permanent)
- [ ] Configure Zapier logic for status mapping
- [ ] Test with 5 products: each status
- [ ] Monitor first week for correctness
- [ ] If using Option 3: add "Restock Date" field in Alyante
