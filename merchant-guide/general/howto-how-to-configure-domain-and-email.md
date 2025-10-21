---
title: "How to Configure Domain and Email"
description: "Set up custom domain and professional email for your store"
type: how-to
audience:
  - "Magento merchants"
  - "Advanced users"
difficulty: Intermediate
estimated_time: "60–120 minutes (plus DNS propagation)"
prerequisites:
  - "Access to your domain registrar/DNS provider"
  - "Access to your hosting control panel or server (or your Adobe Commerce on cloud project)"
  - "Magento Admin credentials"
  - "Optional: SSH access for server/CLI tasks (e.g., Certbot, Magento CLI)"
tags:
  - how-to
last_updated: "2025-09-17T13:14:28.813Z"
---

# How to Configure Domain and Email

## Overview

Configure a custom domain and professional email to boost brand trust, SEO, and email deliverability. A canonical HTTPS domain prevents duplicate content in search, protecting rankings. Authenticated email (SPF/DKIM/DMARC) improves inbox placement, reducing support tickets from missed order emails. Expect improved CTR from organic traffic (single canonical URLs) and fewer failed order confirmations, which can decrease customer service load and abandonment.

Applies to: Magento Open Source and Adobe Commerce 2.4.x. For Adobe Commerce on cloud infrastructure, see Cloud callouts in each step.

Track results to quantify ROI:
- Order email delivery rate (% delivered vs. sent)
- Open rate for order confirmations and shipping updates
- Support tickets related to missing emails
- Organic sessions and CTR to canonical URLs
- Bounce rate and time-to-first-byte on canonical vs. non-canonical URLs

How to measure impact effectively:
- Before changes, capture a 7–14 day baseline: export delivery/open rates from your ESP dashboard, count support tickets tagged “missing email”, and record organic sessions/CTR from analytics (per canonical landing pages).
- After changes, compare the same KPIs over the next 7–14 days. Use ESP message logs to validate Delivered (250 OK) vs bounces/blocks.

Targets and guidance:
- Delivery rate: ≥ 98%
- Order confirmation open rate: 60–80%
- Support tickets about missing emails: decrease by ≥ 50% after DMARC enforcement
- Redirects: single-hop 301 to the canonical HTTPS host for all entry points

Quick ROI example:
- If you send 4,000 order emails/month and 3% fail today (120 emails), with a 15% support touch rate at $6/ticket and 5% churn on those customers at $120 AOV, improving delivery to 98% saves ≈ $108/month in support costs and preserves ≈ $720/month in at-risk revenue.

Note for Adobe Commerce on cloud infrastructure:
- If you use Adobe Commerce on cloud infrastructure, configure domain, TLS, and redirects with Fastly (do not install certificates on application nodes). Outbound port 25 is blocked; use an email service provider (ESP) via SMTP or API.
- See cloud docs: Fastly and custom domains: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/cdn/fastly/custom-domains.html and Fastly TLS/SSL: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/cdn/fastly/fastly-ssl-tls.html. Cloud overview: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/overview.html

Cloud Quick Track (summary):
- Use Fastly for domains, TLS, and 301 redirects; point DNS to Fastly per cloud docs
- Set Magento Base URLs to the Fastly HTTPS domains
- Use an ESP via SMTP/API (port 25 blocked)
- See the detailed “If you are on Adobe Commerce on cloud infrastructure (quick track)” section under Prerequisites for step mapping

Tip: Allow 45–90 minutes to complete, plus DNS propagation time.

## Prerequisites

Before you begin, make sure you have:

- Access to your domain registrar/DNS provider
- Access to your hosting control panel or server (or your Adobe Commerce on cloud project)
- Magento Admin credentials
- Optional: SSH access for server/CLI tasks (e.g., Certbot, Magento CLI)

Who should use this guide: Merchant administrators and technical managers can complete most steps. A developer or hosting provider may assist with server/CDN configuration.

### If you are on Adobe Commerce on cloud infrastructure (quick track)

- Configure domains, TLS, and redirects in Fastly (edge). Do not terminate TLS on application nodes.
- Point DNS to Fastly per cloud docs; use Fastly for 301 canonical/HTTPS redirects.
- Set Magento Base URLs to the Fastly-configured HTTPS domains.
- Use an ESP via SMTP or API (port 25 is blocked). Do not rely on host SMTP.
- Cloud step mapping (reference as you follow this guide):
  - Step 2 (DNS) → Point domains to Fastly per "Custom domains" docs.
  - Step 3 (TLS) → Upload/manage certificates in Fastly.
  - Step 4 (Redirects) → Implement canonical/HTTPS redirects via Fastly VCL or settings.
  - Step 5 (Base URLs) → Set Base URLs to Fastly-hosted HTTPS domains; keep Magento auto-redirect off.
  - Steps 6–9 (Email) → Use an ESP (SMTP/API). Do not use host SMTP.
- Reference:
  - Custom domains: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/cdn/fastly/custom-domains.html
  - TLS/SSL: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/cdn/fastly/fastly-ssl-tls.html

## What You'll Accomplish

By following this guide, you will:

- Configure your store’s custom domain and professional email correctly
- Improve your store's performance and customer experience

## Step-by-Step Instructions

Quick checklist (experienced users):
- DNS (Step 2): Create A/AAAA (if supported), CNAME (www), MX (mail provider), TXT for SPF, DKIM, and DMARC
- TLS/SSL (Step 3): Issue and install certificates (or configure at CDN)
- Redirects (Step 4): Force HTTPS and canonical host at the edge (CDN or web server)
- Magento (Step 5): Set Base URLs (secure), enable web server rewrites
- Email (Steps 6–9): Choose an Email Service Provider (ESP) and install a single SMTP/API module; set store email identities
- Authentication (Step 7): Configure SPF, DKIM, DMARC; set return-path/bounce domain
- Sales Emails (Step 9): Enable and map senders; enable asynchronous sending; ensure cron is running
- Validate (Step 10): DNS, single-hop redirects, TLS grade, email authentication headers, ESP delivery logs

This guide is divided into two themes to streamline your setup:
- Domain & HTTPS (Steps 1–5)
- Email & Deliverability (Steps 6–10)

### Step 1: Plan your canonical domain and email identities

Choose your canonical host:
- Pick www.yourdomain.com if you use a CDN or want maximum flexibility and cookie isolation (recommended).
- Pick yourdomain.com if you prefer simplicity and your DNS/CDN supports apex records (ALIAS/ANAME).

List the professional email addresses you will use (e.g., support@yourdomain.com, orders@yourdomain.com). If you plan to use a transactional email provider (recommended), create your account now.

Decide first (to reduce rework later):
- Canonical host: www vs apex (see above)
- Delivery architecture: direct-to-origin vs CDN in front of your origin (affects DNS and TLS steps)

### Step 2: Create DNS records for your website

Note: In most DNS UIs, Host/Name "@" represents the apex (yourdomain.com). If your provider does not support "@", enter the full domain name (yourdomain.com). Most DNS UIs do not require a trailing dot. If your UI shows a trailing dot in examples (e.g., ASPMX.L.GOOGLE.COM.), you can usually enter it without the dot. Always enter the full hostname for record targets (e.g., yourdomain.com), not "@".

In your DNS provider, create the following records:
- A (apex): Host/Name = @ (or yourdomain.com), Value = <your IPv4>, TTL = 300
- AAAA (IPv6, optional): Host/Name = @ (or yourdomain.com), Value = <your IPv6>
- CNAME (www): Host/Name = www, Value = yourdomain.com (most DNS UIs accept without a trailing dot; if required, use yourdomain.com.). Do not create a CNAME at the apex (RFC-compliant providers disallow it).
  - If your DNS provider requires it, use the apex hostname (yourdomain.com) instead of @.

CDN variant (non-cloud):
- If using a CDN in front of your origin, point DNS to the CDN per provider docs (e.g., ALIAS/ANAME for apex to the CDN hostname, and CNAME for www to the CDN). Do not point apex directly to the origin when your CDN requires termination at the edge.
- If your CDN has a proxy toggle (e.g., Cloudflare’s orange cloud), enable proxying on the records that serve web traffic.
- Validate CDN health checks/backends are green before switching traffic. Only cut over DNS after TLS is ready at the CDN.

Examples (replace with your actual values):
- A record (apex): Host/Name = @ (or yourdomain.com), Value = 203.0.113.10, TTL = 300
- CNAME (www): Host/Name = www, Value = yourdomain.com

Note on IPv6: Only add an AAAA record if your origin and CDN fully support IPv6. Test with: curl -6 -I https://yourdomain.com

Cutover planning (reduce downtime and risk):
- Lower TTL to 300 seconds 24–48 hours before changes.
- Pre-provision TLS (on origin or CDN) and verify with a hosts-file test if possible.
- Schedule DNS changes in a low-traffic window. Prepare a quick rollback (restore prior DNS, disable new redirects) if a critical issue appears.

Save changes and wait for DNS propagation (5–30 minutes, sometimes longer).

If you need to receive email at your domain (e.g., support@yourdomain.com), you will add MX records for your mailbox provider in Step 7.

### Step 3: Provision and install an SSL/TLS certificate

Choose one approach:
- Hosting control panel: Enable a free SSL option (e.g., Let’s Encrypt) for both yourdomain.com and www.yourdomain.com.
- Server with Certbot:
  1) Install certbot per your OS.
  2) For Nginx: sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
  3) For Apache: sudo certbot --apache -d yourdomain.com -d www.yourdomain.com
  4) Test renewal: sudo certbot renew --dry-run

Let’s Encrypt issuance: HTTP-01 challenges require inbound port 80 to be reachable from the public internet. Ensure firewalls/CDNs route /.well-known/acme-challenge requests to your origin during issuance.

If HTTP-01 cannot reach your origin (CDN/WAF blocks, maintenance windows), use DNS-01 validation with your DNS provider’s API to automate issuance (e.g., certbot-dns-cloudflare, certbot-dns-route53, or ACME clients with DNS plugins).

If using a CDN (e.g., Fastly, Cloudflare): Terminate TLS at the CDN and upload/manage certificates there. After confirming site-wide HTTPS is working, consider enabling HSTS (e.g., max-age=31536000; includeSubDomains; preload) at the CDN or server.

Caution on HSTS/preload: Enable HSTS preload only after confirming stable HTTPS across all subdomains. Preload is hard to roll back and can cause site inaccessibility if misconfigured. Submit to hstspreload.org only after verification.

Cloudflare-specific: Set SSL/TLS mode to Full (Strict). Do not use Flexible, as it causes HTTP at origin and can break secure admin and mixed-content policies. Full (Strict) requires a valid certificate on your origin. Install a publicly trusted cert (e.g., Let’s Encrypt) or a Cloudflare Origin Certificate on your server before enabling Full (Strict).

Cloud: For Adobe Commerce on cloud infrastructure, manage certificates and TLS termination with Fastly. See: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/cdn/fastly/fastly-ssl-tls.html

### Step 4: Set canonical redirects at the web server or hosting panel

Enable a 301 redirect to your canonical host and HTTPS in your hosting panel/CDN (often called “Force WWW/Non-WWW” and “Force HTTPS”). If editing Nginx, add a 301 from the non-canonical to canonical host. Example (for www canonical):

```
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://www.yourdomain.com$request_uri;
}

# Also redirect HTTPS requests for the non-canonical host
server {
    listen 443 ssl http2;
    server_name yourdomain.com;
    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;
    return 301 https://www.yourdomain.com$request_uri;
}
```

Create separate server blocks for each host (canonical and non-canonical). Use exact server_name matches and ensure only the canonical host serves Magento. Optionally mark a single HTTP server block as default_server to catch unmatched hosts and return 301 to the canonical domain.

Apache example (non-www canonical): place before Magento rewrite rules in the VirtualHost or .htaccess.

```
RewriteEngine On
# Redirect www to non-www
RewriteCond %{HTTP_HOST} ^www\.yourdomain\.com$ [NC]
RewriteRule ^(.*)$ https://yourdomain.com/$1 [L,R=301]
# Force HTTPS (supports proxies)
RewriteCond %{HTTPS} !=on [OR]
RewriteCond %{HTTP:X-Forwarded-Proto} !https
RewriteRule ^(.*)$ https://%{HTTP_HOST}/$1 [L,R=301]
```

Important:
- Handle canonical and HTTPS redirects at the edge (CDN/web server) to avoid double redirects. Set Magento’s “Auto-redirect to Base URL” to No in Step 5 unless you cannot control redirects at the edge.
- A single 301 hop to the canonical HTTPS URL improves crawl efficiency and preserves link equity (SEO). Single-hop redirects also preserve referral data and reduce page load time, which can lift conversion rates and SEO rankings.

Reference Magento sample configs to ensure correct rewrites:
- Nginx: https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/setup/web-server/nginx.html
- Apache: https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/setup/web-server/apache.html

Cloud: On Adobe Commerce on cloud infrastructure, configure redirects via Fastly VCL/edge settings. See: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/cdn/fastly/custom-domains.html

### Step 5: Update Magento Base URLs to HTTPS and canonical domain

In Magento Admin:
1) Go to Stores > Settings > Configuration > General > Web.
2) In the scope switcher (upper-left), open the dropdown and select the Website you want to configure. If you run different domains per store view (e.g., en.example.com and fr.example.com), switch scope to the specific Store View and set Base URLs there. Ensure Add Store Code to URLs = No to prevent duplicate URLs. Repeat for each Website or Store View that uses a different domain.

Scope tip: Configure Base URLs at Website scope. Only use Store View scope if a store view has its own domain (e.g., language-specific sites).

3) Under Base URLs, set Base URL to https://yourdomain.com/ or https://www.yourdomain.com/ (match your canonical).
4) Under Base URLs (Secure), set Base URL (Secure) to the same HTTPS URL. Enable “Use Secure URLs on Storefront” and “Use Secure URLs in Admin”.
5) Under General > Web > URL Options, set Auto-redirect to Base URL = No (recommended when redirects are handled at CDN/web server).
6) Under General > Web > Search Engine Optimization, set Use Web Server Rewrites = Yes (removes index.php from URLs when your server rewrites are configured).
7) Static/Media/CDN: Leave Base URL for Static View Files and Base URL for User Media Files empty to inherit the Base URL, unless you are using a CDN. If using a CDN, set:
   - Base URL for Static View Files: https://cdn.yourdomain.com/static/
   - Base URL for User Media Files: https://cdn.yourdomain.com/media/
   Then enable signing for cache-busting. In production, set via CLI or configuration management: bin/magento config:set dev/static/sign 1, then commit app/etc/config.php (use bin/magento app:config:dump to export). The Developer section is hidden in Production mode. Purge the CDN after saving.
   - Caution: If you use app:config:dump, it exports many settings to app/etc/config.php and locks them for Admin edits. Do NOT export Base URLs to config.php for multi-environment setups. On Cloud, prefer environment variables (CONFIG__web__secure__base_url, etc.) or set per environment via Admin.
8) Click Save Config, then go to System > Tools > Cache Management and click Flush Magento Cache.
9) Default Cookie Settings (align with your canonical domain): Stores > Configuration > General > Web > Default Cookie Settings
   - Cookie Domain: .yourdomain.com (for subdomain sharing) or leave blank to scope to the current host
   - Cookie Path: /
   - Use HTTP Only: Yes
   - Use Secure Cookies: Yes (after HTTPS works end-to-end)

Recovery if locked out due to incorrect Base URL:
- CLI recovery (default scope):
  - bin/magento setup:store-config:set --base-url="https://yourdomain.com/" --base-url-secure="https://yourdomain.com/" --use-secure=1 --use-secure-admin=1
  - bin/magento cache:flush
- CLI recovery (per-website scope):
  - bin/magento config:set web/unsecure/base_url "https://yourdomain.com/" --scope=websites --scope-code=YOUR_WEBSITE_CODE
  - bin/magento config:set web/secure/base_url "https://yourdomain.com/" --scope=websites --scope-code=YOUR_WEBSITE_CODE
  - bin/magento cache:flush
  - Use the actual website code (e.g., base for the main website).
- SQL (advanced): Update core_config_data paths web/unsecure/base_url and web/secure/base_url for the correct scope, then clear cache.

Cloud: On Adobe Commerce on cloud infrastructure, ensure Base URLs match the Fastly-configured domains and that redirects are handled at Fastly. Prefer setting Base URLs via environment variables instead of committing environment-specific URLs: for example, CONFIG__web__unsecure__base_url=https://www.yourdomain.com/ and CONFIG__web__secure__base_url=https://www.yourdomain.com/. Redeploy after setting. See: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/configure/env/stage/variables-global.html

### Step 6: Set store email identities in Magento

In Magento Admin:
1) Go to Stores > Settings > Configuration > General > Store Email Addresses.
2) For each identity (General Contact, Sales Representative, Customer Support, Custom Emails), set Sender Name (your brand) and Sender Email (e.g., support@yourdomain.com, sales@yourdomain.com).
   - Scope note: Use the Store View scope switcher to configure Sender Name and Sender Email for each store view that has a different brand or domain.
3) Go to Stores > Settings > Configuration > General > Contacts. Set Enable Contact Us = Yes, Send Emails To = your support address, and Email Sender = the appropriate Store Email Identity (e.g., Customer Support).
4) Click Save Config.

### Step 7: Configure SPF, DKIM, DMARC (and MX) for your domain

In your DNS:
- MX (inbound email): Add MX records for your mailbox provider (e.g., Google Workspace, Microsoft 365). Example (Workspace):
  - Host/Name = @, Value = ASPMX.L.GOOGLE.COM., Priority = 1 (plus the additional MX hosts per provider docs)
  Ensure SPF includes your mailbox provider only if it sends mail on your behalf.
- SPF (single TXT record at root): v=spf1 include:YOUR_EMAIL_PROVIDER include:YOUR_ESP -all
  - Only include providers you actively use. Do not include both your workspace email (e.g., Google/Microsoft) and an ESP unless mail is sent from both. Ensure total DNS lookups ≤10 across all includes.
  - Examples:
    - Google Workspace only: v=spf1 include:_spf.google.com -all
    - Microsoft 365 only: v=spf1 include:spf.protection.outlook.com -all
    - Google Workspace + SendGrid: v=spf1 include:_spf.google.com include:sendgrid.net -all
- DKIM: Add the TXT or CNAME records provided by your email/ESP (e.g., s1._domainkey.yourdomain.com) exactly as instructed.
- DMARC: TXT at _dmarc.yourdomain.com. Start with monitoring and alignment:
  - Example: v=DMARC1; p=none; rua=mailto:dmarc@yourdomain.com; ruf=mailto:dmarc-forensic@yourdomain.com; adkim=s; aspf=s; fo=1; pct=100
  - Many providers limit or redact forensic (ruf) reports; use them optionally for debugging. After 1–2 weeks of clean reports, tighten to p=quarantine or p=reject. Moving to p=reject after validation helps block phishing that impersonates your store.

Alignment and reputation tips:
- Ensure the From address domain (e.g., orders@yourdomain.com) matches the domain you authenticated with your ESP (SPF/DKIM) to achieve DMARC alignment and maximize inbox placement.
- Best practice for reputation isolation: Consider using a subdomain for marketing (e.g., m.yourdomain.com) and keep transactional on the root or a separate subdomain. This isolates reputation to safeguard order emails.

Monitoring:
- Use a DMARC analyzer to review aggregate (rua) reports.
- Add your domain to Google Postmaster Tools to monitor reputation and deliverability.
- Configure a custom Return-Path/bounce domain with your ESP (e.g., bounce.yourdomain.com) and verify it. Set the SMTP/ESP module to use this return-path so bounces are processed and DMARC alignment is maintained.

### Step 8: Choose and configure your email sending method

Recommended (best deliverability): Use an ESP (e.g., SendGrid, Mailgun, Amazon SES) with a Magento integration.
- Install a reputable SMTP or ESP integration module via Composer (from your ESP or a well-supported community module). Then configure it in Admin under Stores > Settings > Configuration (module path varies by vendor). Set host and port per your ESP: typically port 587 with STARTTLS (TLS) or port 465 with SSL. Use the module’s Test feature to verify connectivity.
- Install via Composer (example):
  - composer require mageplaza/module-smtp
  - bin/magento module:enable Mageplaza_Smtp
  - bin/magento setup:upgrade
  - bin/magento cache:flush
  Then configure at Stores > Configuration > Mageplaza Extensions > SMTP and send a test email.
- Example configuration paths:
  - Mageplaza SMTP: Stores > Configuration > Mageplaza Extensions > SMTP
  - MagePal SMTP: Stores > Configuration > Advanced > System > SMTP Configuration and Credentials
- Prefer official provider modules or API-based sending (e.g., provider SDK) when available for better throughput, retries, suppression handling, and reporting.
- Selection tips (tie to outcomes):
  - Volume < 50k emails/month: shared IP on a reputable ESP (fast start, no warm-up).
  - Volume 50k–300k/month: consider dedicated IP(s) with a 2–3 week warm-up; prioritize providers with strong deliverability tooling and feedback loops.
  - Volume > 300k/month or strict compliance needs: dedicated IP pool + subdomain segmentation; ensure SLAs and account support.
  - Typical cost range: ~$0.10–$1.50 per 1,000 emails depending on tier and features.
- Important: Ensure only one SMTP/ESP module is enabled at a time to avoid transport conflicts.

Dedicated IP warm-up (if applicable):
- Ramp volume gradually over 2–3 weeks, starting with your most engaged recipients (recent purchasers, frequent openers). Example ramp: 2k/day → 5k/day → 10k/day → 20k+/day.
- Monitor ESP dashboards (blocks, spam complaints, bounce rate). Slow the ramp if complaint or bounce rates spike.

Not recommended for production: Host-provided SMTP. Many hosts block outbound SMTP or have poor IP reputation, which can harm deliverability and DMARC alignment. If you must use it temporarily, obtain SMTP host, port (587 TLS or 465 SSL), username, and password from your host and verify outbound mail is permitted.

Cloud note: On Adobe Commerce on cloud infrastructure, use an ESP via SMTP/API; host SMTP is not supported and port 25 is blocked.

### Step 9: Enable and test Magento transactional emails

In Magento Admin:
1) Go to Stores > Settings > Configuration > Advanced > System > Mail Sending Settings. Set "Disable Email Communications" = No. Set "Set Return Path" = Specified and "Return-Path Email" = a bounce-capable address managed by your ESP (e.g., bounce@yourdomain.com), unless your ESP module overrides this with a verified return-path domain.
2) Go to Stores > Settings > Configuration > Sales > Sales Emails and ensure emails are enabled for Orders, Invoices, Shipments, and Credit Memos. For each section, set Email Sender to the appropriate Store Email Identity (e.g., Sales Representative or Customer Support). Under Sales Emails > General Settings, set Asynchronous Sending = Yes (recommended). Save Config.
3) Ensure Magento cron is running; otherwise emails will not send.
   - Verify cron: crontab -u <magento_user> -l and ensure bin/magento cron:run is scheduled every minute. If not installed, run: bin/magento cron:install
   - Check last run times in var/log/cron.log and system.log
   - Cloud note: Adobe Commerce on cloud infrastructure manages cron via ece-tools. Do not run bin/magento cron:install on Cloud. See: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/configure/app/properties/crons.html
4) On the storefront, place a small test order or use the Contact Us form to send a message to your personal inbox. Verify the From address, brand name, and that the email arrives successfully.

### Step 10: Validate DNS, redirects, and authentication

- Check DNS (Unix/macOS):
  - dig +short A yourdomain.com
  - dig +short CNAME www.yourdomain.com
  - If www is an A record: dig +short A www.yourdomain.com
  - dig +short TXT _dmarc.yourdomain.com
- Check DNS (Windows):
  - nslookup -type=A yourdomain.com
  - nslookup -type=CNAME www.yourdomain.com
  - If www is an A record: nslookup -type=A www.yourdomain.com
  - nslookup -type=TXT _dmarc.yourdomain.com
- Confirm redirects:
  - Browser test: Visit http://yourdomain.com and http://www.yourdomain.com and verify a single 301 to the canonical HTTPS domain.
  - CLI test: curl -I http://yourdomain.com and curl -I http://www.yourdomain.com — you should see a single 301 to https://<canonical-host>/ with no intermediate hops.
- Validate TLS: Test at https://www.ssllabs.com/ssltest/ and aim for an A grade.
- Email authentication: Open the headers of a received test email and confirm SPF=PASS, DKIM=PASS, DMARC=PASS.
  - View full headers: Gmail: open message > More (⋮) > Show original. Outlook (Windows): File > Properties > Internet headers.
- ESP validation: In your ESP dashboard, confirm test messages show Delivered (250 OK), and check bounces/complaints to ensure feedback loops and suppression are working.
- In Magento, clear caches again after any changes: System > Tools > Cache Management → Flush Magento Cache.

## Verification

To confirm everything is working correctly:

1. Review the checks in Step 10 (DNS, redirects/TLS, email authentication, and caches)
2. Navigate to the Admin panel and verify your changes are visible and saved
3. Place a live test order and ensure the Order and Shipment emails are delivered promptly
4. Verify Magento cron is installed and running (see Step 9) so queued transactional emails are delivered
5. Monitor KPIs for 1–2 weeks: delivery rate, open rate, support tickets about missing emails, and organic CTR to canonical URLs

Business Signoff:
- Delivery rate ≥ 98% (ESP dashboard)
- Order/shipment open rate ≥ 60%
- Support tickets about missing emails ↓ ≥ 50% vs baseline
- Organic CTR to canonical URLs ↑ vs baseline

Go-Live Checklist (printable):
- DNS: Apex and www records point to origin or CDN as designed; MX/SPF/DKIM/DMARC valid
- TLS/SSL: Valid certs (A grade on SSL Labs); HSTS considered after verification
- Redirects: Single 301 hop to canonical HTTPS host from all entry points
- Magento: Base URLs (secure) set at correct scope; Web Server Rewrites enabled; cookies configured; caches flushed
- Email: Store Email Identities set; SMTP/ESP module configured (only one enabled); Return-Path set/verified; Asynchronous Sending enabled; cron operational
- ESP: Test emails show Delivered; bounces/complaints processed; IP warm-up plan (if dedicated IP)
- Analytics/SEO: Sitemaps updated and submitted; Search Console/Bing properties verified; top landing URLs tested for single-hop redirects

SEO follow-up (within 24–48 hours of changes):
- Marketing > SEO & Search > Sitemap: Regenerate your sitemap so it uses the canonical HTTPS domain
- Submit the updated sitemap URL in Google Search Console and Bing Webmaster Tools
- If changing the primary domain, add the new property in Search Console and (where applicable) use the Change of Address tool
- Verify 301s for your top 50 landing URLs return a single hop to the canonical host
- Robots: Ensure production settings allow indexing. Go to Stores > Configuration > General > Design > Search Engine Robots. Set Default Robots to "INDEX, FOLLOW". If using a custom robots.txt (Content > Design > Configuration), update canonical sitemap and host.

## Common Issues and Solutions

- Emails not sending
  - Check Stores > Settings > Configuration > Advanced > System > Mail Sending Settings → Disable Email Communications = No
  - Verify SMTP/ESP module credentials and encryption; ensure only one SMTP/ESP module is enabled
  - Ensure Magento cron and system cron are running; review var/log/exception.log
- SPF failures
  - Use a single SPF TXT record; include only providers you actively use; keep DNS lookups ≤10
  - Remove permissive a or mx unless required by your providers
- DKIM failures
  - Confirm selector names match the ESP’s instructions and use TXT or CNAME as specified
  - Allow DNS propagation and re-test
- Redirect loops or multiple 301s
  - Handle redirects at CDN/web server and set Magento Auto-redirect to Base URL = No
  - Ensure Magento Base URLs exactly match your canonical domain and protocol
- Mixed content warnings
  - Clear caches; ensure all Base URLs are HTTPS and update any hardcoded http assets in custom themes/blocks
- Admin lockout after enabling secure admin or changing Base URLs
  - Revert Base URLs via CLI/DB as noted in Step 5 (use correct website scope), then recheck SSL and redirects

## Related Resources

- Adobe Commerce Admin User Guide: https://experienceleague.adobe.com/docs/commerce-admin/user-guides/home.html
- Adobe Commerce on cloud infrastructure (Fastly and domains): https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/cdn/fastly/custom-domains.html
- Fastly TLS/SSL for cloud projects: https://experienceleague.adobe.com/docs/commerce-cloud-service/user-guide/cdn/fastly/fastly-ssl-tls.html
- Google Postmaster Tools: https://postmaster.google.com/
- SSL Labs Server Test: https://www.ssllabs.com/ssltest/
