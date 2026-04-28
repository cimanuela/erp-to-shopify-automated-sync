# Implementation Checklist - Step by Step

## PHASE 1: Preparation (Week 1)

### 1.1 Alyante Configuration
- [ ] Contact Alyante support
- [ ] Request daily CSV export configuration
- [ ] Specify: Daily at 00:30 (or your preferred time)
- [ ] Confirm file location: /server/exports/alyante-daily.csv
- [ ] Confirm encoding: UTF-8
- [ ] Confirm delimiter: Semicolon (;)
- [ ] Confirm columns needed (copy from alyante-export-spec.md)
- [ ] Test: Download one CSV manually to verify format
- [ ] Verify all 10 columns present:
  - [ ] CODICE ARTICOLO
  - [ ] NOME ARTICOLO
  - [ ] PESO
  - [ ] PREZZO BASE NO IVA
  - [ ] IVA
  - [ ] PREZZO LISTINO CON IVA INCL
  - [ ] COSTO
  - [ ] EAN ARTICOLO SINGOLO
  - [ ] QUANTITA' A MAGAZZINO ESCLUSI IMPEGNATI
  - [ ] STATUS ARTICOLO

### 1.2 Shopify Preparation
- [ ] Log in to Shopify admin: https://your-store.myshopify.com/admin
- [ ] Go to: Settings → Custom Fields → Products
- [ ] Create Custom Field 1:
  - [ ] Name: "Costo"
  - [ ] Type: Number
  - [ ] Description: "Cost/COGS from ERP"
- [ ] Create Custom Field 2:
  - [ ] Name: "Status Alyante"
  - [ ] Type: Dropdown
  - [ ] Options:
    - [ ] IN USO
    - [ ] IN ESAURIMENTO
    - [ ] ESAURITO_TEMPORARILY
    - [ ] ESAURITO_PERMANENTLY
    - [ ] SYNC_ERROR
- [ ] Verify custom fields visible in product edit page
- [ ] Go to: Settings → Custom fields → Products
- [ ] Create Tags in Products:
  - [ ] Tag: LAST_PIECES (for IN ESAURIMENTO)
  - [ ] Tag: TEMPORARILY_OUT (for ESAURITO temporary)
  - [ ] Tag: PERMANENTLY_OUT (for ESAURITO permanent)
  - [ ] Tag: SYNC_ERROR (for failed syncs)
  - [ ] Tag: DELISTED (optional alternative)

### 1.3 Shopify API Access
- [ ] Go to: Settings → Apps and integrations → Develop apps
- [ ] Create private app: "ERP Sync Automation"
- [ ] Set permissions:
  - [ ] read_products ✓
  - [ ] write_products ✓
  - [ ] read_inventory ✓
  - [ ] write_inventory ✓
- [ ] Copy API access token (save securely)
- [ ] Note Shopify store URL: myshopify.com

### 1.4 Zapier Setup
- [ ] Create Zapier account (free): zapier.com
- [ ] Or log in if you have account
- [ ] Create new Zap: "ERP-to-Shopify Daily Sync"
- [ ] Set to PRIVATE (not shared)

### 1.5 Server Access Verification
- [ ] Verify you can access the CSV file location
  - [ ] If SFTP: test login with credentials
  - [ ] If HTTP: test URL in browser
  - [ ] If Dropbox/Drive: verify Zapier can connect
- [ ] Keep server details handy for Zapier

---

## PHASE 2: Build Zapier Workflow (Week 2-3)

### 2.1 Create Trigger
- [ ] Zap Step 1: Trigger
- [ ] Type: Zapier Schedule
- [ ] Frequency: Daily
- [ ] Time: 00:35 (5 minutes after Alyante export)
- [ ] Timezone: Europe/Rome
- [ ] Save trigger
- [ ] Test trigger (click "Send test data")

### 2.2 Add Read CSV Step
- [ ] Zap Step 2: Action
- [ ] Type: HTTP request (or Webhooks by Zapier)
- [ ] Method: GET
- [ ] URL: {{your-server-url}}/alyante-daily.csv
  (OR if SFTP: use Zapier SFTP connector)
- [ ] Headers: If auth needed, add credentials
- [ ] Test: Send test request
- [ ] Verify: You see CSV content in response

### 2.3 Add Parse CSV Step
- [ ] Zap Step 3: Action
- [ ] Type: Formatter by Zapier
- [ ] Transform: CSV to JSON / Parse CSV
- [ ] Input: {{Response from Step 2 (CSV content)}}
- [ ] Settings: Set delimiter (semicolon)
- [ ] Test: Send test data
- [ ] Verify: Output is array of objects with 10 fields

### 2.4 Add Filter (Skip CODIFICATO)
- [ ] Zap Step 4: Filter
- [ ] Condition: STATUS ARTICOLO is not "CODIFICATO"
- [ ] Logic: Only continue if TRUE
- [ ] Test: Send test data
- [ ] Verify: CODIFICATO products filtered out

### 2.5 Add Loop
- [ ] Zap Step 5: Loop by Zapier
- [ ] Input: {{Array from Step 3}}
- [ ] Each iteration: Execute next steps for one product

### 2.6 Search Product in Shopify (Inside Loop)
- [ ] Zap Step 6: Action (inside loop)
- [ ] Type: Shopify (Search Products)
- [ ] Account: Connect your Shopify store
- [ ] Query: {{CODICE_ARTICOLO}} (SKU from CSV)
- [ ] Test: Use one SKU from your test data
- [ ] Verify: Returns product ID if exists

### 2.7 Add Conditional (IF Found)
- [ ] Zap Step 7: Filter
- [ ] Condition: "Product ID" from Step 6 exists (is not empty)
- [ ] Logic: If found, continue to Step 8
- [ ] If not found, skip to Step 11 (log as skipped)

### 2.8 Build Update Payload (If Found)
- [ ] Zap Step 8: Formatter
- [ ] Action: JSON builder
- [ ] Build object with fields:

{  
"product": {  
"id": {{Shopify Product ID from Step 6}},  
"variants": [  
{  
"id": {{Variant ID}},  
"price": "{{PREZZO_LISTINO}}",  
"barcode": "{{EAN}}",  
"inventory_quantity": {{QUANTITA}},  
"weight": "{{PESO}}"  
}  
],  
"metafields": [  
{  
"namespace": "custom",  
"key": "costo",  
"value": "{{COSTO}}",  
"type": "number_decimal"  
},  
{  
"namespace": "custom",  
"key": "status_alyante",  
"value": "{{STATUS}}",  
"type": "single_line_text_field"  
}  
]  
}  
}

- [ ] Test: Verify JSON output

### 2.9 Determine Tags & Visibility
- [ ] Zap Step 9: Formatter (or Path - Zapier)
- [ ] Logic: Create formula based on STATUS

IF STATUS == "IN USO":  
tags = ""  
visibility = "visible"  
IF STATUS == "IN ESAURIMENTO":  
tags = "LAST_PIECES"  
visibility = "visible"  
IF STATUS == "ESAURITO":  
tags = "TEMPORARILY_OUT"  
visibility = "hidden"

- [ ] Test: Verify outputs for each status

### 2.10 Update Shopify Product
- [ ] Zap Step 10: Action
- [ ] Type: Shopify (Update Product)
- [ ] Account: {{Same Shopify store}}
- [ ] Product: {{Product ID from Step 6}}
- [ ] Payload: {{JSON from Step 8}}
- [ ] Test: Update test product
- [ ] Verify in Shopify: Price/qty/fields updated

### 2.11 Apply Tags & Visibility
- [ ] Zap Step 11: Action (separate from Step 10)
- [ ] Type: Shopify (Update Product)
- [ ] Product: {{Product ID}}
- [ ] Tags: {{Tags from Step 9}}
- [ ] Visibility: {{Visibility from Step 9}}
- [ ] Test: Verify tags applied in Shopify

### 2.12 Log Success to Google Sheets
- [ ] Zap Step 12: Action (inside loop, inside IF found)
- [ ] Type: Google Sheets (Append spreadsheet row)
- [ ] Account: Connect your Google account
- [ ] Spreadsheet: Create new: "ERP Sync Logs"
- [ ] Worksheet: "Sync Log"
- [ ] Row values:
  - [ ] Timestamp: {{Now}}
  - [ ] SKU: {{CODICE_ARTICOLO}}
  - [ ] Product Name: {{NOME_ARTICOLO}}
  - [ ] Fields Updated: "Prezzo, Qty, Costo, Status"
  - [ ] Old Price: {{Shopify current price from Step 6}}
  - [ ] New Price: {{PREZZO_LISTINO}}
  - [ ] Old Qty: {{Shopify current qty}}
  - [ ] New Qty: {{QUANTITA}}
  - [ ] Status: "SUCCESS"
- [ ] Test: Verify row appended to Sheets

### 2.13 Handle Not Found (Else Branch)
- [ ] Zap Step 13: After Step 7 filter (else branch)
- [ ] Add Google Sheets action (if product not found)
- [ ] Spreadsheet: "ERP Sync Logs"
- [ ] Worksheet: "Skipped Products"
- [ ] Row values:
  - [ ] Timestamp: {{Now}}
  - [ ] SKU: {{CODICE_ARTICOLO}}
  - [ ] Product Name: {{NOME_ARTICOLO}}
  - [ ] Reason: "Product not found in Shopify"
  - [ ] Status: "SKIPPED"

### 2.14 Error Handling
- [ ] Add Zapier error handler (for each action step)
- [ ] If error: Log to Google Sheets "Errors" sheet
- [ ] Log: {{Error message}}, {{Step where error occurred}}
- [ ] Catch errors but don't stop workflow

---

## PHASE 3: Testing (Week 3)

### 3.1 Test with Small Dataset
- [ ] Pick 5 products from your Alyante CSV
- [ ] Make sure all 3 statuses are included:
  - [ ] 1x IN USO
  - [ ] 1x IN ESAURIMENTO
  - [ ] 1x ESAURITO
  - [ ] 1x CODIFICATO (to verify it's filtered)
  - [ ] 1x not in Shopify (to verify skipped)
- [ ] Run Zap manually (click "Test")
- [ ] Monitor execution: Check each step

### 3.2 Verify Shopify Updates
- [ ] After test run, check Shopify admin
- [ ] Product 1 (IN USO): Verify visible + price updated + qty updated
- [ ] Product 2 (IN ESAURIMIENTO): Verify visible + tag "LAST_PIECES" applied
- [ ] Product 3 (ESAURITO): Verify hidden + tag "TEMPORARILY_OUT"
- [ ] Product 4 (CODIFICATO): Verify NOT updated (skipped)
- [ ] Product 5 (not in Shopify): Verify logged as SKIPPED

### 3.3 Verify Google Sheets Logging
- [ ] Open "ERP Sync Logs" sheet
- [ ] Check "Sync Log" worksheet: 5 rows (or fewer if some skipped)
- [ ] Check "Skipped Products" worksheet: at least 2 rows
- [ ] Verify all data logged correctly

### 3.4 Test Error Scenarios
- [ ] Temporarily block Shopify API (to test error handling)
- [ ] Verify error logged to "Errors" sheet
- [ ] Verify workflow continues despite error

### 3.5 Run Full Test with All Products
- [ ] After 3.1-3.4 pass, run with FULL CSV
- [ ] Monitor: Does it complete without errors?
- [ ] Check: How many updated vs skipped vs errors?
- [ ] Time: How long did it take? (expect 10-20 min for 500 products)

---

## PHASE 4: Go Live (Week 4)

### 4.1 Final Review
- [ ] All test passes completed
- [ ] No errors in last test run
- [ ] Google Sheets logging working
- [ ] Shopify products updated correctly
- [ ] Visibility & tags applied correctly

### 4.2 Set Zap to Live
- [ ] In Zapier: Find your Zap "ERP-to-Shopify Daily Sync"
- [ ] Status: Turn ON (click toggle)
- [ ] Schedule: Verify set to 00:35 daily
- [ ] Save

### 4.3 Monitor First Week
- [ ] Day 1 (May 1): Check logs at 01:00
  - [ ] Was Zap triggered? (check "Zap runs" in Zapier)
  - [ ] Were products updated? (check Shopify)
  - [ ] Any errors? (check Google Sheets)
- [ ] Day 2-3: Spot check (repeat)
- [ ] Day 4-7: Random checks, no action needed if working

### 4.4 Set Up Alert Emails
- [ ] (Optional) Add Zapier action: Send email if error
- [ ] Email to: you@example.com
- [ ] Subject: "ERP Sync Error - {{Timestamp}}"
- [ ] Body: "{{Error message}}"
- [ ] So you're notified if something breaks

### 4.5 Document in Google Sheets
- [ ] Create "System Status" sheet in ERP Sync Logs
- [ ] Log:
  - [ ] Go-live date: May 1, 2026
  - [ ] Daily sync time: 00:35
  - [ ] CSV source: Alyante
  - [ ] Products synced: 487
  - [ ] Last successful run: {{date}}
  - [ ] Known issues: (none yet)

---

## PHASE 5: Maintenance (Ongoing)

### 5.1 Weekly
- [ ] Check Google Sheets "Errors" sheet (any errors?)
- [ ] Spot check 2-3 products in Shopify (up to date?)
- [ ] Check Zapier "Zap runs" (all executed?)

### 5.2 Monthly
- [ ] Review sync logs (summary)
  - [ ] Total products synced
  - [ ] Avg execution time
  - [ ] Any patterns in errors
- [ ] Verify Alyante export still running at 00:30
- [ ] Verify server file accessible
- [ ] Check for "dirty data" (CODIFICATO) in Alyante
  - [ ] Ask team to clean if needed

### 5.3 Quarterly
- [ ] Review for optimization opportunities
  - [ ] Can we reduce execution time?
  - [ ] Can we add more fields to sync?
  - [ ] Should we add Claude AI for descriptions? (July)
- [ ] Update documentation if changes made

---

## TROUBLESHOOTING

### Problem: Zap didn't run at 00:35
**Solution:**
- [ ] Check Zapier Status page (is Zapier down?)
- [ ] Check Zap is set to ON (toggle)
- [ ] Check timezone is correct (Europe/Rome)
- [ ] Manually trigger Zap to test

### Problem: CSV file not found
**Solution:**
- [ ] Verify file exists on server at {{url}}
- [ ] Verify Zapier has access (firewall?)
- [ ] Verify file permissions allow read
- [ ] Contact IT if SFTP needed

### Problem: Products not updating in Shopify
**Solution:**
- [ ] Verify Shopify API token is valid
- [ ] Check "Errors" sheet in Google Sheets
- [ ] Manual test: Try 1 product via Zapier test
- [ ] Verify custom fields exist in Shopify

### Problem: Wrong products being filtered
**Solution:**
- [ ] Check CSV data (is STATUS column correct?)
- [ ] Verify filter condition in Step 4
- [ ] Test filter manually with sample data

### Problem: Execution time too long (> 30 min)
**Solution:**
- [ ] Reduce number of products in CSV (filter out CODIFICATO better)
- [ ] Add batching: sync 100 products per run, run 5x daily
- [ ] Upgrade to Zapier Pro (faster)

---

## Completion Checklist

When all done:
- [ ] All 5 phases completed
- [ ] 1 week of successful daily syncs
- [ ] No critical errors
- [ ] Google Sheets logging working
- [ ] Shopify products updated correctly
- [ ] Documentation complete (this file)
- [ ] GitHub repo updated with any learnings

**Status: PRODUCTION READY ✓**

---

## Next Steps

After this is working (May 31):
- [ ] Phase 2: Add Claude AI for description optimization (June)
- [ ] Phase 3: Add Klaviyo integration for customer notifications (June)
- [ ] Phase 4: Add webhook for real-time updates (July - if needed)

---


