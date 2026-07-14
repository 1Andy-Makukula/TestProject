# KithLy Security, Performance, and UX Audit Reports

## 1. Sentinel (The Security Auditor) - Security Audit Report
**Scope:** `orders` and `merchant_balances` tables

### Findings:
1. **Missing Row Level Security (RLS) on `merchant_balances`:**
   - The `merchant_balances` table is queried directly by `MerchantDashboard.tsx` (`from('merchant_balances')`), but it is missing from `DATABASE_SETUP.md` and appears to lack any RLS policies.
   - **Vulnerability:** If RLS is not strictly enforced, any authenticated user could theoretically run `supabase.from('merchant_balances').select('*')` and view the balances of any other merchant on the platform.
   - **Fix:** Enable RLS on the table (`ALTER TABLE merchant_balances ENABLE ROW LEVEL SECURITY;`) and add a policy that restricts access using the `merchant_shops` junction table:
     ```sql
     CREATE POLICY "Merchants can read their shop balance" ON merchant_balances FOR SELECT USING (
       EXISTS (SELECT 1 FROM merchant_shops WHERE merchant_shops.shop_id = merchant_balances.shop_id AND merchant_shops.user_id = auth.uid())
     );
     ```

2. **Overly Permissive RLS on `orders` table:**
   - The policy `Anyone can read order by code (for recipient page)` is currently defined as:
     ```sql
     CREATE POLICY "Anyone can read order by code (for recipient page)" ON orders FOR SELECT USING (true);
     ```
   - **Vulnerability:** Although the intention is to allow reads for the recipient page, using `true` effectively allows any authenticated user (or anonymous user, if public) to query and list *all* orders in the database, ignoring the intended recipient code check. This is a severe data leak exposing sender IDs, recipient details, and transaction amounts.
   - **Fix:** Remove this overly broad policy. Instead, create a secure Supabase Edge Function to fetch order details for the recipient page by code, or restrict the policy securely if possible.

## 2. Bolt (The Performance Analyst) - Performance Report
**Scope:** `MerchantDashboard.tsx` and `server/index.ts` edge function

### Findings:
1. **Inefficient Client-Side Aggregation (`MerchantDashboard.tsx`):**
   - The `fetchAnalytics` function pulls the raw list of all historical fulfilled orders (`select('amount, fulfilled_at')`) to the client, and calculates total value/fulfilled counts in the browser.
   - **Latency Impact:** For a merchant with thousands or millions of transactions, fetching this volume of data will lead to massive payload sizes, extremely slow loading times, bandwidth spikes, and UI lag.
   - **Fix Suggestion:** Shift the analytics aggregation to the backend. Create a PostgreSQL Database View or a Supabase RPC function (e.g., `get_merchant_analytics`) that performs the calculations (`COUNT()`, `SUM()`) in the database. The client should only fetch the final computed numbers.

2. **Missing Database Indexing (`server/index.ts` and `DATABASE_SETUP.md`):**
   - While filtering on `shop_id` and `status = 'fulfilled'`, these columns could slow down large queries.
   - **Fix Suggestion:** Ensure a composite database index exists on `(shop_id, status)` for the `orders` table to speed up dashboard queries dramatically.

## 3. Palette (The UX Consultant) - UX Enhancement Report
**Scope:** `SendFlow.tsx` and `OrderSummary.tsx`

### Findings:
1. **Missing ARIA Labels for Icon Buttons (`SendFlow.tsx` & `OrderSummary.tsx`):**
   - **Issue:** Navigation icon buttons, such as the back arrow (`<Button variant="ghost" size="icon" ...><ArrowLeft /></Button>`) or the edit button, lack text context. Screen readers will read them ambiguously.
   - **Fix:** Add descriptive `aria-label` attributes to these buttons (e.g., `aria-label="Go back to shop"`, `aria-label="Edit order details"`).

2. **Form Accessibility and Error Linking (`SendFlow.tsx`):**
   - **Issue:** Input fields use `aria-invalid` correctly, but the error messages rendered beneath them (e.g., "Recipient name is required") are not programmatically linked to the inputs.
   - **Fix:** Assign `id`s to the error message paragraphs and link them using `aria-describedby` on the respective `<Input>` or `<Textarea>` elements.

3. **Inadequate Loading States during Payment Handshake (`OrderSummary.tsx`):**
   - **Issue:** During the `handlePayNow` API call (which interacts with `server/index.ts` and Flutterwave), the button changes to a spinner, but the page doesn't fully indicate that a background process is locking navigation. Users might think the app is frozen and attempt to navigate away or click multiple times.
   - **Fix:** Introduce an `aria-live="polite"` region to announce "Initializing payment, please wait". Furthermore, consider adding a subtle full-screen semi-transparent overlay to prevent interaction elsewhere on the page while `creating` is true, ensuring a "delightful" and reliable UX.
