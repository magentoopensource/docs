---
title: "How to Optimize Product Images"
description: "Upload, resize, and optimize product images for performance and conversion"
type: how-to
audience:
  - "Magento merchants"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Product catalog setup"
  - "Ability to purge Varnish/CDN cache (e.g., Fastly)"
  - "Admin role with permissions to Catalog and Content > Media > Media Gallery"
  - "If using a staging environment, deploy and validate there first"
last_updated: "2025-09-12T08:32:34.492Z"
tags:
  - how-to
  - product-management
  - performance
---

# How to Optimize Product Images

## Overview

Optimize product images to improve page speed, conversions, and SEO. A faster, cleaner product gallery reduces bounce rate and increases add-to-cart and revenue, especially for growing catalogs.

Set measurable goals: Aim for LCP under 2.5s on product detail pages (mobile) and reduce total image bytes by 30–50%. These improvements typically increase add-to-cart rate, revenue per session, and organic rankings.

Impact first (time-to-value):
- Enable CDN image optimization (30–60% byte reduction; 15–30 minutes)
- Compress existing assets (20–40% reduction; hours for large catalogs)
- Resize mis-sized images to target widths (reduces blurriness and bytes; ongoing)

Note on version compatibility:
- New Media Gallery Replace action is available in Magento 2.4.2+.
- Adobe Commerce on cloud supports Fastly Image Optimization (IO) for next‑gen formats and compression.
- Magento Open Source or self-hosted environments can use third-party CDNs for similar optimizations.

## Prerequisites

Before you begin, make sure you have:

- Access to Magento Admin Panel
- Product catalog setup
- Ability to purge Varnish/CDN cache (e.g., Fastly)
- Admin role with permissions to Catalog and Content > Media > Media Gallery
- If using a staging environment, deploy and validate there first
- Recommended: Use the ImageMagick adapter if available for better quality/performance. Set via CLI: bin/magento config:set dev/image/adapter imagemagick (flush cache after).
- Plan changes off-peak, back up originals, and keep a list of SKUs/images changed so you can roll back quickly if needed.

## What You'll Accomplish

By following this guide, you will:

- Optimize product images end-to-end (format, size, compression, filenames)
- Improve page speed, LCP, and visual quality to boost conversions

## Step-by-Step Instructions

### Step 0: Establish a baseline

Do this for 2–3 high-traffic PDPs:
- Open Chrome Incognito > DevTools > Network, check "Disable cache", select the Img filter.
- Reload the page; note total Transferred (images only) and identify the largest product image size.
- Run Lighthouse (Mobile) or PageSpeed Insights; record LCP and image-related opportunities (e.g., "Serve images in next-gen formats", "Efficiently encode images"). Save these results as your baseline.



### Step 1: Choose the right format per image

Quick win: If Fastly (or another CDN optimizer) is available, enable its image optimization first—this alone can reduce image payload by 30–60% within minutes.

Decide format before editing: Upload JPEG for photos and PNG when transparency is required. Out of the box (as of Magento 2.4.x), product image uploads accept JPEG/JPG, PNG, and GIF. WebP uploads for product images are not supported without customization. To serve WebP/AVIF to browsers, upload JPEG/PNG and enable image optimization on your CDN (e.g., Fastly IO on Adobe Commerce) so the CDN delivers WebP/AVIF derivatives automatically.

Adobe Commerce on cloud: Go to Stores > Configuration > Advanced > System > Full Page Cache > Fastly Configuration > Image Optimization. Enable WebP/AVIF and set quality (around 70–80). After enabling Image Optimization and saving, click Upload VCL to Fastly (if prompted) and Purge All to apply changes.

Magento Open Source/self-hosted: Use a CDN (e.g., Cloudflare, Akamai) to auto-convert and compress images at the edge.

Verification: In Chrome DevTools > Network, select a product image and check Headers > Content-Type. Confirm image/webp or image/avif is served. If JPEG/PNG is still served to compatible browsers, recheck CDN config and purge caches.



### Step 2: Resize to target dimensions

Find your target sizes in your theme’s view.xml (ask your developer), or measure rendered sizes on the storefront with DevTools. Prepare source images at ~2x the largest displayed width for each role (avoid upscaling). Do not crop to square unless your theme specifically uses square images.

Check your theme's image role definitions to determine target sizes and aspect ratios. In your theme, review app/design/frontend/<Vendor>/<theme>/etc/view.xml for roles such as product_page_image_medium, category_page_grid, and product_page_image_small. Use those sizes as guidance and prepare source images at ~2x the largest width to support high-DPI displays (avoid upscaling).

Open each source image in your editor. Do not crop to square unless your theme requires it. As examples only (verify in your theme): category images are often ~600 px wide, thumbnails ~300 px; swatch sizes vary by theme/CSS and can be adjusted via CSS.

Tip: Theme image sizes come from view.xml. Share this file with your implementer to align source image sizes and aspect ratios with the theme, preventing distortion or blurriness.



### Step 3: Compress and clean metadata

Use Squoosh, TinyJPG/TinyPNG, or ImageOptim to compress: set JPEG quality ~70–80; strip EXIF/metadata. Before exporting, convert images to the sRGB color profile to ensure consistent rendering across browsers and devices. Aim for under 200 KB per main image (under 100 KB ideal for grid). Visually compare to ensure quality is acceptable. After compression, zoom to 200% on key details (logos, stitching, text) and compare against the original to ensure artifacts are acceptable.

Bulk optimization: For large catalogs, export originals and run batch compression with ImageOptim CLI or Squoosh CLI. After re-upload, run `bin/magento catalog:images:resize` to pre-generate cached sizes before peak traffic. Note: catalog:images:resize is CPU and IO intensive. Run during off-peak hours or in maintenance mode. If available, use --batch-size to limit concurrent processing and monitor server load.



### Step 4: Apply clear, SEO-friendly filenames

Rename files with SKU and descriptive keywords, using hyphens (e.g., sku-12345-red-sneaker-side.jpg). Keep names lowercase; avoid spaces and special characters.

Why it matters: Descriptive filenames improve image search visibility and help deduplicate assets by SKU for faster operations.



### Step 5: Upload and assign image roles in the product

Go to Catalog > Products. Edit a product. In Images and Videos, drag-and-drop your optimized files. Set image roles as needed: Base Image, Small Image, Thumbnail, and Swatch. (Base Image is used on the PDP, Small Image on listings, Thumbnail in carts/mini cart.) Magento generates resized images per role automatically—pick the appropriate source image(s); you don't need to upload different sizes. Add meaningful Alt Text. Reorder images as needed. Click Save.

Alt Text: 80–125 characters. Describe product + variant (color/material/angle). No keyword stuffing. Make it unique per variant. Example: "Men's red leather sneaker, side view, size 10 (SKU 12345)".

Optional: Check Exclude to hide an image from the gallery while still using it for a role (e.g., Swatch). If you run multiple store views/languages, switch the Store View (upper-left) and enter localized Alt Text; uncheck Use Default Value to override.



### Step 6: Organize and, if needed, replace via Media Gallery

For multiple images, go to Content > Media > Media Gallery. Create folders by category/season. Upload optimized images. To replace a heavy existing file that’s reused, select it and use Replace to upload the lighter version, preserving the file path so references update automatically—this avoids broken links and saves time reassigning in CMS blocks, categories, and products.

Note: The Replace action is available in the New Media Gallery (Magento 2.4.2+). On older versions, upload the new file and reassign it in products or CMS content. If you do not see Replace, ensure the New Media Gallery is enabled (Stores > Configuration > Advanced > System > Media Gallery: set Enable Old Media Gallery to No) and you are on Magento 2.4.2 or later.



### Step 7: Flush image cache

- System > Cache Management > Click Flush Catalog Images Cache.
- Click Flush Page Cache. If you use Varnish, flush the Page Cache and trigger a Varnish purge.
- Purge your CDN cache. For Fastly: Stores > Configuration > Advanced > System > Full Page Cache > Fastly Configuration > Purge (Purge All).

Avoid using Flush Cache Storage on production systems.



### Step 8: Validate on the storefront

Open the product page and a category grid in a private window. Confirm images render quickly, look sharp, and are not stretched. Hover/zoom should work. Right-click > Inspect > Elements to confirm alt attributes are present. In DevTools > Network, confirm smaller image transfer sizes.

Test on mobile conditions: In Chrome DevTools > Network, enable Fast 3G or Slow 4G throttling and re-run Lighthouse on Mobile. Aim for LCP < 2.5s on PDPs and a 30–50% reduction in total image bytes vs. baseline.

Check Core Web Vitals details: Ensure product images have width/height attributes or CSS aspect-ratio to prevent layout shifts (CLS). For LCP, ensure the main above-the-fold product image is not lazy-loaded (loading='eager' or no lazy attribute). Coordinate with your developer if adjustments are needed.



### Step 9: Measure the improvement

Re-run Lighthouse or PageSpeed Insights. Compare LCP and total image bytes with your baseline. Record improvements. Target LCP < 2.5s (Mobile) and at least a 30–50% reduction in total image bytes on PDPs and category pages.

Track impact over 2–4 weeks: monitor Add-to-Cart rate, Conversion rate, Revenue per session, and Bounce rate in your analytics tool. Annotate the optimization date to attribute improvements.

If targets aren’t met, take these actions and retest: re-compress the heaviest images, enable or tune CDN image optimization (WebP/AVIF, quality 70–80), align aspect ratios with view.xml, and avoid upscaling. If possible, limit the number of above-the-fold images on PDPs.



## Verification

To confirm everything is working correctly:

1. In Admin > Catalog > Products, confirm images display with correct roles (Base/Small/Thumbnail/Swatch) and meaningful Alt Text.
1. On PDP and category pages (Incognito), confirm images load quickly, look sharp, and are not stretched or distorted.
1. In DevTools > Network (Img filter), confirm total image bytes decreased vs. baseline.
1. Re-run Lighthouse (Mobile) and confirm LCP improved relative to the baseline.

## Common Issues and Solutions

- Images look blurry: Ensure you didn't upscale small originals; provide source images ~2x the target width for high-DPI displays.
- Images look distorted: Match source aspect ratio to the theme’s roles defined in view.xml.
- New images not visible: Flush Catalog Images Cache, invalidate/purge Page Cache (Varnish), and purge your CDN.
- 404s on images: Verify pub/media permissions (e.g., 755 for directories, 644 for files) and correct filesystem ownership. If you use the recommended two‑user setup (separate file owner and web server group), set directories to 770 and files to 660. See Magento file permissions guidelines.
- Swatches not updating: Assign the Swatch role to the correct image and clear Full Page Cache.
- Swatches not updating the main image: Edit the swatch attribute (Stores > Attributes > Product > [attribute]) and set Update Product Preview Image to Yes under Storefront Properties. Reindex if required and flush Page Cache.

## Related Resources

- Magento/Adobe Commerce User Guide (Experience League): https://experienceleague.adobe.com/en/docs/commerce-admin
- Media Gallery: https://experienceleague.adobe.com/en/docs/commerce-admin/content-design/media/gallery
- Product Images: https://experienceleague.adobe.com/en/docs/commerce-admin/catalog/products/product-images
- Fastly Image Optimization (Adobe Commerce on cloud): https://experienceleague.adobe.com/en/docs/commerce-cloud-service/user-guide/cdn/fastly-image-optimization/overview
- CLI: catalog:images:resize: https://experienceleague.adobe.com/en/docs/commerce-operations/tools/cli-reference/commands#catalogimagesresize
- File system permissions: https://experienceleague.adobe.com/en/docs/commerce-operations/installation-guide/prerequisites/file-system-permissions

---

Need help? Contact Adobe Commerce Support at https://support.magento.com or review Troubleshooting: Media Gallery at https://experienceleague.adobe.com/en/docs/commerce-admin/content-design/media/gallery#troubleshoot
