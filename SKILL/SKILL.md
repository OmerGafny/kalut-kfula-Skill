---
name: Kalut Kfula WEB API
display_name: Offical skill for Kalut Kfula Web API - Accounting and Invoicing software.
description: Offical skill for Kalut Kfula Web API - Accounting and Invoicing software. This is a limited demo version. For api key or full version send email to support@gafny.com
---

# Kalut Kfula WEB Invoice API — Developer Skill

Build invoice applications with Claude using Kalut Kfula WEB's compliant Israeli tax API.

**For full Hebrew documentation, see [SKILL_HE.md](SKILL_HE.md)**

## Quick Start
1. Create an account at https://app.gafny.com/Identity/Account/Register 
2. Get your API key from account settings
3. Add this skill to your IDE:
   - **Cursor/Windsurf:** paste into `.cursor/rules/yourapp.md`
   - **Claude:** paste into Project instructions
4. Start building: `"Add invoice creation to my app"`
## What This Skill Does

Teaches Claude how to integrate with Kalut Kfula WEB's invoicing API — search clients, create compliant Israeli invoices, handle payments, and generate secure document links.

## API Endpoints

- `GET /clients?q={term}` — search clients
- `GET /products?q={term}` — search products  
- `GET /nextdocnumber?docType={type}` — get next document number
- `POST /invoice/preview` — preview HTML before creation
- `POST /invoice` — create and email document

All requests require header: `X-Api-Key: {API_KEY}`

## Document Types

- **"1"** = חשבונית (Invoice) — no payments
- **"2"** = חשבונית מס קבלה (Invoice + Receipt) — requires payments
- **"3"** = חשבונית זיכוי (Credit Invoice) — no payments
- **"4"** = קבלה (Receipt) — requires payments

## Mandatory Sequence

1. Search client → GET /clients
2. Confirm client selection
3. Search products → GET /products
4. Confirm product selection
5. Get document number → GET /nextdocnumber
6. Preview → POST /invoice/preview
7. Get explicit approval
8. Create → POST /invoice

---

# KalutKfula Web API — Developer Skill
version: 1.1.0
updated: 2026-05-21

You are helping a developer integrate with KalutKfula Web — a compliant 
Israeli tax document API. Apply these rules to every piece of code 
you generate that touches invoice creation, client search, or 
product search.

Confirm you have read and understood this skill before writing 
any integration code.

---

## API Reference

Base URL: https://app.gafny.com/api/Main

Authentication — every request must include:
X-Api-Key: gaf_xxxxx-xxxxxx  ← always from environment variable, never hardcoded

Endpoints:
GET  /clients?q={term}&max=10        → search clients
GET  /products?q={term}&max=10       → search products  
GET  /nextdocnumber?docType={type}   → get next document number
POST /invoice/preview                → generate HTML preview
POST /invoice                        → create document

All responses follow this envelope:
{ "success": true,  "data": ... }
{ "success": false, "error": "..." }

Always check success field — never assume 200 means success.

---

## Environment Variables
Always use environment variables for credentials.
Never hardcode API key in any file served to the browser.

Backend (.env):
Kalut Kfula WEB_API_KEY=gaf_xxxxx-xxxxxx
Kalut Kfula WEB_BASE_URL=https://app.gafny.com/api/Main

Frontend — API key must never appear. Route all calls through 
a backend proxy or server function.

---

## Core API Client Pattern
Generate this utility once per project, adapted to the project's 
existing HTTP library and patterns:

// Node.js / Next.js server side
const Kalut Kfula WEBApi = {
  baseUrl: process.env.Kalut Kfula WEB_BASE_URL,
  headers: {
    "X-Api-Key": process.env.Kalut Kfula WEB_API_KEY,
    "Content-Type": "application/json"
  },

  async get(path) {
    const res = await fetch(`${this.baseUrl}${path}`, {
      headers: this.headers
    });
    const data = await res.json();
    if (!data.success) throw new Error(data.error);
    return data.data;
  },

  async post(path, body) {
    const res = await fetch(`${this.baseUrl}${path}`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify(body)
    });
    const data = await res.json();
    if (!data.success) throw new Error(data.error);
    return data;
  }
};

Adapt this pattern to the project's existing conventions:
- If project uses axios → use axios
- If project uses a custom fetch wrapper → use that
- If project is React Native → use appropriate fetch
- Always keep the credential handling server-side

---

## Mandatory Integration Flow
Every invoice creation flow must implement these steps in order.
Never skip or reorder steps. Generate all steps even if the 
developer only asks for part of the flow.

1. CLIENT SEARCH
   GET /clients?q={userInput}&max=10
   → present results to user
   → user selects one explicitly
   → if no results: offer to create new client

2. PRODUCT SEARCH (once per line item)
   GET /products?q={userInput}&max=10
   → present results to user
   → user selects one explicitly  
   → if no results: collect name and price manually
   → never invent or estimate prices

3. GET DOCUMENT NUMBER
   GET /nextdocnumber?docType={type}&year={currentYear}
   → store result, do not show to user as editable field
   → call this as late as possible before preview
   → never reuse a number from a failed attempt

4. PAYMENT DETAILS (DocType 2 and 4 only)
   → calculate total from line items including VAT
   → ask user which payment method(s)
   → never ask user for payment amounts — derive from invoice total
   → collect required fields per payment type (see Payment section)

5. GENERATE PREVIEW
   POST /invoice/preview
   → same payload as creation
   → render previewHtml in iframe — never in div or modal
   → stateless — safe to call multiple times
   → wait for explicit user confirmation

6. CREATE DOCUMENT (only after explicit confirmation)
   POST /invoice
   → exact same payload as preview
   → render returned messageHtml directly
   → if messageHtml contains email confirmation: inform user

---

## Request Shape

### Invoice / Preview (same payload):
{
  DocType:      string   // "1","2","3","4" — see Document Types
  DocNumber:    string   // from /nextdocnumber only
  PublishDate:  string   // yyyy-MM-dd
  PublishTime:  string   // HH:mm
  ClientId:     string   // from /clients — "0" for new client
  ClientName:   string   // exact name from /clients
  ClientEmail:  string   // "" if none — never omit
  ClientCity:   string   // "" if none — never omit
  ClientAddress: string  // "" if none — never omit
  ClientZip:    string   // "" if none — never omit
  ClientMobile: string   // "" if none — never omit
  IncludeVat:   boolean  // true/false — never string
  DocComments:  string   // "" if none — never omit
  Lines: [
    {
      Id:     string  // catalogNumber from /products — "" if manual
      Name:   string  // product description
      Price:  number  // unit price before VAT — never string
      Amount: number  // quantity — never string
    }
  ]
  Payments: [...]  // required for DocType 2 and 4 only
}

### Line item rules:
- Price and Amount are always numbers — never strings
- Never calculate VAT in the frontend — server handles it
- Never invent prices — only from /products or explicit user input

---

## Response Shapes

### GET /clients
{
  "success": true,
  "data": [
    {
      "accountId":  "123",
      "name":       "Acme Ltd",
      "email":      "acme@example.com",
      "phone":      "050-0000000",
      "city":       "תל אביב",
      "address":    "רחוב הרצל 1",
      "zip":        "6100000",
      "vatNumber":  "123456789"
    }
  ]
}

### GET /products
{
  "success": true,
  "data": [
    {
      "catalogNumber": "P1",
      "name":          "Consulting",
      "price":         "100.00",
      "unit":          "שעה"
    }
  ]
}

### GET /nextdocnumber
{
  "success": true,
  "data": {
    "nextDocumentNumber": 1235,
    "lastDocumentNumber": 1234,
    "docType": "1",
    "year": "2026"
  }
}

CRITICAL: Always use data.nextDocumentNumber — not data itself.
Never stringify the entire data object — it will show "[object Object]".

Example usage:
const response = await fetch(...);
const result = await response.json();
const docNumber = result.data.nextDocumentNumber;  ← extract this property

### POST /invoice/preview
{
  "success": true,
  "data": {
    "previewUrl": "<html>...</html>"
  }
}

- previewUrl contains formatted HTML preview of the document
- Render it directly in an iframe — never in a div or modal
- Never sanitize or parse as text — render as-is
- Preview is completely stateless — safe to call multiple times

Example rendering:
const response = await fetch(...);
const result = await response.json();
const iframe = document.createElement('iframe');
iframe.srcDoc = result.data.previewUrl;
document.getElementById('preview-container').appendChild(iframe);

### POST /invoice
{
  "success": true,
  "data": {
    "messageHtml": "<html>...<a href='https://s3.../document.pdf'>לצפייה במסמך</a>...</html>"
  }
}

- messageHtml contains formatted HTML confirmation
- The presigned document link is already embedded in the HTML
- Render it directly in your app — never sanitize or parse it
- Do not navigate away automatically — let user decide

Example rendering:
const response = await fetch(...);
const result = await response.json();
document.getElementById('success-container').innerHTML = result.data.messageHtml;

### Error Response (all endpoints)
{
  "success": false,
  "error": "exact error message"
}

- Show the exact error message to the user
- Do not retry automatically
- Wait for user instruction

---

## Document Types
"1" = חשבונית (Invoice)
      triggers: "חשבונית", "invoice", "inv"
      no Payments array

"2" = חשבונית מס קבלה (Invoice + Receipt)
      triggers: "חשבונית מס קבלה", "חשבונית קבלה", "invoice receipt"
      requires Payments array

"3" = חשבונית זיכוי (Credit Invoice)
      triggers: "חשבונית זיכוי", "זיכוי", "credit invoice", "credit note"
      send line prices as positive numbers
      server applies negative values automatically
      no Payments array

"4" = קבלה (Receipt)
      triggers: "קבלה", "receipt"
      requires Payments array

If document type is ambiguous — generate a selector UI, never hardcode an assumption.

---

## Payment Types and Required Fields

"7" מזומן      → PaymentType, PaymentAmount, PaymentDate
"1" שיק        → + PaymentNumberId, BankId, BankName, 
                   BankBranch, BankAccount, PaymentDate
"2" שטרות      → PaymentType, PaymentAmount, PaymentDate
"3" כרטיס אשראי → + PaymentNumberId, CVV, CardDate, CreditDealType
                   if CreditDealType=="8": + NumberOfCreditPayments,
                   CreditFirstPaymentAmount, CreditNotFirstPaymentAmount
                   BankId and BankName are auto-detected — not user input
"4" שובר תשלום → PaymentType, PaymentAmount, PaymentDate
"5" הוראת קבע  → PaymentType, PaymentAmount, PaymentDate
"6" העברה      → PaymentType, PaymentAmount, PaymentDate
"8" ניכוי במקור → PaymentType, PaymentAmount, PaymentDate — once only
"9" bit        → PaymentType, PaymentAmount, PaymentDate

Payment constraints:
- PaymentAmount always derived from invoice total — never from user input
- Sum of all PaymentAmounts must equal invoice total exactly
- BankAccount max 9 digits, no dashes
- BankBranch no dashes
- Only one credit card line per document
- ניכוי במקור only once per document

---

## Payment Collection Sequence

1. Complete the full Lines sequence first — client, products, document number
2. Calculate the final total including VAT
3. Ask the user: "כיצד תרצה לשלם? (סכום לתשלום: ₪{total})"
4. Ask which payment method(s) they want to use
5. If multiple methods: ask how to split — but never ask for amounts,
   only ask "כמה בכרטיס אשראי?" and derive the rest automatically
6. Collect any additional required fields based on payment type
7. Build the Payments array with correct amounts
8. Generate preview → wait for approval → create document

---

## UI Component Patterns
Generate these patterns regardless of framework:

CLIENT SEARCH COMPONENT:
- Autocomplete input calling GET /clients as user types
- Debounce 300ms minimum
- Show name and accountId in dropdown
- On selection: store full client object in state
- Never allow free text submission without API confirmation
- Show "לא נמצא לקוח — ליצור חדש?" if empty results

PRODUCT SEARCH COMPONENT:
- Autocomplete input calling GET /products as user types
- Debounce 300ms minimum
- Show name, catalogNumber, price in dropdown
- On selection: store full product object, auto-fill price
- Price field read-only if product selected from API
- Price field editable only for manual (not found) products
- Never allow price of 0

DOCUMENT NUMBER FIELD:
- Never render as editable input
- Fetch automatically on component mount or DocType change
- Show as read-only display — e.g. "מסמך מספר: 1235"
- Refetch if DocType changes

PREVIEW COMPONENT:
- Always iframe, never div or innerHTML
- Full width, minimum 600px height
- Show loading spinner while fetching
- Disable confirm button until preview has loaded
- Show preview before showing confirm button

CONFIRM BUTTON:
- Disabled until preview is displayed
- Disabled during submission — prevent double click
- Show loading state during POST /invoice
- Label: "אישור יצירת מסמך" / "Confirm Document Creation"

SUCCESS CONTAINER:
- Render messageHtml directly — never sanitize
- The HTML already contains the document link
- Show loading spinner until response arrives
- Allow user to close/dismiss after viewing

PAYMENT FORM (DocType 2 and 4):
- Show payment total derived from invoice — read only
- Allow adding multiple payment lines
- Show relevant fields per selected PaymentType
- Validate sum of payments equals invoice total before enabling preview

---

## Error Handling Patterns
Generate consistent error handling:

// API errors
try {
  const result = await Kalut Kfula WEBApi.post("/invoice", payload);
  document.getElementById('success').innerHTML = result.data.messageHtml;
} catch (error) {
  handleApiError(error.message);
}

// Error display by status:
401 → "מפתח API לא תקין — אנא צור מפתח חדש בהגדרות החשבון"
400 → show exact error message from response
500 → "שגיאת שרת — אנא נסה שוב"
network → "בעיית תקשורת — בדוק את החיבור לאינטרנט"

Always show errors in Hebrew for end user facing messages.
Always log full error details for developer debugging.

---

## What Never To Do
- Never put API key in any frontend file
- Never call POST /invoice without preceding POST /invoice/preview
- Never reuse document numbers
- Never let price fields be free text for catalog products
- Never let document number be an editable field
- Never calculate VAT in frontend code
- Never send Price or Amount as strings
- Never omit optional string fields — send "" instead
- Never call POST /invoice with payload different from last preview
- Never generate mock or placeholder invoice data
- Never skip client or product search steps
- Never assume payment amount — always derive from invoice total
- Never stringify response data objects directly — extract specific properties
- Never sanitize or parse messageHtml — render as-is
- Never use div innerHTML for HTML responses — use iframe for previews
