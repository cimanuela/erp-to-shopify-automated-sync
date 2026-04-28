# Error Handling & Recovery Guide

## Common Errors & Solutions

### ERROR 1: HTTP 404 - File Not Found

**Where it happens:** Step 2 (Read CSV from server)

**Cause:** 
- Server path is wrong
- File doesn't exist yet
- Server is down
- Alyante export failed

**What you'll see in Zapier:**

Error: 404 Not Found  
URL: https://server/exports/alyante-daily.csv

**What you'll see in Google Sheets "Errors":**

Timestamp: 2026-05-01 00:35:30  
Step: Read CSV from server  
Error: 404 File not found  
Action: Retry at 12:00 noon

**How to fix:**
1. Check server path is correct
2. SSH into server, verify file exists: `ls -la /exports/alyante-daily.csv`
3. Verify Alyante export ran (check cron job)
4. If Alyante failed, contact their team

**Prevention:**
- [ ] Add monitoring: alert if file missing at 00:40
- [ ] Set Zapier to retry at 12:00 if failed

---

### ERROR 2: Authentication Failed

**Where it happens:** Step 2 (Read CSV via SFTP/HTTP)

**Cause:**
- Wrong username/password
- SSH key expired
- Token revoked
- Wrong server URL

**What you'll see in Zapier:**

Error: 401 Unauthorized  
Authentication failed for SFTP server

**How to fix:**
1. Verify credentials are correct
2. Check if SSH key needs renewal: `ssh-keygen -y -f key.pem`
3. Test connection manually: `sftp user@server.com`
4. Re-authenticate in Zapier: Settings → Connections → Reconnect

**Prevention:**
- [ ] Save credentials securely (use password manager)
- [ ] Set reminder: SSH key renewal 1 month before expiry

---

### ERROR 3: Shopify API Rate Limit

**Where it happens:** Step 10 (Update Shopify Product)

**Cause:**
- Too many API calls too fast
- Shopify limits: ~2 calls per second
- If 500+ products: might hit limit

**What you'll see in Zapier:**

Error: 429 Too Many Requests  
Retry-After: 60 (seconds)

**What happens:**
- Zap pauses
- Waits 60 seconds
- Retries automatically
- Workflow continues (slower)

**How to fix:**
1. Nothing needed - Zapier handles automatically
2. Workflow just takes longer (15-20 min instead of 10)

**Prevention:**
- [ ] If >500 products: Use Zapier Pro to batch requests
- [ ] Run sync during off-peak hours (00:35 is good)

---

### ERROR 4: SKU Not Found in Shopify

**Where it happens:** Step 6 (Search Shopify by SKU)

**Cause:**
- Product not created in Shopify yet
- SKU mismatch (Alyante SKU ≠ Shopify SKU)
- Product archived/deleted

**What you'll see in Zapier:**

No product found with SKU "12345"  
Product ID: (empty)

**What happens:**
- Filter at Step 7 catches this (product not found)
- Skips update for this product
- Logs to "Skipped Products" sheet

**How to fix:**
1. Check Google Sheets "Skipped Products"
2. For each skipped:
   - Is product actually in Shopify? Search manually
   - Is SKU different in Shopify? Update SKU to match Alyante
   - Is product archived? Un-archive it
3. Re-run Zap manually or wait for next day

**Prevention:**
- [ ] Before going live: Audit SKU matches (Alyante vs Shopify)
- [ ] Create SKU mapping document
- [ ] Standardize: Always use Alyante SKU as Shopify SKU

---

### ERROR 5: Bad CSV Data

**Where it happens:** Step 3 (Parse CSV) or Step 8 (Build payload)

**Cause:**
- Null values in CSV (price = empty)
- Wrong data type (price = "abc" instead of number)
- Special characters (é, ñ) encoding issues
- Missing columns

**What you'll see in Zapier:**

Error: Cannot parse CSV  
Row 45: Invalid data type  
Field "PREZZO_LISTINO": Expected number, got "abc"

**What happens:**
- That row fails
- Logged to "Errors" sheet
- Workflow continues with next row

**How to fix:**
1. Check Alyante CSV export (look for row 45)
2. Find the bad data
3. Fix in Alyante
4. Wait for next export (or manually export)
5. Run Zap again

**Prevention:**
- [ ] Add data validation in Alyante
- [ ] Before sync: Run CSV validation script
- [ ] Test with sample data first

---

### ERROR 6: Shopify Custom Field Doesn't Exist

**Where it happens:** Step 8 (Build payload with custom fields)

**Cause:**
- Custom field "Costo" or "Status Alyante" not created
- Field deleted accidentally
- Typo in field name

**What you'll see in Zapier:**

Error: Unknown field "costo"  
Custom field does not exist in Shopify

**How to fix:**
1. Go to Shopify: Settings → Custom Fields → Products
2. Verify custom fields exist:
   - [ ] "Costo"
   - [ ] "Status Alyante"
3. If missing, create them (see Phase 1.2)
4. Re-run Zap

**Prevention:**
- [ ] Don't delete custom fields (archive instead)
- [ ] Document custom fields in GitHub repo

---

### ERROR 7: Visibility Toggle Fails

**Where it happens:** Step 11 (Apply visibility & tags)

**Cause:**
- Product has draft variant
- Product has no variants
- Visibility API endpoint changed

**What you'll see in Zapier:**

Error: Cannot update product visibility  
Invalid variant state

**How to fix:**
1. Check product in Shopify (is it normal product with variants?)
2. Verify product has at least 1 variant
3. Try updating visibility manually in Shopify
4. If manual update works, try Zapier again

**Prevention:**
- [ ] Before sync: Audit all products have variants
- [ ] Don't use Shopify "Collections" as products

---

### ERROR 8: Google Sheets Sheet Not Found

**Where it happens:** Step 12 (Log to Google Sheets)

**Cause:**
- Spreadsheet deleted
- Sheet/worksheet name changed
- Google account disconnected

**What you'll see in Zapier:**

Error: Sheet "Sync Log" not found in spreadsheet

**What happens:**
- Logging fails
- Zap continues (doesn't stop)
- No audit trail for that run

**How to fix:**
1. Verify Google Sheet exists: https://sheets.google.com
2. Verify sheet name is exactly "Sync Log" (case-sensitive)
3. Re-authenticate in Zapier: Settings → Connections → Google Sheets
4. Update Zap with correct sheet name

**Prevention:**
- [ ] Don't delete or rename sheets
- [ ] Test Google Sheets connection monthly
- [ ] Keep shared Google Sheet URL in GitHub

---

## Error Handling Strategy

### Tier 1: Automatic Retry
- Zapier retries automatically for:
  - Network errors (timeouts)
  - Temporary API errors (429, 503)
  - Transient failures

**Your action:** None (Zapier handles)

### Tier 2: Skip & Log
- For expected errors (SKU not found):
  - Skip product
  - Log to "Skipped" sheet
  - Continue with next product

**Your action:** Check "Skipped" sheet monthly, fix manually

### Tier 3: Alert & Escalate
- For critical errors (no CSV file, API auth):
  - Log to "Errors" sheet
  - Send email alert
  - Require manual intervention

**Your action:** Fix immediately when alerted

### Tier 4: Pause & Wait
- For rate limits (Shopify API):
  - Zapier pauses
  - Waits for reset
  - Retries
  - Continues automatically

**Your action:** None (Zapier handles, just takes longer)

---

## Monitoring Dashboard (Google Sheets)

Create a "System Status" sheet with:

Sync Date | Trigger Time | Products Processed | Updated | Skipped | Errors | Duration | Status | Notes  
2026-05-01 | 00:35 | 487 | 485 | 2 | 0 | 12 min | ✓ SUCCESS | All good  
2026-05-02 | 00:35 | 487 | 484 | 2 | 1 | 13 min | ⚠ WARNING | 1 API error, retried  
2026-05-03 | 00:35 | 0 | 0 | 0 | 1 | - | ✗ FAILED | CSV file not found - Alyante down

**Update manually each morning (or automate with Zapier)**

---

## Recovery Procedures

### Scenario: Zap Failed, Need to Re-sync Yesterday's Data

**Steps:**
1. Download yesterday's CSV from server
2. In Zapier: Find the failed Zap
3. Click "Replay" (if available)
4. Or manually re-run with saved data

### Scenario: Wrong Data Updated in Shopify, Need Rollback

**Steps:**
1. Check "Sync Log" sheet - see what changed
2. Manually revert in Shopify (or restore from backup)
3. Fix source data in Alyante
4. Wait for next sync (or manually trigger)

### Scenario: Custom Field Deleted Accidentally

**Steps:**
1. Go to Shopify: Settings → Custom Fields → Products
2. Recreate field with same name & type
3. Existing products won't have data (that's ok)
4. Next sync will populate it

### Scenario: Need to Pause Sync Temporarily

**Steps:**
1. In Zapier: Find Zap "ERP-to-Shopify"
2. Click toggle: Turn OFF
3. Do your maintenance in Shopify
4. Turn Zap back ON
5. (Or manually trigger if needed urgently)

---

## Testing Error Scenarios

**Before going live, test:**

- [ ] What happens if CSV file missing? (test by blocking URL)
- [ ] What happens if Shopify API down? (use Postman to simulate)
- [ ] What happens if invalid CSV data? (manually corrupt data)
- [ ] What happens if custom field missing? (delete it, see error)
- [ ] What happens if rate limit hit? (run with huge dataset)

---

## Contacts & Escalation

**If you need help:**

1. **Alyante Issue:**
   - Contact: Alyante support team
   - Issue: CSV export failed
   - Expected response: 24 hours

2. **Shopify Issue:**
   - Contact: Shopify support (or your IT team)
   - Issue: API error, custom field issue
   - Expected response: 24 hours

3. **Zapier Issue:**
   - Contact: Zapier support (or community forum)
   - Issue: Zap not triggering, logic not working
   - Expected response: 24-48 hours

4. **Server/Network Issue:**
   - Contact: Your IT team
   - Issue: SFTP access, firewall blocking
   - Expected response: Hours

---

## Checklist: Before Declaring "PRODUCTION READY"

- [ ] All error scenarios tested
- [ ] Error handling in place for each step
- [ ] Google Sheets logging working
- [ ] Alert emails configured
- [ ] Team knows how to access logs
- [ ] Runbook/documentation in place
- [ ] Weekly monitoring schedule set
- [ ] Rollback procedure documented
- [ ] Escalation contacts listed
- [ ] One week of clean runs completed

**Once all ✓:** Status = PRODUCTION READY
