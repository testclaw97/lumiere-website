# Lumiere Media & Marketing — Website

**What:** Landing page for Lumiere, an AI-powered social media agency for dental & cosmetic practices.
**Stack:** Static HTML, Tailwind CSS (CDN), vanilla JS, self-hosted Inter font (DSGVO compliant).
**Location:** ~/products/lumiere/site/
**Serve locally:** PM2 static server on port 3100 (no `-s` flag — multi-page, not SPA)
**Live URL:** https://testclaw97.github.io/lumiere-website/
**GitHub:** https://github.com/testclaw97/lumiere-website

## Site Structure
| File | Purpose |
|---|---|
| index.html | Main landing page (hero, process, pricing, results, about, FAQ, contact) |
| impressum.html | Legal Impressum (§5 TMG) — has placeholder brackets for address/phone/USt-IdNr |
| datenschutz.html | DSGVO privacy policy (hosting, Formspree, self-hosted fonts, Tailwind CDN) |
| 404.html | Custom branded 404 page |
| assets/style.css | Custom CSS (variables, animations, FAQ accordion, floating CTA, @font-face) |
| assets/logo.svg | SVG logo (gold sparkle + white LUMIERE text) |
| assets/og-image.svg | Open Graph social sharing image (1200x630) |
| assets/fonts/ | Self-hosted Inter font files (woff2) — no Google CDN |
| vercel.json | Vercel config (if migrating from GitHub Pages) |
| start.sh | PM2 startup script (local dev) |

## Strategy Docs (in docs/)
| Document | What it covers |
|---|---|
| business-plan.md | 10-section plan: market analysis, financials, AI roadmap, milestones |
| social-media-plan.md | 90-day posting plan for Lumiere's own channels (IG, LinkedIn, TikTok) |
| legitimacy-playbook.md | Step-by-step guide to building credibility as a new agency |

## Contact Form
- Currently uses mailto: fallback (opens email client with pre-filled fields)
- Formspree-ready: replace `placeholder` in form action URL to enable AJAX submission
- Email: tejaskapoortennis@gmail.com

## TJ TODO (cannot be automated)
- [ ] Fill in Impressum: street address, postal code/city, phone, USt-IdNr
- [ ] Get professional email (info@lumiere-media.de) — gmail looks unprofessional
- [ ] Create Formspree account → replace `placeholder` in form action URL
- [ ] Create Instagram Business account for Lumiere
- [ ] Create LinkedIn company page
- [ ] Register domain (lumiere-media.de or similar)

## Folder Routing
| Task | Folder | Read first |
|---|---|---|
| Website content/styling | site/ | This file |
| Strategy/planning docs | docs/ | Relevant doc |
| Specs & plans | docs/superpowers/ | Relevant spec |

## How to run locally
```bash
cd ~/products/lumiere/site && npx serve . -l 3100
# Or via PM2:
pm2 start ~/products/lumiere/site/start.sh --name lumiere-website
```

## Deploy
Push to main → GitHub Pages auto-deploys.
```bash
cd ~/products/lumiere/site && git add -A && git commit -m "description" && git push origin main
```
