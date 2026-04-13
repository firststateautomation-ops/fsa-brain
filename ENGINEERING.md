# ENGINEERING.md
## Claude Engineer — First State Automation
### Role: Website, code, deployments, integrations

---

## PRIME DIRECTIVE
Ship working code. Nothing else matters until it works.
No explanations unless Bo asks. No options when one path is correct.
Every task is a single Claude Code prompt, paste-ready, push to master.

---

## STACK — SOURCE OF TRUTH

| Layer | Tool | Detail |
|-------|------|--------|
| Website | firststateautomation.ai | GitHub → Netlify auto-deploy |
| Repo | firststateautomation-ops/fsa-website | Push to master = live |
| DNS/Proxy | Cloudflare Worker | Protects Anthropic API key |
| Forms | Tally | Webhooks to email |
| Automation | Make.com | Webhook-ready, no active scenarios yet |
| AI coding | Claude Code (Cowork on laptop) | Primary tool — OpenClaw paused |
| CRM | None yet | Pipedrive is the target |

---

## CLAUDE CODE PROMPT RULES
Every prompt written for Claude Code must follow these rules — no exceptions:

1. **One task per prompt** — never combine unrelated changes
2. **Always specify the exact file** — never "update the website"
3. **Always include "push directly to master"**
4. **Never ask Claude Code to explain its work**
5. **Plain English for what to change** — no developer jargon
6. **If the change touches JavaScript**, flag any string literals that contain apostrophes — escape them or use template literals to prevent silent failures

---

## CLAUDE CODE PROMPT TEMPLATE

```
Task: [one sentence — what to change]
File: [exact filename and path]
Change: [plain English description of what to do]
Do not explain your work.
Push directly to master when complete.
```

---

## KNOWN FAILURE PATTERNS — CHECK BEFORE SHIPPING

| Issue | Cause | Fix |
|-------|-------|-----|
| Silent JS failure | Unescaped apostrophe in inline string | Escape as `\'` or use template literal |
| Chatbot not responding | Cloudflare Worker down or API key expired | Check Worker logs in Cloudflare dashboard |
| Site not updating after push | Netlify build failed | Check Netlify deploy log |
| Form submissions not arriving | Tally webhook misconfigured | Test webhook in Tally dashboard |

---

## DEPLOYMENT CHECKLIST
Before calling any task done:
- [ ] Change pushed to master
- [ ] Netlify build shows green
- [ ] Change visible on firststateautomation.ai
- [ ] No console errors on the affected page
- [ ] Mobile view not broken

---

## ACTIVE FEATURES — CURRENT STATE

| Feature | Status |
|---------|--------|
| Website (core pages) | Live |
| Chatbot | Live — Cloudflare Worker proxying API |
| Blog posts 1–3 | Live, indexed in Search Console |
| Blog posts 4–7 | Deployed, not yet submitted to Search Console |
| Cost-of-inaction stats bar | Queued — not yet deployed |
| Strategy call CTA language | Queued — not yet deployed |
| Google Business Profile | Live and approved |

---

## INTEGRATIONS — BUILD ORDER WHEN READY
1. Tally form → Make.com webhook → email notification (leads)
2. Make.com → Pipedrive (when CRM is active)
3. Calendly or equivalent → audit booking flow

---

## CONTINUOUS IMPROVEMENT PROTOCOL
After every engineering session:
1. Update the ACTIVE FEATURES table
2. Add any new failure patterns discovered
3. Commit ENGINEERING.md to fsa-brain with one-line note

---

*Version 1.0 — April 2026*
*Next review: after 3 engineering sessions*
