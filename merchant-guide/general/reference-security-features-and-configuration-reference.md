---
title: "Security Features and Configuration Reference"
description: "Complete reference of security settings, permissions, and protection mechanisms"
type: reference
audience: ["Magento merchants", "New Magento users"]
tags:
  - reference
  - configuration
  - settings
  - security
last_updated: "2025-09-12T08:59:36.350Z"
---

# Security Features and Configuration Reference

## Overview

This reference catalogs Magento 2 security features and configuration options that merchants can use to harden Admin access, protect storefront interactions, and control permissions. Applying the correct settings enforces least-privilege permissions, reduces fraud and account takeover risk, and increases customer trust—improving conversion and lifetime value.

Scope: Applies to Adobe Commerce and Magento Open Source 2.4.x unless noted. Two-Factor Authentication is mandatory in 2.4+. Admin Action Logs are available in Adobe Commerce only.

Edition/Version differences:
- 2FA is required for Admin users in 2.4+.
- Admin Action Logs are available in Adobe Commerce only.
- Google reCAPTCHA v3 Invisible is supported on storefront forms only (not Admin).

Measure impact:
- Track failed Admin logins, customer account takeover incidents, bot checkout attempts, chargebacks tied to card testing, and reCAPTCHA challenge/score distributions. Reduced fraud chargebacks and support tickets, fewer compromised accounts, and lower infrastructure costs from bot traffic improve margin and conversion.

How to use this reference: For each security area, follow the Admin path provided, apply the recommended values, and complete the Verify steps to ensure the control works as expected.

Before you begin:
- Ensure you have at least two Admin users with full permissions.
- Communicate upcoming changes to all Admin users and schedule work during low traffic.
- Keep SSH/CLI access available to reset 2FA or unlock users if necessary.
- Test changes in a staging environment. For reCAPTCHA, use test keys/sandbox mode where applicable.
- Rollback: If you are locked out after Admin URL changes, retrieve the current Admin URI with `bin/magento info:adminuri` or update app/etc/env.php (`'backend' => ['frontName' => '...']`), then run `bin/magento cache:flush`.

Quick Wins Checklist (30–60 minutes):
- Change the Admin URL and ensure Add Secret Key to URLs is set to Yes (Stores > Settings > Configuration > Advanced > Admin > Security) to reduce automated admin credential stuffing risk.
- Enable 2FA for all Admin users (blocks the vast majority of account takeover attempts).
- Turn on Google reCAPTCHA for login/forgot password/checkout (reduces bot signups and card testing).
- Set Admin lockout and session timeout (limits brute force and unattended sessions).
- Enforce a customer password policy (reduces account takeover).


## Configuration

Quick navigation:
- [Admin access hardening](#admin-access-hardening)
- [Two-Factor Authentication (2FA)](#two-factor-authentication-2fa)
- [Google reCAPTCHA](#google-recaptcha)
- [Magento CAPTCHA (fallback)](#magento-captcha-fallback)
- [Customer password policy](#customer-password-policy)
- [Roles and permissions (least privilege)](#roles-and-permissions-least-privilege)
- [Admin Action Logs (Adobe Commerce only)](#admin-action-logs-adobe-commerce-only)
- [Session and cookie security](#session-and-cookie-security)
- [Network and transport](#network-and-transport)
- [Verification checklist](#verification-checklist)

### Admin access hardening

#### Authentication & Session Controls
- Path: Stores > Settings > Configuration > Advanced > Admin > Security
- Purpose: Strengthen Admin authentication and session controls.
- Recommended:
  - Add Secret Key to URLs: Yes
  - Admin Account Sharing: No
  - Password Reset Protection Type: By IP and Email
    - Note: If your admins work behind rotating IPs/VPNs, consider By Email to avoid blocked resets, and document a fallback CLI recovery process.
  - Max Login Failures to Lockout Account: 5 (adjust per risk tolerance)
  - Lockout Time (minutes): 30
  - Session Lifetime (seconds): 900 (recommended). Increase to 1800 only if admins routinely perform long-running tasks that time out.
  - Password Lifetime (days): 90 (or per company policy)
  - Max Number of Password Reset Requests: 3
  - Min Time Between Password Reset Requests (minutes): 10
- Steps:
  - Navigate to the path above.
  - Set each value as recommended; click Save Config.
  - Flush caches if prompted.
- Verify:
  - Attempt multiple failed Admin logins to confirm lockout after the set threshold and that the user-facing lockout message is displayed.
  - Log in and remain idle to confirm session expiration aligns with the configured lifetime.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x.

#### Admin URL/Path
- Path: Stores > Settings > Configuration > Advanced > Admin > Admin Base URL
- Purpose: Reduce automated discovery of the Admin and add entropy to the login surface.
- Recommended:
  - Use Custom Admin URL: Yes (set a unique subdomain or path under your domain)
  - Use Custom Admin Path: Yes (set a non-default path)
  - Note: Depending on web server/CDN rules, the old Admin URL may 302 redirect to the new path. This is expected when correctly configured.
  - Recovery: If you cannot access Admin after the change, run `bin/magento info:adminuri` or update app/etc/env.php (`'backend' => ['frontName' => 'your_path']`) on the server to recover. Clear caches after updating.
- Steps:
  - Enable and set the custom URL/path; Save Config.
  - Log out, then access the Admin using the new URL.
- Verify:
  - Confirm the old Admin URL no longer resolves to the login page.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x.

### Two-Factor Authentication (2FA)
- Path: Stores > Settings > Configuration > Security > 2FA
- Purpose: Require a second factor for Admin logins to block credential-only attacks.
- Notes: 2FA is required in 2.4+ for Admin users. Ensure at least one provider is enabled and document recovery procedures.
  - Recovery: If a user loses 2FA access, an Admin with appropriate permissions can reset 2FA enrollment for that user via CLI or by temporarily disabling the provider per the official 2FA documentation. Maintain at least two super admin users with recovery capability.
- Steps:
  - Enable at least one provider (for example, Google Authenticator, WebAuthn).
  - Save Config, then log out.
  - Log in and register the 2FA device when prompted.
- Verify:
  - Confirm that Admin login prompts for the second factor and that recovery codes or backup methods work as expected.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4+.

### Google reCAPTCHA
- Purpose: Mitigate automated bot activity on Admin and storefront forms while balancing friction vs conversion.
- Choosing a type:
  - Use v3 Invisible for checkout and high-conversion storefront forms to minimize friction; increase the v3 score threshold temporarily during active abuse.
  - Use v2 ("I'm not a robot") or v2 Invisible on forms under attack (for example, login) when you need stronger, challenge-based protection.
  - Admin Panel supports only v2 types. reCAPTCHA v3 Invisible is not supported for Admin forms in Magento 2.4.x.

#### Google reCAPTCHA v2 ("I'm not a robot")
- Path: Stores > Settings > Configuration > Security > Google reCAPTCHA v2 ("I'm not a robot")
- Supported forms:
  - Admin: Login, Forgot Password
  - Storefront: Create Account, Login, Forgot Password, Contact Us, and other supported forms
- Recommended:
  - Enable for Admin Login and Forgot Password.
  - Enable for storefront forms that see abuse and where a visible challenge is acceptable.
- Steps:
  - In Google reCAPTCHA, create v2 (checkbox) site and secret keys for your exact domain(s)/subdomains.
  - Enter keys, select desired forms, and Save Config.
  - If Content Security Policy (CSP) is enabled, allow google.com, gstatic.com, and recaptcha.net for script/frame/connect sources to prevent blocking.
- Verify:
  - Submit Admin/storefront login to confirm the checkbox challenge appears and functions.
  - Review reCAPTCHA dashboard for challenge and pass/fail metrics.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x.

#### Google reCAPTCHA v2 Invisible
- Path: Stores > Settings > Configuration > Security > Google reCAPTCHA v2 Invisible
- Supported forms:
  - Admin: Login, Forgot Password
  - Storefront: Create Account, Login, Forgot Password, Contact Us, and other supported forms
- Recommended:
  - Use for Admin forms to reduce friction while maintaining protection.
  - Use on storefront login/registration when you want fewer visible challenges.
- Steps:
  - In Google reCAPTCHA, create v2 Invisible site and secret keys for your exact domain(s)/subdomains.
  - Enter keys, enable for target forms, and Save Config.
  - If CSP is enabled, allow *.google.com, *.gstatic.com, and *.recaptcha.net in script-src, frame-src, and connect-src.
- Verify:
  - Confirm the invisible challenge triggers and allows legitimate submissions while blocking bots.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x.

#### Google reCAPTCHA v3 Invisible
- Path: Stores > Settings > Configuration > Security > Google reCAPTCHA v3 Invisible
- Supported forms:
  - Storefront only: Create Account, Login, Forgot Password, Contact Us, checkout steps (for example, Place Order)
- Recommended:
  - Use for checkout and other high-conversion storefront flows to minimize friction.
  - Start Score Threshold at 0.5–0.7; monitor and adjust by ±0.1 based on false positives/negatives.
- Steps:
  - In Google reCAPTCHA, create v3 site and secret keys for your exact domain(s)/subdomains.
  - Enter keys, enable the desired forms, set the Score Threshold, and Save Config.
  - If CSP is enabled, allow *.google.com, *.gstatic.com, and *.recaptcha.net in script-src, frame-src, and connect-src.
- Verify:
  - Submit storefront login/registration/checkout and confirm v3 triggers.
  - Review reCAPTCHA dashboard for score distributions and challenges; adjust threshold accordingly.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x. Not supported for Admin forms.

### Magento CAPTCHA (fallback)
- Path: Stores > Settings > Configuration > Customers > Customer Configuration > CAPTCHA
- Purpose: Provide a basic challenge mechanism when Google reCAPTCHA is not used or as a fallback.
- Recommended:
  - Use only if Google reCAPTCHA is not available for specific forms.
  - Warning: Do not enable Magento CAPTCHA and Google reCAPTCHA on the same form. Use only one protection mechanism per form to avoid conflicts.
- Steps:
  - Enable for applicable forms; set Difficulty and Number of Symbols; Save Config.
- Verify:
  - Confirm the CAPTCHA appears and blocks automated submissions.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x.

### Customer password policy
- Path: Stores > Settings > Configuration > Customers > Customer Configuration > Password Options
- Purpose: Improve customer account resilience against brute force and credential stuffing.
- Recommended:
  - Minimum Password Length: 12 (recommended for compliance; use 8+ at minimum if business constraints apply)
  - Required Character Classes Number: 3 or 4 (letters, digits, special characters, different case)
  - Forgot Email Sender and templates configured and tested; verify email deliverability (SPF/DKIM/DMARC) to avoid password reset abandonment
- Steps:
  - Set the options; Save Config.
- Verify:
  - Attempt to set weak passwords to ensure enforcement; test Forgot Password flow and email delivery.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x.

### Roles and permissions (least privilege)
- Path: System > Permissions > User Roles
- Purpose: Limit access to only what each job function requires.
- Steps:
  1) Go to System > Permissions > User Roles > Add New Role.
  2) Name the role and Save.
  3) Open the role, grant only required Resources, then Save Role.
  4) Go to System > Permissions > All Users, edit each user, and assign the new role.
- Business rationale:
  - Limit catalog pricing rights to specific roles to prevent costly pricing errors.
  - Restrict refund creation to senior support to reduce fraud exposure.
- Example role templates:
  - Customer Support: View/Edit Customers, View Orders, Create Credit Memos; no Catalog price changes.
  - Content Editor: CMS Pages/Blocks, Media Storage; no Sales or System config access.
- Verify:
  - Log in as a user with each role and confirm only intended menus/resources are accessible.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x.

### Admin Action Logs (Adobe Commerce only)
- Path: Stores > Settings > Configuration > Advanced > Admin > Admin Actions Logging
- Purpose: Track administrative actions for accountability and troubleshooting.
- Steps:
  - Enable Admin Actions Logging; Save Config.
  - Path to view: System > Actions Logs > Report
  - Retention and archiving (Commerce only): Define retention and archive/export cadence under System > Actions Logs > Archive to meet audit requirements (for example, 90–180 days retention with monthly archive/export).
- Verify:
  - Perform an Admin change (for example, update a CMS page) and confirm an entry appears in the report.
- Edition: Adobe Commerce only.

### Session and cookie security
- Path: Stores > Settings > Configuration > General > Web > Default Cookie Settings
- Purpose: Ensure cookies follow secure best practices to protect sessions.
- Recommended:
  - Use HTTP Only: Yes
  - Cookie Lifetime: 3600–7200 seconds
  - Cookie SameSite: Lax (or None + Secure if required by third-party embeds)
  - Cookie Restriction Mode: Enable as required for consent management
- Path: Stores > Settings > Configuration > General > Web > Base URLs (Secure)
- Recommended:
  - Use Secure URLs on Storefront: Yes
  - Use Secure URLs in Admin: Yes
- Path: Stores > Settings > Configuration > General > Web > URL Options
- Recommended:
  - Use SID on Frontend: No (prevents session IDs from appearing in URLs)
- Path: Stores > Settings > Configuration > Advanced > System > Session Storage Management
- Recommended:
  - Use secure, centralized session storage (for example, Redis or memcached) with authentication enabled.
- Steps:
  - Apply settings and Save Config.
- Verify:
  - In browser dev tools, confirm cookies are Secure and HttpOnly, and SameSite is set as configured.
  - Confirm all storefront and Admin pages load via HTTPS.
  - Ensure URLs do not include SID parameters; confirm sessions persist correctly with configured storage.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x.

### Network and transport
- Purpose: Reduce exposure of sensitive endpoints and mitigate abuse.
- Recommendations:
  - Restrict Admin access by IP using your CDN/WAF or web server.
  - For Adobe Commerce on cloud infrastructure, use project/environment access controls.
  - Enable and tune a WAF to block common attacks (for example, SQLi, XSS) and credential stuffing.
  - Apply rate limiting for API routes (REST/GraphQL) at your CDN/WAF. Throttle token generation endpoints and guest checkout abuse patterns. Ensure WAF policies include rules for /rest/* and /graphql with sensible per-IP thresholds.
  - Examples: rate-limit POST /graphql to 30 requests/minute/IP; restrict /admin* to office VPN IPs; block known credential-stuffing user agents with bot management.
  - Business impact: Bot management can reduce bot traffic by 30–70% on targeted stores; review WAF dashboards weekly to tune rules for conversion-safe blocking.
- Verify:
  - Confirm blocked IPs cannot reach the Admin URL.
  - Review WAF logs and dashboards for attempted threats and false positives; validate API rate limits are enforced without harming legitimate traffic.
- Edition/Version: Applies to Adobe Commerce and Magento Open Source 2.4.x.

### Verification checklist
- After each configuration change:
  - Log out and log back in to confirm expected prompts (for example, 2FA enrollment) and session behavior.
  - Intentionally fail Admin login to confirm lockout behavior and the user-facing lockout message.
  - Submit storefront forms to confirm reCAPTCHA triggers appropriately.
  - Monitor failed logins, reCAPTCHA scores, and WAF events to fine-tune settings.
  - 14-day tuning plan: Review reCAPTCHA score distributions, WAF false positives, and Admin lockout rates weekly; adjust thresholds and rules to minimize false positives during promotions/high-traffic events to protect conversion.


## See Also

- Security overview: https://experienceleague.adobe.com/docs/commerce-operations/security/overview.html
- Two-Factor Authentication: https://experienceleague.adobe.com/docs/commerce-admin/systems/security/2fa.html
- Google reCAPTCHA: https://experienceleague.adobe.com/docs/commerce-admin/systems/security/google-recaptcha.html
- Admin security settings: https://experienceleague.adobe.com/docs/commerce-admin/systems/security/security-admin.html
- Cookie and session settings: https://experienceleague.adobe.com/docs/commerce-admin/config/general/web.html
- Admin action logs (Commerce only): https://experienceleague.adobe.com/docs/commerce-admin/systems/action-logs/action-logs.html
- Security Scan: https://experienceleague.adobe.com/docs/commerce-operations/security/security-scan.html

---

*This reference was last updated on 2025-09-12T08:59:36.350Z. For the most current version-specific guidance, see the Adobe Commerce and Magento Open Source security docs: https://experienceleague.adobe.com/docs/commerce-operations/security/overview.html*
