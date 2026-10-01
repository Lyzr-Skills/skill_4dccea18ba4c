---
name: kpmg-bid-negotiation-flow
description: End-to-end test for the bid negotiation query flow — login, open the RFQ Journey sidebar, filter to an RFQ in the "In Negotiation" status, ask for a negotiation analysis, then keep answering follow-up questions the user wants to ask until they say stop.
---

# Name

```
bid-negotiation-query
```

# Description

```
End-to-end test for the bid negotiation query flow — login, open the RFQ Journey sidebar, filter to an RFQ in the "In Negotiation" status, ask for a negotiation analysis, then keep answering follow-up questions the user wants to ask until they say stop.
```

# Instructions

```
# Skill: Bid Negotiation Query

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
1. There is a narrow (~60px wide) icon rail fixed to the left edge of the screen, with a few icons stacked vertically (Home icon first, then a **compass icon** right below it). **Click the compass icon** — this is a CLICK, not a hover (hovering it only shows a "Journey" tooltip; the click is what actually opens the panel).
2. Call `browser_read` to confirm.
3. **Assert**: page text now contains "RFQ Journey".
4. Once open, the panel auto-closes about 250ms after the mouse leaves it — keep your next actions (status filter, row click) targeted at elements inside the panel rather than moving to something else in between. If a later step fails because the panel closed, click the compass icon again before retrying.

### Step 5 — Filter to "In Negotiation" and pick an RFQ
1. Call `browser_select_option` on the status filter — selector `select[aria-label="Filter by status"]` — with value/label `"In Negotiation"`.
2. Selecting a status always triggers a real new network request (it is not a client-side filter over already-loaded data), so the list will show "Loading..." again — poll for it to clear rather than assuming it's instant. Repeat up to 8 times (about 40 seconds total): call `browser_read`; if it still shows "Loading..." call `browser_wait` on `body` with `timeoutMs: 5000` and poll again; stop once "Loading..." is gone.
3. If it is STILL showing "Loading..." after ~40 seconds, this is a known possible backend issue, not something you can fix by clicking around: the app's RFQ-list fetch has no timeout or retry, so if the backend request stalls, the UI can hang on "Loading..." indefinitely with no recovery. You may try re-opening the panel (click the compass icon, wait again) ONCE as a single sanity check, but do not loop on this indefinitely. If it's still stuck after that, report **BLOCKED** at this step: "RFQ list stuck on Loading after filtering to In Negotiation — likely a backend non-responsiveness issue (the frontend has no fetch timeout), not a browser-automation problem."
4. Once the list has loaded, call `browser_read` to see the filtered list.
5. **Assert**: at least one RFQ is shown. If the list loaded but is genuinely empty, report **BLOCKED** — there is no RFQ in `in_negotiation` status to test against in this environment.
6. If there are 2 or more RFQs, call `ask_user` with the question "Here are the RFQs currently in negotiation — which one should I test?" and `options` set to the actual RFQ titles you just read (up to 4; if there are more than 4, list the first 4). If there is exactly one, use it directly without asking.
7. **This panel only stays open while it's being interacted with — it auto-closes about 250ms after it stops being hovered, and a full round-trip to decide your next action is almost always slower than that, so treat it as already closed by the time you act.** Right before clicking the row:
   - Call `browser_hover` on the status filter (or any stable element inside the panel) first, to make sure the panel is actually open right now.
   - Call `browser_evaluate` with a script like `[...document.querySelectorAll('button.w-full')].map(e => e.innerText)` to get each row's CURRENT exact text (don't reuse text you read several steps ago — match the exact casing shown, e.g. "In Negotiation" with capitals, not "in negotiation").
   - Build the click selector from an exact, currently-true substring (e.g. the RFQ's title/number, which is unique) — don't chain multiple `:has()`/`:text-is()` conditions guessing at exact wording, one precise fragment is more robust than several guesses.
   - Call `browser_click` immediately after the above, with no other calls in between.
8. If the click fails with "no element found" / 0 matches, don't try a longer or more exotic selector — that almost always means the panel already closed, not that the selector was wrong. Click the compass icon again, re-apply the status filter (step 1 of this section), and retry the row click immediately.
9. Call `browser_read`.
10. **Assert**: the page no longer shows the RFQ list — it now shows the RFQ workspace (look for section/tab names such as "Track", "Negotiation Setup", "Bid Comparison", "Bid Response Analysis", "Ranking", or the RFQ's own title).

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
Steps 4, 5, 6 pass (an RFQ in `in_negotiation` was found and opened, and the first analysis query produced a real, non-empty response) = test passes. Step 3 may be BLOCKED by OTP timeout, and Step 5 may be BLOCKED by the RFQ list hanging on Loading — note either clearly. Step 7 isn't pass/fail — it's interactive follow-up driven entirely by the user.

## Report format
After the user says stop, report:
```
## Bid Negotiation Query Test
**Verdict: PASS / FAIL / BLOCKED**

- [PASS/FAIL/BLOCKED] Step 1 — Login page loaded
- [PASS/FAIL/BLOCKED] Step 2 — OTP requested
- [PASS/FAIL/BLOCKED] Step 3 — Reached main app
- [PASS/FAIL] Step 4 — RFQ Journey sidebar opened (compass icon clicked)
- [PASS/FAIL/BLOCKED] Step 5 — RFQ in "In Negotiation" status found and opened (name it; note if blocked by a stuck Loading state)
- [PASS/FAIL] Step 6 — Negotiation analysis query answered
- Follow-up questions asked: [list each question the user asked and a one-line summary of the response]
- [PASS/FAIL] Step 8 — No unexpected console/network errors

### Issues Found
[List any FAIL items with what was expected vs. what was seen]

### Screenshots
[List screenshot file paths taken]
```
```
