# Migration Runbook: Netlify → Cloudflare Pages

**Site:** firststateautomation.ai  
**Repo:** firststateautomation-ops/fsa-website  
**Date prepared:** April 2026  
**Status:** Research only — no changes made

---

## 1. What to Capture from Netlify Before Disconnecting

### Environment Variables
Log in to Netlify → select the site → **Site configuration → Environment variables**.

For this site there is no build process, so it is unlikely any env vars are set. Confirm by looking at that screen. If any exist, copy the key names and values to a secure location (password manager or 1Password) before disconnecting. **Do not commit them to GitHub.**

### Redirects
There is no `netlify.toml` and no `_redirects` file in the fsa-website repository. Go to **Site configuration → Redirects** in the Netlify UI and confirm the list is empty. If any rules appear there, write them down exactly — you will need to recreate them as a `_redirects` file in the repo or as Cloudflare Pages redirect rules.

Currently no redirects are expected based on the repo contents.

### Build Settings
This site has no build step. The current effective settings are:
- Build command: *(none)*
- Publish directory: `/` (repo root)
- Branch to deploy: `master`

No action needed — these are trivial and have no equivalent to carry forward.

### Forms
Check **Site configuration → Forms** in Netlify. If any Netlify Forms are active, note them. The site currently uses a Tally form (`https://tally.so/r/oboj0X`) which posts to an external service and does not depend on Netlify Forms. Confirm nothing is wired to Netlify's form handling before disconnecting.

### Deploy Notifications and Webhooks
Check **Site configuration → Build & deploy → Deploy notifications**. If any outgoing webhooks are configured (e.g. Slack alerts, Make.com triggers on deploy), note them so you can recreate equivalent notifications in Cloudflare Pages if needed.

### Netlify Site ID (for reference)
```
f402eae6-608b-478c-9602-b081143e6491
```
Keep this on hand in case you need to reference old deploy logs during the transition.

### Subdomain on Netlify
Note the `*.netlify.app` subdomain assigned to this site (visible on the Domains page). You may want to keep the Netlify site in a disconnected-but-not-deleted state for a few days after cutover so you can roll back quickly if needed.

---

## 2. Connecting fsa-website to Cloudflare Pages

Do this before touching any DNS records.

1. Log in to the Cloudflare dashboard at dash.cloudflare.com.
2. In the left sidebar, click **Workers & Pages**.
3. Click **Create application**, then select the **Pages** tab.
4. Click **Connect to Git**.
5. If prompted, authorize the Cloudflare Pages GitHub app for the `firststateautomation-ops` organization. Grant access to the `fsa-website` repository (you can limit access to just that one repo).
6. Select the `firststateautomation-ops / fsa-website` repository and click **Begin setup**.
7. Fill in the build configuration:
   - **Project name:** `fsa-website` (this sets your preview URL to `fsa-website.pages.dev`)
   - **Production branch:** `master`
   - **Framework preset:** None
   - **Build command:** *(leave blank)*
   - **Build output directory:** *(leave blank — Cloudflare Pages will serve from the repo root)*
8. Click **Save and Deploy**. Cloudflare will pull the repo and deploy it. This takes about 60 seconds.
9. When the deploy finishes, click the `fsa-website.pages.dev` preview URL and confirm the site loads correctly at that URL before proceeding to DNS.
10. Add the custom domain:
    - In the Pages project, go to **Custom domains**.
    - Click **Set up a custom domain**.
    - Enter `firststateautomation.ai` and click **Continue**. Cloudflare will detect that this domain is already in your account and offer to add the DNS record automatically.
    - Repeat for `www.firststateautomation.ai`.
    - Cloudflare Pages will issue a TLS certificate automatically for both. Wait for the certificate to show **Active** before cutting over DNS.

---

## 3. DNS Records to Change in Cloudflare

Cloudflare already manages DNS for `firststateautomation.ai`. Because of that, the "DNS change" is just updating existing records — no registrar action needed.

### Current state (pointing to Netlify)
The DNS records currently point the apex domain and www to Netlify's infrastructure. They look like one of these patterns:

| Type | Name | Value |
|------|------|-------|
| A | `@` | Netlify load balancer IP (e.g. `75.2.60.5` or `99.83.231.61`) |
| CNAME | `www` | `[site-name].netlify.app` |

or

| Type | Name | Value |
|------|------|-------|
| CNAME | `@` | `[site-name].netlify.app` (CNAME-flattened by Cloudflare) |
| CNAME | `www` | `[site-name].netlify.app` |

Open the Cloudflare DNS dashboard for `firststateautomation.ai` and note exactly what is there before making any change.

### What to change

When you add the custom domain inside the Cloudflare Pages UI (Step 10 above), Cloudflare will offer to update the DNS automatically. **Let it do this.** It will:

- Set the `@` (apex) record to a CNAME pointing to `fsa-website.pages.dev` (Cloudflare flattens this at the edge).
- Set the `www` CNAME to `fsa-website.pages.dev`.

If you prefer to do it manually instead:

| Action | Type | Name | New value | Proxy status |
|--------|------|------|-----------|---------------|
| Update (or delete old + add new) | CNAME | `@` | `fsa-website.pages.dev` | Proxied (orange cloud) |
| Update (or delete old + add new) | CNAME | `www` | `fsa-website.pages.dev` | Proxied (orange cloud) |

**Leave all other DNS records untouched** — especially any MX records for email and any TXT records (SPF, DKIM, domain verification tokens).

### Propagation
Because Cloudflare manages the DNS, changes are effective almost instantly (seconds to a few minutes). There is no 24-48 hour wait as with external registrars.

---

## 4. Risk to the Existing Cloudflare Worker

### Worker details
- URL: `https://fsa-claude-proxy.edwardhollyday.workers.dev`
- Purpose: Proxies requests from the website frontend to the Anthropic API, keeping the API key server-side.
- Deployment: Deployed on the `edwardhollyday.workers.dev` subdomain, which belongs to a separate Cloudflare zone/account from `firststateautomation.ai`.

### Risk assessment
**The Worker itself is not at risk from this migration.** Here is why:

1. The Worker URL (`fsa-claude-proxy.edwardhollyday.workers.dev`) is an absolute URL hardcoded in the website's HTML. Moving the site from Netlify to Cloudflare Pages does not change what URL the browser calls — it will still call the same Worker endpoint.
2. The Worker runs on the `edwardhollyday.workers.dev` domain, which is completely independent of the `firststateautomation.ai` domain. Changes to DNS or hosting for `firststateautomation.ai` have no effect on the Worker's routing.
3. There are no Cloudflare Pages route rules that would intercept requests to `edwardhollyday.workers.dev`.

### One risk to verify: CORS headers
If the Worker includes a CORS policy like `Access-Control-Allow-Origin: https://firststateautomation.ai`, it will continue to work because the production domain does not change. However, if the Worker was also configured to allow Netlify preview URLs (e.g. `*.netlify.app`), those will stop working after the Netlify site is disconnected — but that is expected and acceptable. Cloudflare Pages preview deployments use `*.pages.dev` URLs; if you want the chatbot to work on preview branches, you would need to add `https://fsa-website.pages.dev` (or a wildcard pattern) to the Worker's allowed origins.

**Action:** Open the Worker in the Cloudflare dashboard under **Workers & Pages → fsa-claude-proxy**, go to the source code, and check for any `Access-Control-Allow-Origin` logic. If the value is hardcoded to `https://firststateautomation.ai` or set to `*`, no change is needed. Only update it if you want preview-URL support.

### No route conflicts
Cloudflare Pages and Cloudflare Workers can coexist on the same account. If both are in the same Cloudflare account, there is no conflict because they serve different hostnames. Confirm there are no existing Worker routes set to match `firststateautomation.ai/*` in the `firststateautomation.ai` zone (check **Workers & Pages → Overview → Routes** for the zone). If any such routes exist, they could intercept traffic — but this is unlikely given the current architecture.

---

## 5. Post-Migration Checklist

Run through every item below before considering the migration complete.

### DNS and TLS
- [ ] `https://firststateautomation.ai` loads the site (no certificate warning)
- [ ] `https://www.firststateautomation.ai` loads the site (no certificate warning)
- [ ] TLS certificate in Cloudflare Pages shows **Active** for both domains
- [ ] Old Netlify A/CNAME records are no longer present in Cloudflare DNS

### Page content
- [ ] `https://firststateautomation.ai/` — homepage loads with correct branding (Delaware Blue / Gold)
- [ ] `https://firststateautomation.ai/blog-post-1.html` — loads correctly
- [ ] `https://firststateautomation.ai/blog-post-2.html` — loads correctly
- [ ] `https://firststateautomation.ai/blog-post-3.html` — loads correctly
- [ ] `https://firststateautomation.ai/roi-calculator.html` — loads correctly
- [ ] `https://firststateautomation.ai/sitemap.xml` — returns XML (not a 404)
- [ ] `https://firststateautomation.ai/robots.txt` — returns the correct content

### Functionality
- [ ] Chatbot widget opens on the homepage
- [ ] Chatbot sends a test message and receives a real response (confirms the Worker is reachable from the new hosting)
- [ ] Contact/Tally form submits successfully and a notification arrives as expected
- [ ] Make.com webhook (`hook.us1.make.com/...`) receives the form submission

### Cloudflare Pages deployment pipeline
- [ ] Make a trivial change to `master` in fsa-website (e.g. add a space and revert it) and confirm Cloudflare Pages automatically picks up the push and deploys within 2 minutes
- [ ] Confirm deployment shows as **Success** in Workers & Pages → fsa-website → Deployments

### SEO / search
- [ ] Google Search Console — confirm the sitemap URL (`https://firststateautomation.ai/sitemap.xml`) is still reachable; resubmit if needed
- [ ] Run a quick crawl with a tool like `curl -I https://firststateautomation.ai` to confirm HTTP 200 response and no unexpected redirects

### Cleanup (after 48 hours with no issues)
- [ ] Disconnect the custom domain from Netlify (Netlify site dashboard → Domain management → remove `firststateautomation.ai`)
- [ ] Optionally delete the Netlify site entirely, or leave it archived
- [ ] Update `CLAUDE.md` and `AGENTS.md` in fsa-website to replace all references to Netlify with Cloudflare Pages
- [ ] Update `STATUS.md` in fsa-brain to reflect the new hosting platform
