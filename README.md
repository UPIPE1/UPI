# Splitr — UPI Bill Splitter (PWA)

A mobile-first, installable Progressive Web App that lets anyone scan a merchant's
UPI QR code, split a bill into multiple payments under a configurable limit
(₹2,000 by default), and pay each part with whatever UPI app they already have
installed.

**Fully static. No backend, no database, no login, no analytics.** Everything
(scanning, splitting, payment history) runs in the browser. Local split history
is stored only in `localStorage` on the user's own device and is never sent
anywhere.

## Files

```
upi-splitter/
├── index.html      # the entire app: HTML + CSS + JS in one file
├── manifest.json   # PWA manifest (name, icons, theme colors)
├── sw.js           # service worker — caches the app shell for offline/fast loads
├── icons/
│   ├── icon-192.png
│   ├── icon-512.png
│   └── icon-512-maskable.png
└── README.md
```

## How it works

1. **Scan** — camera opens (via the [html5-qrcode](https://github.com/mebjas/html5-qrcode)
   library, loaded from a CDN) and reads the QR code, or the user can upload a
   photo/screenshot of a QR code instead (`scanFile`, same library, still
   entirely client-side). Two payment QR formats are supported — see below.
   A manual entry field is always available as a fallback for QR codes that
   won't scan.
2. **Amount** — the user types the total bill amount and can adjust the
   per-payment limit (defaults to ₹2,000).
3. **Split** — the app computes the minimum number of payments needed so that
   every payment is ≤ the limit and all payments sum exactly to the bill.
4. **Pay** — tapping "Pay" on any split opens a `upi://pay?...` deep link with
   that split's exact amount pre-filled, handing off to whichever UPI app
   (GPay, PhonePe, Paytm, BHIM, etc.) the user has installed. The web app
   **never** touches PINs, OTPs, or bank credentials — it only launches the
   intent and lets the native app take over.
5. **Confirm** — because a static site cannot verify payment success (UPI
   doesn't expose that to web pages), the user marks each split as *Paid* or
   *Failed* themselves after completing it in their UPI app.
6. **Done / History** — a summary is stored locally so the user can review past
   splits. There is a "Clear history" option in the History screen.

## Supported QR formats

Both are parsed entirely client-side, and neither path ever invents a UPI ID
that wasn't actually present in the scanned data — if no payee address is
found, the scan is rejected rather than guessed at.

**1. UPI deep-link QR** — `upi://pay?pa=merchant@bank&pn=Name&am=..&cu=INR`
(and the equivalent query parameters under other schemes some apps use). This
covers most app-generated "Scan & Pay" QR codes, including ones with no
amount (`am` omitted) and ones with a fixed amount baked in.

**2. EMVCo / BharatQR "Merchant Presented Mode" QR** — the TLV
(tag-length-value) format behind BharatQR and most bank- or POS-issued static
and dynamic UPI stickers, recognizable by its `000201...` prefix. Because
different banks place the UPI VPA under different sub-tags inside the
merchant account information block (tags 02–51), the parser searches all of
them for a value that looks like a valid VPA (`name@handle`) instead of
assuming a fixed tag number — this is what makes it work across different
banks' and merchants' QR implementations. From this format the app also
extracts, when present:
- Merchant name (tag 59)
- Transaction amount, for dynamic QR codes (tag 54) — pre-fills the amount
  field but the user can still change it
- Currency (tag 53) — flagged if it isn't INR
- CRC checksum (tag 63) — verified as a best-effort integrity check; a
  mismatch shows a warning but doesn't block the scan

If a QR code matches neither format, or an EMVCo-shaped code has no VPA
anywhere in it, the app shows "doesn't look like a supported UPI/payment QR
code" rather than fabricating a destination.

## Split algorithm

```js
function splitBill(totalRupees, limitRupees) {
  const totalPaise = Math.round(totalRupees * 100); // avoid float errors
  const limitPaise = Math.round(limitRupees * 100);
  const parts = [];
  let remaining = totalPaise;
  while (remaining > limitPaise) {
    parts.push(limitPaise);
    remaining -= limitPaise;
  }
  if (remaining > 0) parts.push(remaining);
  return parts.map(p => p / 100);
}
```

All arithmetic is done in integer paise to avoid floating-point drift, then
converted back to rupees for display. Two invariants are asserted in code
before the result is used:

- `sum(parts) === totalPaise`
- every `part <= limitPaise`

If either check ever fails, the app raises an error instead of silently
producing a bad split.

### Verified test cases (limit = ₹2,000)

| Bill    | Result              | Sum matches? | All ≤ limit? |
|---------|---------------------|:---:|:---:|
| ₹500    | ₹500                 | ✅ | ✅ |
| ₹2,000  | ₹2,000               | ✅ | ✅ |
| ₹2,001  | ₹2,000 + ₹1          | ✅ | ✅ |
| ₹3,999  | ₹2,000 + ₹1,999      | ✅ | ✅ |
| ₹4,000  | ₹2,000 + ₹2,000      | ✅ | ✅ |
| ₹4,500  | ₹2,000 + ₹2,000 + ₹500 | ✅ | ✅ |

Also spot-checked: amounts under ₹1, amounts with paise (e.g. ₹2,000.50), and
large bills (₹12+ lakh, which correctly produces hundreds of ₹2,000 chunks
plus a remainder).

Note: this is a **greedy** split (fill full limit-sized chunks, remainder
last), which is what produces the ₹2,000 + ₹500 pattern above rather than an
even 3-way split. If you'd prefer balanced splits instead (e.g. ₹1,500 ×
3 for a ₹4,500 bill), that's a one-function change in `splitBill` — ask and
I can add it as a toggle.

## Deploying to GitHub Pages

1. Create a new GitHub repository (or use an existing one).
2. Copy all the files in this folder (`index.html`, `manifest.json`, `sw.js`,
   `icons/`) into the repo root — or into a `/docs` folder if you prefer that
   Pages source.
3. Commit and push:
   ```bash
   git add .
   git commit -m "Add Splitr UPI bill splitter PWA"
   git push
   ```
4. In your repo on GitHub: **Settings → Pages → Build and deployment → Source**,
   choose "Deploy from a branch", pick `main` (or your default branch) and the
   `/ (root)` folder (or `/docs` if you used that), then Save.
5. GitHub will give you a URL like `https://<username>.github.io/<repo>/`.
   Open it — this **must be HTTPS** (which GitHub Pages provides automatically)
   for camera access to work.
6. Optional: on Android Chrome, visit the site and use "Add to Home screen" to
   install it as a standalone app using the manifest and icons included here.

No build step, bundler, or `npm install` is required — it's plain static
files.

## Testing locally

Camera access requires a secure context, so opening `index.html` directly
with `file://` won't allow scanning (manual entry will still work). Serve it
over `localhost` instead, e.g.:

```bash
cd upi-splitter
python3 -m http.server 8000
# then open http://localhost:8000 on your phone via your computer's LAN IP,
# or use http://localhost:8000 in a desktop browser (camera permitted on localhost)
```

To test on a phone without deploying, use a tunnel like `ngrok` or `cloudflared`
so you get a public HTTPS URL for camera access.

## Security & privacy notes

- No UPI PIN, OTP, password, or banking credential is ever requested,
  displayed, or stored by this app — payments are completed entirely inside
  the user's own UPI app after the deep link hands off.
- No network calls are made except to load the app itself, Google Fonts, and
  the QR-scanning library — none of them ever see the bill amount, UPI ID, or
  payment status.
- Split history lives only in the browser's `localStorage`; there is no
  server component to breach, and clearing browser data removes it entirely.
- Because there's no backend, this app cannot and does not claim to verify
  whether a UPI payment actually succeeded — that confirmation is always
  manual, by design, per the brief.
