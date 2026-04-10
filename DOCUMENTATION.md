# DuetIncome — Program Documentation

**Last updated:** 2026-04-10
**Live URL:** https://duetbooksapp.github.io/duetincome/
**Repo:** duetbooksapp/duetincome

## What DuetIncome is

A standalone single-file HTML tool for AAA Life agents to compare fixed-income
products (CDs, annuities, S&P index strategies) side-by-side for clients. Used
during client conversations to show the math and handle objections.

Everything — HTML, CSS, JavaScript, PWA manifest — is embedded in one file
(`DuetIncome.html`, ~131 KB). There is no build step, no external JavaScript,
no CDN dependencies. Open the file in a browser and it runs.

## Tabs (features)

### 1. CD → Annuity
Walks a client through the advantages of converting a CD to a fixed annuity.

- **Inputs:** CD term, deposit amount, annuity product
- **Year-by-year CD APY rates:** one input box per year of the annuity term.
  Lets you model the real scenario where CD rates may drop (or rise) over a
  5-year rollover window rather than assuming one flat rate for the whole
  period. Click "Save CD Rates" to persist.
- **Annuity illustration rates:** one input per year, mirroring the CD side.
  Click "Save Rates" to persist.
- **Year-by-Year Growth Comparison table:** compounds each side independently
  using its own year-by-year rates, shows after-tax CD earnings alongside
  tax-deferred annuity earnings.
- **Feature comparison table:** CD vs annuity on taxes, liquidity, death
  benefit, FL Medicaid asset protection, etc.
- **Quick Reference — Why Annuity Wins:** grid of key advantages.
- **Florida Medicaid section:** asset-protection talking points and
  disclaimers.
- **Email Comparison to Client button:** generates an HTML + plain-text
  comparison, copies to clipboard, opens Gmail compose window. Branded with
  the signed-in agent's name and branch.

### 2. Annuity vs Annuity
Side-by-side comparison of any two annuity products in the agent's book.

- Inputs: deposit amount, client age, risk tolerance, money source, goal,
  Product A, Product B
- Per-product year-by-year illustration rates
- Per-product feature editor (term, strategies, guaranteed minimum, bonus,
  free withdrawal, surrender schedule, death benefit, MVA, issuer, best-for,
  age limit, illustration)
- Defaults seeded for `MM Landmark 5 External` and `Platinum Bonus 5 (AAA Life)`
- Email-to-client export

### 3. S&P vs Annuity
Historical-return comparison tool using actual S&P 500 data 1990–2025.

Two scenarios side-by-side:
- **Scenario A:** 100% market-based allocation (% S&P / % bonds, withdrawal rate)
- **Scenario B:** partial annuity allocation (% S&P / % bonds / % annuity,
  withdrawal rate)

Shows year-by-year balances, withdrawal cash flow, and how many down years
the annuity floor protected the client from market losses.

### 4. Conversations
Scripts for client conversations. Has no calculator inputs.

- **Guided Conversation:** 5 collapsible phases (Situation, Problem Awareness,
  Solution Awareness, Consequence, Commitment) with question prompts and
  "why this works" notes
- **Handling: "It Ties Up My Money":** 5 common CD objections with written
  responses. The "What if I need the money?" response pulls the deposit
  amount from the CD → Annuity tab to show the 10% free-withdrawal number
  in real dollars.

## Authentication and multi-user sync

### How login works

- **Name + PIN login** (no email, no password reset flow)
- Agents self-register from the login screen: Full Name + Branch + 4-6 digit PIN
- PIN is SHA-256 hashed in the browser before being sent anywhere
- Profile and data live at:
  `https://duet-crm-default-rtdb.firebaseio.com/duetincome/{agent_id}/`
- Agent ID is derived from name: `Steve Maxim` → `steve_maxim`
- `sessionStorage` keeps you signed in during a browser session; reload stays
  signed in, close tab signs you out
- `localStorage` remembers the last agent's name to prefill the login form

### What syncs

Per-agent data (namespaced under the agent ID, synced to Firebase on every save):

| Data | Firebase key | Local cache key |
|---|---|---|
| Annuity illustration rates (CD tab) | `cd_illustration_rates` | `doi_cd_rates_{id}` |
| CD yearly APY rates | `cd_yearly_rates` | `doi_cd_yearly_rates_{id}` |
| Annuity illustration rates (AvA tab) | `ava_illustration_rates` | `doi_ava_rates_{id}` |
| Annuity product features | `annuity_features` | `doi_annuity_features_{id}` |
| S&P comparison inputs | `duet_idx_inputs2` | `doi_idx_inputs_{id}` |

Local-only (not synced):
- Dark mode preference (`doi_dark_mode`) — per-device
- Remembered agents list (`doi_remembered_agents`) — per-device prefill convenience

### One-time data migration

On first login/register, if an agent still has data in the old un-prefixed
localStorage keys (`cd_illustration_rates`, `ava_illustration_rates`,
`annuity_features`, `duet_idx_inputs2`), it is copied into the new per-agent
namespace and pushed to Firebase. This is only relevant for Steve's original
browser where the pre-auth version was used.

### Honest security notes

This is a friendly gate, not a vault. Be aware:

- The Firebase Realtime Database has no access rules — anyone who knows the
  URL can read or write any agent's data directly via `curl`
- The PIN check runs in the browser; someone with DevTools can bypass it
- 4-6 digit PINs are brute-forceable (only 10,000–1,000,000 possibilities)
- Data is stored in plaintext on Firebase

This is acceptable because DuetIncome contains no personally identifiable
client information — just sales tools, illustration rates, and product
features. Do **not** enter real client names, account numbers, or any PII
into this app.

## Architecture

### File structure

```
DuetIncome.html (single file)
├── <head>
│   ├── <meta> tags (PWA manifest embedded as data URI)
│   └── <style> — all CSS (topbar, tabs, cards, login, dark mode, mobile)
└── <body>
    ├── #loginScreen (visible by default, hidden after login)
    ├── #appShell (hidden by default, shown after login)
    │   ├── .topbar (agent name/branch, sync indicator, dark mode, logout)
    │   ├── .tabs (4 tab buttons)
    │   └── .panel#p0-p3 (one per tab)
    ├── #toastBox
    └── <script>
        ├── Core utilities ($, $1, fmt, showToast)
        ├── Authentication + Firebase sync (sha256, fbSave, fbLoad,
        │   doLogin, doRegister, doLogout, enterApp, migrateLegacyLocalData,
        │   loadAgentData)
        ├── Dark mode
        ├── Tab system (showTab)
        ├── Settings, products, helpers
        ├── Tab 1: CD → Annuity (renderCDConversion, generateClientComparison,
        │   saveCDInputs, saveIllustrationRates, saveCDYearlyRates)
        ├── Tab 4: Conversations (renderConversations)
        ├── Tab 2: Annuity vs Annuity (renderAnnuityVsAnnuity, saveAvaRates,
        │   saveAnnuityFeature, etc.)
        ├── Tab 3: S&P vs Annuity (renderIndexingTool, saveIdxInputs)
        └── Init IIFE (restores session if any, else shows login)
```

### No external scripts

The entire app runs without loading anything from the network, with two
exceptions:

1. **Firebase Realtime DB REST API** — plain `fetch()` calls to save/load data
2. **Fonts** — system fonts (`Inter`, `system-ui`, `-apple-system`)

No Firebase SDK, no React, no jQuery, no build tooling. This keeps the app
extremely fast to load and trivial to deploy (just copy the HTML file).

## Development workflow

### Source of truth

```
~/.duet-server/DuetIncome.html
```

Served locally by `~/.duet-server/duet_server.py` (Python http.server on
port 8787).

**Local URL:** http://localhost:8787/DuetIncome.html

### Deploy to GitHub Pages

When `DuetIncome.html` is updated:

```bash
cp ~/.duet-server/DuetIncome.html ~/Desktop/DuetIncome/DuetIncome.html
cp ~/.duet-server/DuetIncome.html ~/Desktop/DuetIncome/index.html
cd ~/Desktop/DuetIncome
git add -A
git commit -m "Describe the change"
git push
```

GitHub Pages updates ~30-60 seconds after push.

**Live URL:** https://duetbooksapp.github.io/duetincome/

### Pushing code updates to all agents

Any change pushed to GitHub Pages is automatically delivered to every agent
on their next page load. No install, no update notification — they just
reload. The per-agent Firebase data is unaffected by code updates.

### Testing auth/sync changes

1. Register or sign in with a test account
2. Make a change (e.g., save a CD yearly rate)
3. Verify the sync indicator flashes amber → green
4. Check Firebase directly: `curl 'https://duet-crm-default-rtdb.firebaseio.com/duetincome/{agent_id}/cd_yearly_rates.json'`
5. Open an incognito window, sign in, verify data loads from Firebase

## Related apps (separate projects)

DuetIncome is intentionally isolated from the other "Duet" tools:

- **DuetCRM** — `~/.duet-server/DuetCRM.html` → smaxim-spec/duet
- **DuetBooks** — `~/Desktop/DuetBooks/` → duetbooksapp/duetbooks
- **DuetCoach**, **DuetWithClaude** — `~/Desktop/Duet/`

DuetIncome shares Firebase infrastructure with DuetCRM/DuetBooks (same
`duet-crm-default-rtdb` database, different top-level key `/duetincome/`)
but is otherwise independent.

## Known limitations

- No password reset flow (forget your PIN = register under a different name
  or have someone delete your Firebase profile manually)
- Firebase rules are wide-open — see "Honest security notes" above
- No automated tests
- No phone-specific QA done yet — layout works at mobile widths but has not
  been exhaustively tested on iOS/Android
- Dark mode is per-device (not synced across your own devices)
- No export/import feature — if you lose access to your agent account, data
  in Firebase is still there but the only way to recover it is via the
  Firebase REST API directly

## Future options (not implemented)

- **Export/Import JSON** — self-service backup and restore, would make PIN
  loss less catastrophic
- **Firebase security rules** — per-agent read/write restrictions so agents
  can only touch their own namespace
- **Reference data namespace** — if AAA publishes official product rates
  that should be the same for everyone, add a shared `/duetincome/_shared/`
  path for those, keep per-agent overrides
- **Multi-device sync for dark mode** and other UI prefs — currently local
