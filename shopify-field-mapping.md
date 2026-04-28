# Shopify Field Mapping Guide

## Which Shopify Fields to Update

### Standard Fields (always exist)

| Alyante | Shopify | How to Find |
|---|---|---|
| PREZZO LISTINO CON IVA | Product Price | Edit product → Pricing section → Price |
| QUANTITA' MAGAZZINO | Quantity (available) | Edit product → Inventory section → Available |
| EAN ARTICOLO | Barcode | Edit product → Inventory section → Barcode |
| PESO | Weight | Edit product → Shipping section → Weight |

### Custom Fields (must create first)

CUSTOM FIELD 1: "Costo" (Cost/COGS)

Settings → Custom fields → Products  
Type: Number  
Name: Costo  
Used for: Calculate margin (for internal use)

CUSTOM FIELD 2: "Status Alyante"

Settings → Custom fields → Products  
Type: Dropdown  
Name: Status Alyante  
Options:

- IN USO
- IN ESAURIMENTO
- ESAURITO_TEMPORARILY
- ESAURITO_PERMANENTLY
- SYNC_ERROR

### Tags (use for filtering/behavior)

TAG: LAST_PIECES

- Applied when: Status = IN ESAURIMENTO
- Purpose: Filter low-stock products
- Display: Can be shown in collections/filters

TAG: TEMPORARILY_OUT

- Applied when: Status = ESAURITO (will restock)
- Purpose: Hide product temporarily
- Display: Don't show in storefront

TAG: PERMANENTLY_OUT

- Applied when: Status = ESAURITO (discontinued)
- Purpose: Keep visible but with "Out of Stock" message
- Display: Show in storefront but disable "Add to Cart"

TAG: DELISTED (alternative to above)

- Purpose: Mark for deletion/archiving

## Implementation in Zapier

Zapier step: Update Shopify Product  
Fields to update:  
{  
"product": {  
"id": <shopify_product_id>,  
"title": <keep_existing>,  
"vendor": <keep_existing>,  
"variants": [  
{  
"price": <PREZZO LISTINO>,  
"barcode": <EAN>,  
"weight": <PESO>,  
"inventory_quantity": <QUANTITA'>  
}  
],  
"metafields": [  
{  
"namespace": "custom",  
"key": "costo",  
"value": <COSTO>,  
"type": "number_decimal"  
},  
{  
"namespace": "custom",  
"key": "status_alyante",  
"value": <STATUS>,  
"type": "single_line_text_field"  
}  
],  
"tags": <TAG_BASED_ON_STATUS>  
}  
}

## Status → Tag Mapping

IN USO → tags: []  
IN ESAURIMENTO → tags: ["LAST_PIECES"]  
ESAURITO → IF restock_date < 30days: ["TEMPORARILY_OUT"]  
→ ELSE: ["PERMANENTLY_OUT"]
