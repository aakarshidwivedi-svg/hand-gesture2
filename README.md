# AI Usage

A quiet, dense command center for ChatGPT, Claude, Gemini and Microsoft Copilot usage — built as a Chrome (Manifest V3) extension. No build step: it's plain HTML/CSS/JS, load it unpacked.

## Install (Chrome / Edge / Brave)

1. Go to `chrome://extensions`.
2. Turn on **Developer mode** (top right).
3. Click **Load unpacked** and select this folder.
4. Pin the extension, then open ChatGPT, Claude, Gemini or Copilot in a tab and send a message.
5. The toolbar icon badge updates with your message count for the day. Click the icon for the popup, or "Open dashboard" for the full view.

## What's real vs. estimated

Same honesty policy as before, now applied to two kinds of numbers instead of one:

- **Message counts are real.** A content script watches each provider's own conversation DOM and counts a new message the moment it actually renders — it does not poll or guess. Reloading a page or scrolling through old history doesn't inflate the count: on load, the extension baselines whatever's already on screen and only counts turns added *after* that.
- **Token counts are estimates, always labeled as such ("~1.2k tokens est.").** None of these four providers expose a real tokenizer count in their web app — there's no endpoint that returns "this reply was 214 tokens." The estimate uses the same rough character-to-token ratio (`chars ÷ 4`) that OpenAI and Anthropic both cite for English text. It's a reasonable ballpark, not a billing-accurate number, and the UI never presents it as one.
- **The old "usage limit" banner detection is still here too.** When a provider shows something like "you've hit your limit" or "12 of 40 messages," that text is picked up and shown as a small tag next to the provider's row — a real signal from the provider, not derived from the message counter.

If a provider ever ships a real usage or token API, `src/content/<provider>.js` is exactly where to wire it in — each scraper is isolated so swapping DOM-scraping for a real fetch doesn't touch anything else. Selectors for message detection are best-effort and can drift when a provider redesigns its UI; each content script falls back to a generic text-shape heuristic if its primary selectors stop matching, and under-counts rather than guesses when even that fails.

## Toolbar badge

Settings → Toolbar icon lets you choose what the number on the extension icon means: today's total messages across all four tools (default), your all-time total, or off. It updates the moment a new message is detected, and on a 1-minute timer so the "today" count correctly resets at midnight.

## Architecture

```
manifest.json
src/
  background/background.js   service worker: message routing, staleness, notifications
  content/                   one scraper per provider + shared helpers, runs in-page
  popup/                     the compact popup (main UI)
  dashboard/                 expanded window: history chart, ranking, calendar
  options/                   settings page
  onboarding/                first-run screen
  utils/                     storage, formatting, provider registry, insight logic
  styles/                    design tokens + base component styles
```

Data flow: each content script (`shared.js` + one provider file) watches its page's conversation DOM with a `MutationObserver`, dedupes against messages it's already seen, and sends one event per *new* message to the background service worker. The worker increments that provider's counters in `chrome.storage.local`, buckets them by calendar day for history, and updates the toolbar badge. The popup and dashboard don't poll — they read storage once and then re-render live via `chrome.storage.onChanged`.

A lightweight heartbeat (every 20s while a tab is visible) separately marks a provider **active**. A 1-minute alarm re-evaluates status from timestamps alone: **stale** if nothing in 5 min, **offline** if nothing in 20 min — without ever discarding the counts themselves.

## Privacy

Everything stays local. No network requests leave the browser, no credentials or cookies are read, and history/settings live in `chrome.storage.local` only. The **Export data** button in Settings gives you your own stored history as JSON.
