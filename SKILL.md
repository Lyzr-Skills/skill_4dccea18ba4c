---
name: kpmg-bid-negotiation-flow
description: End-to-end test for the bid negotiation query flow — login, open the RFQ Journey sidebar, filter to an RFQ in the "In Negotiation" status, ask for a negotiation analysis, then keep answering follow-up questions the user wants to ask until they say stop.
---

## What this skill tests
A buyer logs in, opens the RFQ Journey sidebar, filters to RFQs in the "In Negotiation" status, picks one, and asks the agent to analyze the negotiation via the main chat. This skill verifies that flow end-to-end, then keeps asking follow-up questions on the user's behalf until they say to stop.

## Pre-conditions
- If credentials are empty, use `arsalan@lyzr.ai` as the email (OTP-based login).
- Target RFQ status for this test is **`in_negotiation`** (UI label "In Negotiation") — not any other status.

---

## Test steps

### Step 1 — Start browser and go to login
1. Call `browser_start` with `headless: false` so you can observe.
2. Call `browser_navigate` to the site's `/login` page.
3. Call `browser_read` to confirm the login page loaded.
4. **Assert**: page contains an email input (`input#email` or `input[type="email"]`).

### Step 2 — Enter email and request OTP
1. Call `browser_fill` on `input#email` (or `input[type="email"]`) with `arsalan@lyzr.ai`.
2. Call `browser_click` on the submit button (text "Continue").
3. Call `browser_read`.
4. **Assert**: page is now on `/verify-otp`.

### Step 3 — Wait for the user to log in (DO NOT ASK FOR OTP — JUST WAIT)

> CRITICAL: The browser window is visible on screen (headless: false). The user will manually enter the OTP themselves (it's 6 separate single-digit boxes, then "Verify & Login"). Your job is to WAIT silently — do NOT stop, do NOT ask the user for the OTP code, do NOT report blocked. Just keep polling until login completes.

Poll loop — repeat up to 24 times (2 minutes total):
1. Call `browser_read`.
2. Check the URL in the result:
   - Still `/verify-otp` or `/login` → call `browser_wait` on `body` with `timeoutMs: 5000`, then poll again.
   - Changed to anything else (e.g. `/`) → login succeeded, continue to Step 4.
3. If after 24 polls still on `/verify-otp` → mark Step 3 TIMEOUT and stop the test.

**Do not ask the user anything during this wait. Just poll silently.**

### Step 4 — Open the RFQ Journey sidebar
1. The RFQ list lives in a hover-triggered side panel titled "RFQ Journey", not a click-to-open page — there's no direct URL for it. Call `browser_hover` on the slim nav rail/icon at the left edge of the screen (try a left-edge icon or strip first; if unsure which element, call `browser_get_html` and look for a collapsed nav element near the left edge, or an icon with a title/tooltip containing "Journey" or "RFQ").
2. Call `browser_read` to confirm.
3. **Assert**: page text now contains "RFQ Journey".
4. If a later step's click/select seems to fail because the panel closed, re-`browser_hover` the same trigger before retrying — this panel can close when the mouse leaves its trigger area.

### Step 5 — Filter to "In Negotiation" and pick an RFQ
1. Call `browser_select_option` on the status filter — selector `select[aria-label="Filter by status"]` — with value/label `"In Negotiation"`.
2. Call `browser_read` to see the filtered list.
3. **Assert**: at least one RFQ is shown. If none, report **BLOCKED** — there is no RFQ in `in_negotiation` status to test against in this environment.
4. If there are 2 or more RFQs, call `ask_user` with the question "Here are the RFQs currently in negotiation — which one should I test?" and `options` set to the actual RFQ titles you just read (up to 4; if there are more than 4, list the first 4). If there is exactly one, use it directly without asking.
5. Call `browser_click` on the row matching the chosen title — scope the selector to that exact row/text, not a bare tag, since the list can contain several rows.
6. Call `browser_read`.
7. **Assert**: the page no longer shows the RFQ list — it now shows the RFQ workspace (look for section/tab names such as "Track", "Negotiation Setup", "Bid Comparison", "Bid Response Analysis", "Ranking", or the RFQ's own title).

### Step 6 — Ask for the negotiation analysis
1. Find the chat input. It is the single global composer used for everything in this app — a `<textarea>` with placeholder `"How can I help?"` (it may briefly show `"Please wait..."` if the agent is mid-reply — wait it out before typing).
2. Before sending, call `browser_evaluate` with script `document.querySelectorAll('.animate-slide-in-left').length` and remember this number as **N** (the count of existing agent-reply bubbles).
3. Call `browser_fill` on the textarea with: `Give me an analysis of the negotiation for this RFQ.`
4. Submit by pressing Enter (`browser_press_key` with `key: Enter`, `selector` the textarea) or clicking the icon-only submit button (`button[type="submit"]`).
5. Poll up to 12 times (60 seconds total): call `browser_evaluate` with the same count script; stop polling once the count is greater than **N**, waiting 5s (`browser_wait` on `body`, `timeoutMs: 5000`) between checks.
6. Once the count increased, call `browser_evaluate` with script `[...document.querySelectorAll('.animate-slide-in-left')].at(-1)?.innerText` to read the newest reply's text.
7. **Assert**: the text is non-empty and isn't just a loading/placeholder string.
8. Call `browser_screenshot`.

### Step 7 — Keep asking whatever else the user wants, until they say stop
Repeat this loop until the user stops it:
1. Call `ask_user` with the question: "Here's the response above. What else would you like me to ask about this RFQ's negotiation? Reply with your next question, or choose Stop to finish." and `options: ["Stop — finish the test"]`.
2. If the user's reply is the Stop option (or clearly means stop/done/no more) → break the loop and go to the report.
3. Otherwise, treat their reply as the literal next question. Repeat Step 6's send/poll/read/assert sequence (steps 2–8) using the user's text instead of the fixed analysis prompt.

### Step 8 — Console/network health check
1. Call `browser_get_console_errors`.
2. Call `browser_get_failed_requests`.
3. **Assert**: nothing unexpected shows up (ignore benign third-party noise, flag anything referencing the app's own API/assets).

---

## Pass criteria
Steps 4, 5, 6 pass (an RFQ in `in_negotiation` was found and opened, and the first analysis query produced a real, non-empty response) = test passes. Step 3 may be BLOCKED by OTP timeout — note it clearly. Step 7 isn't pass/fail — it's interactive follow-up driven entirely by the user.

## Report format
After the user says stop, report:
```
## Bid Negotiation Query Test
**Verdict: PASS / FAIL / BLOCKED**

- [PASS/FAIL/BLOCKED] Step 1 — Login page loaded
- [PASS/FAIL/BLOCKED] Step 2 — OTP requested
- [PASS/FAIL/BLOCKED] Step 3 — Reached main app
- [PASS/FAIL] Step 4 — RFQ Journey sidebar opened
- [PASS/FAIL/BLOCKED] Step 5 — RFQ in "In Negotiation" status found and opened (name it)
- [PASS/FAIL] Step 6 — Negotiation analysis query answered
- Follow-up questions asked: [list each question the user asked and a one-line summary of the response]
- [PASS/FAIL] Step 8 — No unexpected console/network errors

### Issues Found
[List any FAIL items with what was expected vs. what was seen]

### Screenshots
[List screenshot file paths taken]
```
```
