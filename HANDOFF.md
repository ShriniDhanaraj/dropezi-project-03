# DROPeZi Website — Chat Handoff (Session 4)

## Project Overview
**Brand:** DROPeZi — One-Way Drop Taxi, South India  
**Goal:** Single-file SPA (HTML + CSS + JS inline, no build step)  
**GitHub Pages (dev/staging):** https://shrinidhanaraj.github.io/dropezi-project-03/  
**Live domain (production):** https://dropezi.in (active, collecting feedback)  
**dropezi.com:** Currently redirects → dropezi.in. Plan: point it to GitHub Pages (project-03) independently.

---

## SECURITY RULES — NEVER BREAK THESE

1. **NEVER touch `dropezi-website.web.app`** (Firebase — built with Gemini, kept for future reference for login/registration/loyalty/tracking features)
2. **dropezi.in is live** — do not break or redirect it while it is still serving real users
3. **Real WhatsApp/Phone number:** `919876543210` (placeholder for now until public promotion via Twitter/Facebook/YouTube)

---

## Key Files

| File | Location | Purpose |
|------|----------|---------|
| `index.html` | Project-03 root | Single source of truth — SPA, all CSS+JS inline |
| `hero_v4.mp4` | Project-03 root | Hero video (900×460, ~6s, xfade dissolve) |
| `CNAME` | Project-03 root | Contains `dropezi.com` — enables custom domain on GitHub Pages |
| `index-redesign.html` | Project-01 root | Master source backup — copy to Project-03 as needed |
| `splash-brand.jpeg` | Project-01 root | Hero video end-frame source image |

---

## Deployment Architecture

```
shrinidhanaraj.github.io/dropezi-project-03/  ← GitHub Pages (project-03 repo)
         ↑
     CNAME: dropezi.com (configured, pending DNS change)

dropezi.in  ← currently live, separate hosting
dropezi.com ← currently redirects to dropezi.in (change to GitHub Pages)
dropezi-website.web.app ← Firebase, DO NOT TOUCH
```

### DNS Steps to go live on dropezi.com (pending, user must do)
1. GitHub → repo Settings → Pages → set Custom Domain = `dropezi.com`
2. At DNS registrar, set A records for `dropezi.com`:
   ```
   185.199.108.153
   185.199.109.153
   185.199.110.153
   185.199.111.153
   ```
3. SSL auto-provisions in ~30 min

---

## Git Workflow (IMPORTANT)

**The Cowork bash sandbox cannot access macOS Keychain**, so `git push` always fails from sandbox.

**Claude can:** `git add` + `git commit` ✅  
**User must run from Terminal:** `git push origin main`

If HEAD.lock or index.lock is stuck:
```bash
rm ~/Claude/Projects/DROPeZi-Project-03/.git/HEAD.lock
rm ~/Claude/Projects/DROPeZi-Project-03/.git/index.lock
```

Bash sandbox path: `/sessions/<session-id>/mnt/DROPeZi-Project-03/`  
File tool path: `/Users/aravinddhanaraj/Claude/Projects/DROPeZi-Project-03/`

---

## Hero Video (hero_v4.mp4)

### Structure
```
[0s–3.8s]  vid_clip  — animated car logo, white strip at bottom
[3.8s–6.0s] brand_clip — splash-brand.jpeg end frame
             xfade dissolve at 3.4s, 0.4s duration
```

### Caption (vid_clip white strip, y=368–460)
- `Endless Reach.  Exceptional Care.` — blue `#2563eb`, fontsize=20, y=382
- `DROPeZi` — blue `#2563eb`, fontsize=36, y=410

### Caption (brand_clip natural grey bg)
- `Endless Reach.  Exceptional Care.  DROPeZi` — navy `#0e1f3d`, fontsize=30, y=422

### Rebuild ffmpeg commands (run in Project-01)
```bash
cd ~/Claude/Projects/DROPeZi-Project-01
FONT=/usr/share/fonts/truetype/liberation2/LiberationSans-Bold.ttf

# Step 1 — vid_clip (blue text)
ffmpeg -y -i DROPeZi-Draft03.mp4 \
  -vf "scale=900:-2,crop=900:460:0:570,\
fade=t=out:st=3.4:d=0.4,\
drawbox=x=0:y=368:w=900:h=92:color=white@1.0:t=fill,\
drawtext=fontfile=$FONT:text='Endless Reach.  Exceptional Care.':fontcolor=#2563eb:fontsize=20:x=(w-text_w)/2:y=382,\
drawtext=fontfile=$FONT:text='DROPeZi':fontcolor=#2563eb:fontsize=36:x=(w-text_w)/2:y=410" \
  -t 3.8 -c:v libx264 -preset fast -crf 22 -an vid_clip.mp4

# Step 2 — brand_clip
ffmpeg -y -loop 1 -i splash-brand.jpeg \
  -vf "scale=740:-2,pad=900:ih:(900-iw)/2:0:color=0xf2f2f2,crop=900:460:0:83,\
fade=t=in:st=0:d=0.4,\
drawtext=fontfile=$FONT:text='Endless Reach.  Exceptional Care.  DROPeZi':fontcolor=#0e1f3d:fontsize=30:x=(w-text_w)/2:y=422" \
  -t 2.2 -c:v libx264 -preset fast -crf 22 -pix_fmt yuv420p -an brand_clip.mp4

# Step 3 — xfade merge
ffmpeg -y -i vid_clip.mp4 -i brand_clip.mp4 \
  -filter_complex "[0:v]fps=30[v0];[1:v]fps=30[v1];[v0][v1]xfade=transition=dissolve:duration=0.4:offset=3.4[v]" \
  -map "[v]" -c:v libx264 -preset fast -crf 22 -movflags +faststart -an hero_v4.mp4
```

---

## Quote Calculator — Full Logic

### Fare Constants
```javascript
const MIN_BILLED_KM     = 130;   // minimum billable distance per one-way leg
const DRIVER_CHARGE_DAY = 300;   // ₹300 per day
const DRIVER_MAX_KM_DAY = 400;   // driver max 400 KM/day
const TOLL_PER_KM       = 0.8;   // ₹0.8/km, rounded to nearest ₹10
```

### Vehicle Rates
| Vehicle | Rate | Contact |
|---------|------|---------|
| Sedan   | ₹14/km | No |
| SUV     | ₹19/km | No |
| Premium | — | Call us |

### Calculation Model

**Step 1 — Effective distance**
- `billedOneWayKm = max(distKm, 130)`
- `totalBilledKm = isRound ? billedOneWayKm × 2 : billedOneWayKm`
- For round trip, **distance shown to customer = distKm × 2**

**Step 2 — Ride fare**
- `rideFare = totalBilledKm × ratePerKm`
- Round trip discount: `rtDiscount = isRound ? rideFare × 10% : 0`

**Step 3 — Night surcharge**
- Applied if departure hour is 22–23 or 0–5
- `nightSurcharge = billedOneWayKm × ratePerKm × 10%` (one-way leg only)

**Step 4 — Driver charge**
- `driverDays = Math.ceil(totalActualKm / 400)`  ← uses actual km, not billed
- `totalActualKm = isRound ? distKm × 2 : distKm`
- `driverCharge = driverDays × ₹300`

**Step 5 — Tolls**
- `tollEst = Math.round(totalActualKm × 0.8 / 10) × 10`

**Step 6 — State permit**
- TN→TN: ₹0
- Outside TN: `estimatedStates = min(ceil(distKm / 400), 5)` × ₹500
- State detection via Nominatim `addressdetails=1` checking `address.state`

**Grand Total = (rideFare − rtDiscount + nightSurcharge) + driverCharge + tollEst + permitCharge**

### Fare Card Display Order
1. Distance (X KM × 2 for round trip)
2. Vehicle rate
3. Ride Fare (N KM)
4. 🔖 Round Trip Discount −10% [green, hidden for one-way]
5. 🌙 Night Surcharge [hidden if daytime]
6. 👤 Driver Charge (N days × ₹300)
7. 🛣️ Tolls (est.)
8. 📋 State Permit (~N states × ₹500) [hidden within TN]
9. 💰 Grand Total (gold)
10. Notes: * Tolls are estimates / * Parking paid by customer

---

## Multi-Language Support
Languages: Tamil (default), English, Kannada, Malayalam, Hindi  
Toggle in navbar — switches instantly, no reload  
i18n object in JS: `i18n.ta`, `i18n.en`, `i18n.kn`, `i18n.ml`, `i18n.hi`

---

## Data Sovereignty / DPDP Compliance (Current State)

**Current site is static HTML — no backend, no database.**
- No personal data stored server-side ✅
- Third-party API calls that send city names:
  - Nominatim (OpenStreetMap) — Germany servers ⚠️
  - OSRM routing — international servers ⚠️
- Phone numbers entered by user go directly to WhatsApp (Meta) — not stored
- GitHub Pages CDN: global (Fastly) — not India-specific

**For v1.0 (static site):** DPDP risk is low — no personal data collected or stored.

**When login/registration is added:** MUST use AWS Mumbai (`ap-south-1`) or GCP Mumbai (`asia-south1`). Consider replacing Nominatim/OSRM with Indian alternatives (MapMyIndia/OlaMap API).

---

## Known Issues / Enhancement Backlog

### High Priority (v1.1)
- [ ] Date picker: past dates still selectable on some browsers (JS `min` attr set but browser inconsistency)
- [ ] Premium vehicle: call/WA buttons use placeholder number — replace when real number confirmed
- [ ] Toll estimate is rough (₹0.8/km) — consider MapMyIndia Toll API for accuracy
- [ ] Multi-state permit (₹500/state) is estimated — actual rates vary by state/vehicle

### Medium Priority
- [ ] Add more popular routes as pre-filled quick-book cards
- [ ] Tamil Nadu city auto-suggest on From/To fields
- [ ] WhatsApp Business API integration (Gupshup/AISensy) for automated response
- [ ] OTP-based phone verification before sending quote

### Future (v2.0)
- [ ] User registration + login (Firebase Auth, data in Mumbai region)
- [ ] Booking history / loyalty points
- [ ] Real-time driver tracking
- [ ] Payment gateway (Razorpay)
- [ ] Merge dropezi.in + dropezi.com into single codebase

---

## CSS Variables (Key Colours)
```css
--navy:     #0e1f3d
--gold:     #F0A500
--gold-dark:#d4920a
--wa:       #25D366   (WhatsApp green)
--wa-dark:  #1ea352
--white:    #ffffff
--off-white:#f8f6f0
```

---

## Commit Message Convention
```
feat: new feature
fix: bug fix
ux: UX/styling change
refactor: logic restructure
```
