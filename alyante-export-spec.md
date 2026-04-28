# TeamSystem Alyante CSV Export Specification

## What to Request from Alyante

Ask your Alyante team to configure:

**Export Type:** Daily automated product export to CSV

**Schedule:** Daily at 00:30 (or your preferred time)

**File Location:** /server/exports/alyante-daily.csv
(or accessible via SFTP / HTTP)

**File Encoding:** UTF-8

**Delimiter:** Semicolon (;) or comma (,) - specify

**Columns Required (in any order):**
- CODICE ARTICOLO (Product SKU)
- NOME ARTICOLO (Product Name)
- PESO (Weight)
- PREZZO BASE NO IVA (Base Price without VAT)
- IVA (Tax %)
- PREZZO LISTINO CON IVA INCL (List Price with VAT)
- COSTO (Cost/COGS)
- EAN ARTICOLO SINGOLO (EAN/Barcode)
- QUANTITA' A MAGAZZINO ESCLUSI IMPEGNATI (Available Qty, excluding committed)
- STATUS ARTICOLO (Product Status)

## Example CSV Format

CODICE ARTICOLO;NOME ARTICOLO;PESO;PREZZO BASE NO IVA;IVA;PREZZO LISTINO CON IVA INCL;COSTO;EAN ARTICOLO SINGOLO;QUANTITA' A MAGAZZINO ESCLUSI IMPEGNATI;STATUS ARTICOLO  
12345;Happy Cat Minkas 10kg;10.5;37.80;22;46.50;18.50;8024109123456;45;IN USO  
12346;Dog Food Premium 5kg;5.2;18.00;22;21.96;8.50;8024109654321;12;IN ESAURIMENTO  
12347;Cat Treats 200g;0.2;2.50;22;3.05;1.00;8024109999999;0;ESAURITO  
12348;Prototype Product;1.0;50.00;22;61.00;25.00;;0;CODIFICATO

## Status Values

Only these values in STATUS ARTICOLO column:
- `CODIFICATO` - Not ready (skip in Zapier)
- `IN USO` - Active (sync to Shopify)
- `IN ESAURIMENTO` - Low stock (sync + warn)
- `ESAURITO` - Out of stock (sync + decision: hide or show?)

## Validation Rules

Zapier will validate each row:
- SKU cannot be empty
- Status must be one of above 4 values
- Quantity must be number ≥ 0
- Price must be > 0 (or skip if issue)
- EAN: if empty, OK (optional field)

## File Retention

Keep CSV file on server for at least 7 days
(in case Zapier needs to re-sync)

