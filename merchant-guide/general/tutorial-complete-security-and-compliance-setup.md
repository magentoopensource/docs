---
title: "Complete Security and Compliance Setup"
description: "Comprehensive tutorial on securing your store and meeting compliance requirements"
type: tutorial
audience:
  - "Magento merchants"
  - "New Magento users"
difficulty: Intermediate–Advanced
estimated_time: "60–90 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
tags:
  - tutorial
  - security
  - high-impact
last_updated: "2025-09-16T17:18:12.062Z"
---

# Complete Security and Compliance Setup

Scope: Magento Open Source/Adobe Commerce 2.4+. This tutorial covers platform configuration and operational best practices. It is not legal advice. Consult your QSA/legal counsel for PCI DSS, GDPR/CCPA compliance requirements.

## Why This Tutorial Matters

This tutorial gives you a proven, step-by-step process to harden your Magento store, reduce fraud/bot traffic, and align with PCI and privacy expectations—without needing deep developer skills.

**Business Impact:** Reduce fraud and account takeover risk, avoid compliance fines, and protect revenue by hardening your store against common attacks.

**What You'll Achieve:** A hardened Admin, bot-resistant storefront, HTTPS-only site, PCI-aligned payment setup, and a repeatable security maintenance checklist.

## Learning Journey Overview

### Your Situation
You manage or administer a Magento store and need a fast, reliable way to improve security, reduce PCI scope, and put continuous monitoring in place—without disrupting sales.

### What You'll Learn
By completing this tutorial, you will:

- Enable and verify Admin Two-Factor Authentication (2FA)
- Configure Google reCAPTCHA for Admin and storefront forms
- Enforce HTTPS on storefront and Admin
- Harden Admin security settings (lockouts, session, password policies)
- Configure CAPTCHA for customers and password policy
- Turn on Cookie Restriction Mode and set SameSite/HTTPOnly flags
- Choose lower-PCI-scope payment methods and document your SAQ path
- Enroll your site in the Magento Security Scan and set up alerts
- Create a monthly security and compliance maintenance routine

### Success Criteria
You'll know you've succeeded when:

- Admin login requires 2FA for all users
- reCAPTCHA challenges appear on Admin login and customer forms as configured
- Storefront and Admin load only over HTTPS; no mixed content errors
- Security Scan shows no critical issues
- Selected payment method does not collect card data on your servers (scope reduced to SAQ A or A-EP per provider)
- Admin lockout and password policies are enforced and tested
- Cookie consent displays and SameSite/HTTPOnly settings verified

### Time Investment
- **Estimated time:** 60–90 minutes
- **Skill level after completion:** Confident Intermediate–Advanced security admin for Magento
- **Business value unlock:** In ~60–90 minutes, you reduce PCI scope, block common attack paths, and implement continuous scanning—protecting revenue and lowering compliance effort.

## Before We Start

### Who This Is For
This tutorial is designed for:

- Store owners and operations managers responsible for security
- Technical admins with Magento Admin access (Intermediate–Advanced)
- Merchants preparing for PCI DSS self-assessment (SAQ A/A-EP)
- Required experience: Comfortable navigating Magento Admin and changing configuration; developer or hosting support recommended for firewall/CDN tasks

### What You Need
Make sure you have:

- Magento 2.4.x (Open Source or Adobe Commerce)
- Magento Admin account with full permissions
- Google reCAPTCHA Site and Secret keys (v2 Checkbox and/or v3 Invisible)
- Access to your payment gateway account (e.g., PayPal, Braintree, Adyen) to confirm hosted/redirect settings
- Access to CDN/WAF or hosting panel (optional) for IP allowlisting and HTTPS/HSTS
- A maintenance window to test login policies without impacting customers

### Preparation Checklist
Before starting, complete these preparation steps:

- Notify your team of the maintenance window
- Ensure you have a second Admin user as a backup
- Save current configuration (screenshot key settings)
- Have reCAPTCHA keys ready
- Confirm you can access email for 2FA and admin notifications

## Step-by-Step Learning Path

Follow these steps in order. Estimated time per step is included.

1) Enforce HTTPS (5–10 min)
- Admin path: Stores > Configuration > General > Web > Base URLs (Secure)
- Set:
  - Use Secure URLs on Storefront = Yes
  - Use Secure URLs in Admin = Yes
- Save Config
- Clear cache: System > Cache Management > Flush Magento Cache
- Verify: Visit storefront and Admin; confirm https:// and no mixed content warnings

2) Harden Admin Security (10–15 min)
- Admin path: Stores > Configuration > Advanced > Admin > Security
- Set:
  - Admin Account Sharing = No
  - Add Secret Key to URLs = Yes
  - Max Login Failures to Lockout = 5 (or lower per policy)
  - Lockout Time (minutes) = 30
  - Password Lifetime (days) = 90 (optional, per policy)
  - Session Lifetime (seconds) = 3600 (adjust to business needs)
- Save Config and Flush Magento Cache
- Verify: Using a test admin, attempt multiple failed logins to confirm lockout behavior

3) Enable and Verify 2FA for Admin (5–10 min)
- Admin path: Stores > Configuration > Security > 2FA
- Action:
  - Enable 2FA providers required by your policy (e.g., Google Authenticator, WebAuthn/U2F)
- Save Config
- Each Admin user: log out and back in to pair a device
- Verify: All Admin users are prompted for 2FA on next login

4) Configure Google reCAPTCHA (10–15 min)
- Admin reCAPTCHA: Stores > Configuration > Security > Google reCAPTCHA Admin
  - Enable for Admin Login and Forgot Password forms
  - Enter Site Key and Secret Key (Admin)
- Storefront reCAPTCHA: Stores > Configuration > Security > Google reCAPTCHA Storefront
  - Enable for Create Account, Login, Forgot Password, Contact Us, Newsletter, Checkout forms as needed
  - Enter Site Key and Secret Key (Storefront)
  - For v3 Invisible, set Minimum Score (start at 0.3–0.5 and adjust)
- Save Config and Flush Magento Cache
- Verify: Forms show reCAPTCHA (v2) or pass silently (v3) and block obvious bot submissions

5) Enable CAPTCHA and Customer Password Policies (5–10 min)
- Admin CAPTCHA: Stores > Configuration > Advanced > Admin > CAPTCHA
  - Enable CAPTCHA = Yes; configure font size, attempts, timeout
- Customer CAPTCHA: Stores > Configuration > Customers > Customer Configuration > CAPTCHA
  - Enable on Create Account, Login, Forgot Password
- Customer Password Options: Stores > Configuration > Customers > Customer Configuration > Password Options
  - Min Password Length = 8–12; Required Character Classes = 3 (align to policy)
- Save Config and Flush Magento Cache
- Verify: Customer flows require CAPTCHA; password rules enforced at registration

6) Secure Cookies and Consent (5–10 min)
- Admin path: Stores > Configuration > General > Web > Default Cookie Settings
- Set:
  - Use HTTP Only = Yes
  - Cookie SameSite = Lax (use None + Secure only if required by embedded third-party flows)
  - Cookie Restriction Mode = Yes (enables consent banner)
- Save Config and Flush Magento Cache
- Verify: New session sets HttpOnly cookies; consent notice appears for first-time visitors

7) Reduce PCI Scope via Payment Settings (5–10 min)
- Action:
  - Choose hosted/redirect payment methods that avoid card data touching your servers (e.g., Redirect to Provider, Hosted Fields)
  - In each payment method's settings: Stores > Configuration > Sales > Payment Methods, enable redirect/hosted options as available
  - Document your intended SAQ type (A or A-EP) based on provider architecture; confirm with your gateway/QSA
  - Hosted/redirect payment options generally reduce PCI scope (often SAQ A or A-EP), decreasing compliance workload
- Verify: Checkout redirects or renders provider-hosted fields; no card data fields are native to your page

8) Enroll in Magento Security Scan (5–10 min)
- Go to: https://securityscan.magento.com and sign in
- Add your site and verify ownership (meta tag or DNS)
- Schedule weekly scans and email alerts to your team
- Verify: First scan completes and issues are reviewed/triaged

9) Access Control Review (5–10 min)
- Admin paths:
  - System > Permissions > User Roles — apply least privilege
  - System > Permissions > All Users — remove or disable unused accounts; require 2FA for all
  - System > Integrations — disable or delete unused integrations and tokens
- Verify: No stale users; roles are minimal and documented

10) Create a Monthly Security Routine (5 min)
- Establish a calendar reminder: review scan results, user accounts, and apply security patches via Composer (with your developer/host)
- Document owners and SLAs for responding to alerts

## Practice and Reinforcement

Now that you've learned the core process, let's reinforce your skills:

- Review Security Scan report and remediate findings
- Rotate Admin accounts for staff who left; audit roles
- Test 2FA recovery for at least one admin
- Submit 5–10 form entries to validate reCAPTCHA performance and adjust thresholds
- Confirm HTTPS redirects and renew TLS certificates before expiry
- Check for available security patches and schedule updates

## What You've Accomplished

🎉 **Congratulations!** You have successfully:

- Locked down your Admin with strong policies and 2FA
- Reduced bot and spam submissions with reCAPTCHA and CAPTCHA
- Enforced HTTPS across your site
- Lowered PCI scope by using hosted/redirect payment options
- Enrolled in automated Security Scan monitoring
- Established a monthly security and compliance routine

### Business Impact

- Lower chargeback and fraud risk by blocking automated attacks
- Reduce compliance effort and audit scope by avoiding storage/processing of cardholder data
- Protect brand trust with visible security cues (HTTPS, consent)
- Minimize downtime from compromised Admin accounts through strong access controls

### Skills Gained
You now have the ability to:

- Configure and enforce 2FA, lockout, session, and password policies
- Implement and tune reCAPTCHA and CAPTCHA across key forms
- Enforce HTTPS and validate secure cookie settings
- Select payment configurations that reduce PCI scope and document SAQ approach
- Monitor your storefront with Magento Security Scan and act on alerts
- Run a repeatable monthly security and compliance review

## Next Steps in Your Journey

Now that you've mastered this process, here's how to build on your success:

### Immediate Actions
- Set calendar reminders for monthly security checks
- Share the new login and 2FA policy with your team
- Assign an owner for weekly scan reviews with a 48-hour remediation SLA

### Level Up Your Skills
- Work with your host/CDN to enable a WAF and rate limiting
- Implement IP allowlisting for Admin via firewall/CDN where possible

### Advanced Applications
- Establish a patch management process with staging and rollback
- Conduct a quarterly user access review and incident response drill

## When Things Don't Go as Expected

Even experienced merchants encounter challenges. Here's how to handle common situations:

- Locked out of Admin due to 2FA: Ask another Admin to go to System > Permissions > All Users, edit your user, and click Reset 2FA. If no other Admin exists, a developer can run: bin/magento twofactorauth:reset <username>
- reCAPTCHA blocks valid customers: Lower v3 score threshold (e.g., from 0.5 to 0.3) or temporarily switch to v2 Checkbox; re-test and monitor
- Too many Admin lockouts: Increase Max Login Failures slightly or extend Lockout Time and educate staff on password managers
- Checkout issues after enabling HTTPS: Clear caches/CDN, ensure base URLs are https in Stores > Configuration > General > Web
- Cookie consent not showing: Confirm Cookie Restriction Mode = Yes and clear cache

## Continue Learning

### Related Tutorials
- Configure Google reCAPTCHA in Magento
- Setting up Two-Factor Authentication (2FA)

### How-To Guides
- Harden Admin Security Settings
- Reduce PCI Scope with Hosted Payments

### Reference Materials
- Adobe Commerce/Magento Security Best Practices: https://experienceleague.adobe.com/docs/commerce-operations/security/best-practices.html
- Magento Security Scan Tool: https://securityscan.magento.com
- PCI DSS SAQ Guidance: https://www.pcisecuritystandards.org/merchants/

## Summary

### Key Takeaways
- Admin hardened with 2FA, lockouts, and least-privilege roles
- Storefront protected by reCAPTCHA/CAPTCHA and secure cookies
- HTTPS enforced across Admin and storefront
- PCI scope reduced via hosted/redirect payment options
- Ongoing monitoring set up with Magento Security Scan and monthly reviews

### Remember
- Small configuration changes can materially reduce fraud and compliance burden
- Revisit your settings quarterly or after major updates to stay aligned with policy and risk

---

*Found this tutorial helpful? [Share your feedback](https://community.magento.com) or [suggest improvements](https://github.com/magento/magento2/issues).* 

*Questions or stuck? [Get help from the community](https://community.magento.com) or [contact support](https://support.magento.com/).*