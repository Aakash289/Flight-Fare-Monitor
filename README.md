# Flight-Fare-Monitor

Flight Fare Monitor is an n8n-based automation that watches the flight routes in your database “watchlist” and sends a Slack alert when prices drop by at least 20% of their price.
It uses the Kiwi.com Tequila API to fetch current fares, compares them against your stored last_price, and alerts only when the savings are meaningful.

The problem it solves

Airline prices are volatile, they can change multiple times per day. Typical booking sites don’t proactively tell you when a fare on your exact route and dates gets cheaper. That leads to:

Constant manual re-checking (easy to forget, easy to miss dips).

Missed savings when short-lived price drops come and go.

Stale prices on landing pages or internal dashboards unless someone refreshes them.

Flight Fare Monitor solves this by polling every hour, matching results to your watchlist, computing the percent drop vs. your reference last_price, and pinging you only when the drop ≥ threshold.

How the workflow works


High-level flow

Schedule Trigger (hourly)

  → Watchlist (Postgres)
  
  → Build Tequila URL (Code)
  
  → HTTP — Tequila Search
  
  → Merge (item + response)
  
  → Normalize & Compute Drop (Code)
  
  → IF drop ≥ threshold?
  
      → IF has Slack URL?
      
          → HTTP — Slack Webhook
          
              → HTTP — Refresh Landing (optional)


Node-by-node explanation

Schedule Trigger — every 1h
Kicks off the run once per hour (you can change the cadence).

Watchlist (Postgres)
Runs a SELECT to fetch each route/date you’re tracking (e.g., origin, destination, departure_date, return_date, round_trip, adults, bags, max_stops, cabin, currency, threshold_pct, last_price, slack_webhook_url, landing_url).

Build Tequila URL (Code)
Builds a Kiwi Tequila /v2/search URL per row: converts dates to DD/MM/YYYY, sets one-way/round-trip, cabin, stops, passengers, currency, etc.
Output: tequila_url (and often tequila_params for debugging).

HTTP — Tequila Search
Calls Tequila with header apikey = {{$env.TEQUILA_API_KEY}}, response = JSON. Returns itineraries sorted by price (cheapest first).

Merge (item + response) (mergeByPosition)
Combines the original watchlist row (Input 1) with the API response (Input 2) so downstream nodes see both.

Normalize & Compute Drop (Code)

Picks the cheapest itinerary (data[0])

Adds baggage cost if bags > 0 (so totals are apples-to-apples)

Computes:

current_price (base + bags)

drop_pct = ((last_price - current_price) / last_price) * 100

is_drop = (drop_pct ≥ threshold)

Prepares alert_text and includes Tequila deep_link (when present)

About the threshold:

By design, it’s 20% by default.

You can override per row via threshold_pct, or globally via an env var (e.g., DEFAULT_THRESHOLD_PCT), or fix it in code if you prefer it hard-set.

IF — drop ≥ threshold?
Continues only when is_drop is true.

IF — has Slack URL?
Uses a row’s slack_webhook_url if present; otherwise falls back to a global SLACK_WEBHOOK_URL env var. If neither is set, it skips sending.

HTTP — Slack Webhook
Posts a concise alert message (route, dates, old vs new price, percent drop, and deep link).

HTTP — Refresh Landing (optional)
If your row provides a landing_url, this node pings it with ?refresh=1 so your public page or cache can update after an alert.
