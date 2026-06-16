# DropEzi — Production Deployment Guide
**Stack:** Firebase Hosting (Google Cloud) · Squarespace DNS · GitHub Actions CI/CD  
**Domains:** dropezi.in · dropezi.com · dropezi.app  
**Repo:** github.com/shrinidharaj/DROPeZi-Project-01

---

## Prerequisites

| Tool | Install |
|---|---|
| Node.js ≥ 18 | https://nodejs.org |
| Firebase CLI | `npm install -g firebase-tools` |
| Git | pre-installed on macOS |

---

## PHASE 1 — Firebase Project Setup

### 1.1 Create Firebase Project

1. Go to https://console.firebase.google.com
2. Click **Add project** → name it `dropezi-prod`
3. **Disable** Google Analytics (not needed; avoids unnecessary data collection)
4. Click **Continue** until project is created

> **DPDP compliance note:** Firebase Hosting serves static files via Google's CDN. Since this is a pure static site with no form submissions or server-side data storage, no personal data is collected or stored by your infrastructure — DPDP Act compliance is met for this setup. If you add a backend later, use Firestore region `asia-south1` (Mumbai).

---

### 1.2 Initialize Firebase in Your Local Repo

```bash
# Clone the repo locally if not already done
git clone https://github.com/shrinidharaj/DROPeZi-Project-01.git
cd DROPeZi-Project-01

# Login to Firebase
firebase login

# Initialize Hosting
firebase init hosting
```

Answer the prompts:
- **Which Firebase project?** → Select `dropezi-prod`
- **Public directory?** → `.` (dot — the repo root, since index.html is at root)
- **Single-page app (rewrite all to index.html)?** → `Yes`
- **Set up GitHub Actions?** → `Yes` (follow prompts — it asks for your GitHub repo)
- **Overwrite index.html?** → **No**

---

### 1.3 Replace Generated `firebase.json`

After init, replace `firebase.json` with this optimized config:

```json
{
  "hosting": {
    "public": ".",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**",
      "DEPLOY.md",
      "*.md"
    ],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ],
    "headers": [
      {
        "source": "**/*.mp4",
        "headers": [
          { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
        ]
      },
      {
        "source": "**/*.@(webp|jpg|jpeg|png)",
        "headers": [
          { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
        ]
      },
      {
        "source": "**/*.@(js|css)",
        "headers": [
          { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
        ]
      },
      {
        "source": "/",
        "headers": [
          { "key": "Cache-Control", "value": "public, max-age=0, must-revalidate" },
          { "key": "X-Frame-Options", "value": "SAMEORIGIN" },
          { "key": "X-Content-Type-Options", "value": "nosniff" }
        ]
      }
    ]
  }
}
```

> **Why this config:** Media files (MP4, WebP, JPG) get a 1-year immutable cache — huge speed win on repeat visits on Indian mobile networks. `index.html` always revalidates so users always get your latest version.

---

### 1.4 Test Deploy to Firebase Default Domain

```bash
firebase deploy --only hosting
```

This deploys to `https://dropezi-prod.web.app`. Visit that URL and confirm the site looks correct before touching DNS on any of your three domains.

---

## PHASE 2 — Add All Three Custom Domains

In **Firebase Console → Hosting → Add custom domain**, add all 6 entries (root + www for each domain):

```
dropezi.in
www.dropezi.in
dropezi.com
www.dropezi.com
dropezi.app
www.dropezi.app
```

For each domain, Firebase gives you two things in sequence:
1. A **TXT record** for domain ownership verification
2. After verification: two **A records** (Firebase's IP addresses for your project)

**Copy these values exactly from Firebase Console** — do not use hardcoded IPs from online guides, as they vary per project.

Firebase provisions SSL certificates automatically via Let's Encrypt once DNS propagates. No manual certificate steps required.

---

## PHASE 3 — DNS Configuration in Squarespace

Go to: **Squarespace Domains → [domain] → DNS Settings → Custom Records**

### 3.1 dropezi.in

| Type | Host | Value | TTL |
|---|---|---|---|
| TXT | @ | `firebase-domain-verify=XXXXXXXX` ← from Firebase | 3600 |
| A | @ | Firebase IP 1 ← from Firebase | 3600 |
| A | @ | Firebase IP 2 ← from Firebase | 3600 |
| CNAME | www | `dropezi-prod.web.app` | 3600 |

### 3.2 dropezi.com

Same structure as above — use the values Firebase Console shows for `dropezi.com`.

### 3.3 dropezi.app

Same structure. Note: `.app` domains are HSTS-preloaded by browsers, meaning HTTPS is enforced automatically — Firebase Hosting's SSL covers this with no extra steps.

> **Squarespace tip:** If Squarespace shows an existing A record pointing to their own servers, **delete it first** before adding Firebase's A records. Duplicate A records cause SSL provisioning failures.

---

### 3.4 Check DNS Propagation

Run these in Terminal after adding records (usually 15–60 min with Squarespace):

```bash
# Check A records resolve to Firebase's IPs
dig dropezi.in A +short
dig dropezi.com A +short
dig dropezi.app A +short

# Check TXT verification record is present
dig dropezi.in TXT +short
```

Firebase Console shows a green checkmark per domain once SSL is fully provisioned.

---

## PHASE 4 — GitHub Actions CI/CD

`firebase init` creates `.github/workflows/firebase-hosting-merge.yml`. Verify it looks like this:

```yaml
name: Deploy to Firebase Hosting on merge
on:
  push:
    branches:
      - main

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: '${{ secrets.GITHUB_TOKEN }}'
          firebaseServiceAccount: '${{ secrets.FIREBASE_SERVICE_ACCOUNT_DROPEZI_PROD }}'
          channelId: live
          projectId: dropezi-prod
```

The `FIREBASE_SERVICE_ACCOUNT_DROPEZI_PROD` secret is added automatically to your GitHub repo when you run `firebase init hosting` with GitHub Actions selected.

Verify it exists at: **GitHub repo → Settings → Secrets and variables → Actions**

From this point: every push to `main` auto-deploys in ~30 seconds.

---

## PHASE 5 — Pre-Launch Checklist

### SSL & Domains
- [ ] `https://dropezi.in` loads with padlock
- [ ] `https://dropezi.com` loads with padlock
- [ ] `https://dropezi.app` loads with padlock
- [ ] `https://www.dropezi.in` loads (no certificate error)
- [ ] `https://www.dropezi.com` loads
- [ ] `https://www.dropezi.app` loads
- [ ] `http://dropezi.in` auto-redirects to `https://` (Firebase does this automatically)

### Mobile Performance (test on real device, Indian 4G SIM)
- [ ] Page visible in < 2 seconds on 4G (use Chrome DevTools → Network → "Fast 4G" throttle to simulate)
- [ ] `DROPeZi-Draft03.mp4` does not block page render (confirm `autoplay muted playsinline` attributes set)
- [ ] Logo WebP displays correctly; JPG fallback shown on older browsers
- [ ] Page layout correct on 360px wide screen (most common Indian Android width)

### CTAs
- [ ] Click-to-Call button (`tel:+91XXXXXXXXXX`) opens native dialer on Android
- [ ] Click-to-Call button opens native dialer on iOS
- [ ] WhatsApp button (`https://wa.me/91XXXXXXXXXX?text=...`) opens WhatsApp with pre-filled message in Tamil (default)
- [ ] WhatsApp message changes correctly when English is selected
- [ ] WhatsApp message changes correctly for Kannada, Malayalam, Hindi

### Localization
- [ ] Default page loads in Tamil
- [ ] Language toggle switches instantly (no page reload)
- [ ] All 5 languages render without broken characters

### Cross-Browser
- [ ] Chrome on Android ✓
- [ ] Safari on iOS ✓
- [ ] Chrome on Desktop ✓

---

## Post-Launch (Optional)

Disable GitHub Pages to avoid traffic confusion:  
**GitHub repo → Settings → Pages → Source → None**

---

## Quick Reference

| What | Command / URL |
|---|---|
| Manual deploy | `firebase deploy --only hosting` |
| View deploy history | Firebase Console → Hosting → Release history |
| Preview URL | `https://dropezi-prod.web.app` |
| Check DNS | `dig dropezi.in A +short` |
| Firebase CLI docs | https://firebase.google.com/docs/hosting/custom-domain |
