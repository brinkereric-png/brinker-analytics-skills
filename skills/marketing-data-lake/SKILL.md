---
name: marketing-data-lake
description: Build a brand-owned marketing data lake. Use when a DTC brand wants its orders, ad spend, GA4, and Search Console data in one warehouse it controls (BigQuery by default), loaded on a schedule and ready to query.
---

# Build your own marketing data lake

This skill walks you, the agent, through putting a brand's marketing data in one warehouse the brand owns. You'll load orders, ad-platform data, GA4, and Search Console, then join them so the user can ask plain questions of their own data.

Work one source at a time. Confirm each load before moving on.

## Step 0: Fill in the blanks

Ask the user for each value. Never write a real key, token, or password into a file, chat log, or repo. Store secrets in Google Cloud Secret Manager. Prefer attached service identities (workload identity) over downloaded keys. Treat environment variables only as a way to inject secrets at runtime, and keep them out of logs and shell history.

| Blank | Example |
|---|---|
| `{{GCP_PROJECT_ID}}` | `my-brand-data` |
| `{{DATASET}}` | `marketing_raw` |
| `{{REGION}}` | `US` |
| `{{SHOPIFY_STORE}}` | `mybrand.myshopify.com` |
| `{{GA4_PROPERTY_ID}}` | `123456789` |
| `{{GSC_SITE_URL}}` | `sc-domain:mybrand.com` |
| `{{META_AD_ACCOUNT_ID}}` | `act_1234567890` |
| `{{GOOGLE_ADS_CUSTOMER_ID}}` | `123-456-7890` |
| `{{OTHER_SOURCES}}` | TikTok, Klaviyo, Amazon, etc. |

Skip any source the user doesn't use. Ask what else they pay to see their own numbers in; that's a candidate source.

## Step 1: Create the warehouse

1. Confirm the user has a Google Cloud project with billing on and the BigQuery API enabled.
2. Create datasets: `{{DATASET}}` for raw loads and `marketing_models` for joined tables, both in `{{REGION}}`. Plan for the native exports in Step 2 to create or use their own datasets (`analytics_{{GA4_PROPERTY_ID}}`, `searchconsole`, the Google Ads transfer dataset). Put them in the same `{{REGION}}` so you can join them.
3. Create a service account for your custom loaders (Step 3) and give it only these roles:
   - **BigQuery Job User** on the execution project `{{GCP_PROJECT_ID}}`. This role can only be granted on a project, folder, or organization, not on a dataset.
   - **BigQuery Data Editor** on only the datasets it writes to (`{{DATASET}}`, and `marketing_models` if it builds tables).
   Have the user make any IAM grant themselves if you lack permission.
4. This loader account doesn't cover the native exports. Each export in Step 2 needs its own setup permissions and export identity, listed there.

## Step 2: Load the native sources first (cloud usage charges may apply)

These exports have no connector fee. BigQuery still charges for storage and queries, so tell the user before you turn them on.

- **GA4:** The person linking must be an Editor or above on the GA4 property and have permission to link the Cloud project. In GA4 Admin > Product links > BigQuery, link `{{GA4_PROPERTY_ID}}` to `{{GCP_PROJECT_ID}}`, choose `{{REGION}}`, and turn on the daily export. Events land in `analytics_{{GA4_PROPERTY_ID}}.events_*`. Data starts within about 24 hours of setup, and the export doesn't backfill anything from before the link. Daily tables can still update for late events for up to 2 calendar days plus today. Standard properties have a 1 million events per day cap on the daily export. If a property keeps going over it, the export pauses, so monitor for that. Setup guide: https://support.google.com/analytics/answer/9823238
- **Search Console:** Enable the BigQuery API and BigQuery Storage API on the project. Grant `search-console-data-export@system.gserviceaccount.com` the BigQuery Job User and BigQuery Data Editor roles on the project. Then, as a property owner, go to Search Console Settings > Bulk data export, point `{{GSC_SITE_URL}}` at the project, and choose the dataset location. Tables land in `searchconsole` (`searchdata_site_impression`, `searchdata_url_impression`). The first export can take up to 48 hours. There's no data from before setup, so use the Search Console API or the reports for history. Setup guide: https://support.google.com/webmasters/answer/12917675
- **Google Ads:** Set up a BigQuery Data Transfer Service transfer for `{{GOOGLE_ADS_CUSTOMER_ID}}`. The person setting it up needs transfer-configuration permissions in Cloud and read access to the Ads account. Backfill covers dated reporting data only, within Google Ads retention. From June 1, 2026, daily, weekly, and hourly data is kept for 37 months, except reach and frequency data, which is kept for 3 years. Each backfill request covers at most 180 days, and requests run one at a time. The refresh window defaults to 7 days and can be set from 1 to 30. Match-table snapshots aren't backfilled. Setup guide: https://cloud.google.com/bigquery/docs/google-ads-transfer

## Step 3: Load orders and other ad platforms

For Shopify, Meta, TikTok, and the rest, pick one with the user:

- **Managed connector** (hosted sync): a hosted ELT tool syncs into BigQuery on a schedule.
- **Own script** (scheduled API job): a small scheduled job (Cloud Run job or Cloud Function plus Cloud Scheduler) that calls each API and appends to BigQuery.
  - Shopify: Admin GraphQL API, `orders` with line items, discounts, refunds, and customer ID. The `read_orders` scope only reaches the last 60 days of orders. For older history, the app also needs approved `read_all_orders`, which doesn't work on its own. Bulk operations fetch history asynchronously, but they don't get around these scopes. Check protected customer data approval and that bulk operations are enabled for the app. Pull only the customer fields you need.
  - Meta: Marketing API Insights at ad level, daily `time_increment=1`, with spend, impressions, clicks, and actions. Pull ad creative into its own table, with the ad and creative IDs and a retrieval timestamp. Creative fields depend on format. Read `object_story_spec`, or `asset_feed_spec` for dynamic creative, and store missing fields as null. Media URLs can expire, so they aren't a permanent archive. Check permissions and fields against your API version: https://developers.facebook.com/docs/graph-api/reference/adaccount/insights/

For every custom loader:

- Use read-only scopes the source account has authorized.
- Follow pagination and wait for async jobs to finish.
- Retry with backoff within rate limits, and checkpoint progress so a failed run can resume.
- Land raw data with a `_loaded_at` timestamp. Leave out or redact fields you don't need, especially personal customer data. Never write credentials, tokens, or auth headers into raw payloads or logs.
- Limit access to raw datasets to the loader account and the few people who need it. Give everyone else access to the modeled views only.
- Agree with the user on how long raw customer data is kept, for example 13 months. Enforce it with a table or partition expiration, and delete customer records when the brand's policy or a customer's request requires it.
- Ad platforms restate recent metrics, and orders change through refunds and edits. Reload a rolling window, for example the last 7 to 30 days, on every run. A daily append can't pick up later changes by itself.
- Deduplicate each staged batch, then MERGE it into the table on its full source grain: account or store, object ID, every selected dimension, and date where it applies. A bare ID plus date isn't enough.
- Leave tables owned by the native exports alone. Model on top of them instead.

## Step 4: Model the joins

First, write down the definitions: reporting timezone (Search Console dates are Pacific time), currency conversion rule, how refunds count against revenue, what revenue includes, and how guest and new customers are classified. Aggregate each source to the chosen date and grain before joining, so the joins don't multiply rows.

Build views in `marketing_models`:

- `daily_spend`: date, channel, campaign, ad, spend, clicks, impressions, across all ad platforms in one currency.
- `daily_orders`: date, orders, revenue net of refunds, new vs returning customers. Calling someone a first-time customer needs enough order history, and 60 days of orders can't tell you that.
- `daily_search`: date, query, page, clicks, impressions, position. Build it from `searchdata_url_impression`, and don't treat the site and URL tables as interchangeable. Rows repeat keys, so always aggregate. Keep anonymized-query rows in totals. Compute position as `SAFE_DIVIDE(SUM(sum_position), SUM(impressions)) + 1`, never as an average of row averages.
- `daily_summary`: spend, orders, revenue, and blended cost per order by date. Blended cost per new customer is spend divided by distinct eligible new customers. Show it as null when there are no new customers. It isn't attributed or incremental acquisition cost.

Check each view against the source UI on a date that's no longer updating. Line up definitions, filters, attribution settings, currency, and timezone first. Some differences are built in. GA4's interface adds modeling and identity features the export doesn't have, so its numbers won't always match. Write down the expected differences, look into any gap you can't explain, and tell the user about it rather than hiding it.

## Step 5: Query it

Offer starter questions and write the SQL:

- What did we spend by channel last week, and what was blended cost per new customer?
- Which Search Console queries gained clicks this month?
- Which ad creatives had the most spend, and what did their copy say?

Point out recurring manual reports these queries can replace, and dashboards or subscriptions the user may no longer need.

## Step 6: Write the operating instructions

Write a short README for the user: each source, how it loads, its schedule, where secrets live, and how to rerun a failed day, and who watches for failed or paused loads. Test one rerun before you hand off. The brand should be able to run this without you.

---

If you want your marketing data in one place without managing the build yourself, shoot me a note to talk through it: [Schedule a call](https://brinkeranalytics.com/insights/fractional-analytics-lead-90-days-dtc#contact). Nothing to prepare.
