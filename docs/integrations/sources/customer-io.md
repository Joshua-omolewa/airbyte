# Customer.io

<HideInUI>

This page contains the setup guide and reference information for the [Customer.io](https://customer.io/) source connector.

</HideInUI>

The Customer.io source connector uses the [Customer.io App API](https://docs.customer.io/integrations/api/app/) to sync campaign (automation), campaign action, and newsletter metadata from one Customer.io workspace.

## Prerequisites

- A Customer.io App API key for the workspace you want to sync. Track API keys don't work with this connector.
- The Account Admin role, or the account-level **Manage API credentials** permission, to create or view API keys in Customer.io.
- The data center region of your Customer.io account: US or EU.

## Setup guide

### Step 1: Create a Customer.io App API key

1. In Customer.io, go to **Account Settings** > **API Credentials**.
2. Create an App API key for the workspace you want to sync. If you restrict the key's scope, make sure it includes the data you want to sync.
3. Copy the key right away and store it securely. Customer.io shows App API keys only once.

For more information, see Customer.io's [API credentials](https://docs.customer.io/accounts/settings/managing-credentials/) documentation.

### Step 2: Find your account region

If you're an Account Admin, your account region appears in Customer.io under **Settings** > **Account Settings** > **Data and Privacy**. Other roles can't see the region. Customer.io doesn't redirect App API requests between regions, so a key used against the wrong region fails with a `401` error. For more information, see [Account regions](https://docs.customer.io/accounts/settings/data-centers/).

### Step 3: Set up the Customer.io source in Airbyte

<FieldAnchor field="app_api_key">

1. For **App API Key**, enter the key you created in Step 1.

</FieldAnchor>

<FieldAnchor field="region">

1. For **Region**, select **US** or **EU** to match your Customer.io account. The default is **US**.

</FieldAnchor>

<FieldAnchor field="start_date">

1. (Optional) For **Start Date**, enter a UTC date and time in the format `YYYY-MM-DDTHH:MM:SSZ`. The connector only emits records whose `updated` timestamp is at or after this date. Leave it blank to sync all records.

</FieldAnchor>

1. Select **Set up source**. Airbyte tests the connection by reading the `campaigns` stream.

## Supported sync modes

The Customer.io source connector supports the following [sync modes](https://docs.airbyte.com/platform/using-airbyte/core-concepts/sync-modes/):

- Full Refresh | Overwrite
- Full Refresh | Append
- Incremental | Append
- Incremental | Append + Deduped

## Supported streams

| Stream | Customer.io endpoint | Primary key | Cursor field | Pagination |
| :--- | :--- | :--- | :--- | :--- |
| `campaigns` | [List automations](https://docs.customer.io/integrations/api/app/tag/automations/listcampaigns/) (`GET /v1/campaigns`) | `id` | `updated` | None |
| `campaigns_actions` | [List automation actions](https://docs.customer.io/integrations/api/app/tag/automations/listcampaignactions/) (`GET /v1/campaigns/{id}/actions`) | `id` | `updated` | Cursor |
| `newsletters` | [List newsletters](https://docs.customer.io/integrations/api/app/tag/newsletters/listnewsletters/) (`GET /v1/newsletters`) | `id` | `updated` | Cursor, 100 per page |

The Customer.io UI calls campaigns "automations." The API and this connector still use the name `campaigns`.

`campaigns_actions` is a child stream of `campaigns`. The connector reads the list of campaigns, then requests the actions of each campaign. If a campaign is deleted between those two requests, Customer.io returns a `404` error for its actions and the connector skips that campaign instead of failing the sync.

### Incremental sync behavior

None of these Customer.io endpoints can filter by time, so incremental syncs are client-side:

- Every sync, including incremental syncs, reads the full list of campaigns, campaign actions, and newsletters from the API. Incremental sync reduces the number of records written to the destination, not the number of API requests.
- The connector emits records whose `updated` timestamp is at or after the saved cursor minus a one-hour lookback window. The lookback catches records edited while an earlier sync was running. As a result, each incremental sync can re-emit records updated in the hour before the saved cursor. Use **Incremental | Append + Deduped** to remove these duplicates in the destination. With **Incremental | Append**, they appear as repeated rows.
- `campaigns_actions` keeps a separate cursor for each campaign.
- If you set a **Start Date** in the future, the connection test still passes, but until that date arrives, syncs emit only records edited while the sync runs.

### Known limitations

- `campaigns_actions.id` is a string (for example, `"18"`), while `campaigns.actions[].id` is an integer. Cast one of them to join the two streams.
- The `campaigns` fields `audience.person_filters`, `audience.relationship_filters`, `object_attribute_triggers`, and `relationship_attribute_triggers` have no declared type in the schema. Customer.io's API reference documents them as objects but shows JSON strings in its examples, so the connector passes them through as received.

## Rate limits

Most Customer.io App API endpoints, including the ones this connector uses, allow 10 requests per second. See [Rate limits](https://docs.customer.io/integrations/api/app/#rate-limits). The connector limits itself to 10 requests per second across all streams.

Other App API traffic in your workspace counts toward the same limit, so Customer.io can still return `429` errors during a sync. The connector retries `429` responses after the delay in the `Retry-After` header, or with exponential backoff when that header is absent. It also retries `500`, `502`, `503`, and `504` errors. If the errors persist, the sync fails after up to 30 retries, which can take up to about 17 minutes.

`campaigns_actions` makes at least one request per campaign, so workspaces with many campaigns take longer to sync.

## Troubleshooting

| Error | Cause and fix |
| :--- | :--- |
| `401` Unauthorized: "Customer.io rejected the App API key" | The key is invalid or is a Track API key, the **Region** doesn't match your account, or your account restricts API access by IP address and Airbyte's IP addresses aren't allowed. Use a valid App API key, select the correct region, and check the IP allowlist. |
| `403` Forbidden: "Customer.io denied this App API key access" | The key's scope doesn't include the requested data, or an IP allowlist blocks the request. Use a key whose scope includes the data you want to sync, and check the IP allowlist. |

### IP allowlist

If your Customer.io account restricts API access by IP address, add the [Airbyte Cloud IP addresses](https://docs.airbyte.com/platform/operating-airbyte/ip-allowlist) to the allowlist on the Customer.io **Manage API Credentials** page, for the workspace you want to sync. If you self-manage Airbyte, add the outbound IP addresses of your Airbyte deployment instead. See [Restrict API access by IP address](https://docs.customer.io/accounts/settings/managing-credentials/#restrict-api-access-by-ip-address).

## Changelog

<details>
  <summary>Expand to review</summary>

| Version | Date       | Pull Request                                                   | Subject                                     |
|:--------|:-----------| :------------------------------------------------------------- |:--------------------------------------------|
| 0.5.0 | 2026-10-07 | [88129](https://github.com/airbytehq/airbyte/pull/88129) | Add rate limiting, Retry-After retries, clearer authentication errors, a one-hour lookback, and missing fields |
| 0.4.17 | 2026-10-06 | [87812](https://github.com/airbytehq/airbyte/pull/87812) | Update dependencies |
| 0.4.16 | 2026-09-29 | [87129](https://github.com/airbytehq/airbyte/pull/87129) | Update dependencies |
| 0.4.15 | 2026-09-22 | [86584](https://github.com/airbytehq/airbyte/pull/86584) | Update dependencies |
| 0.4.14 | 2026-09-15 | [85996](https://github.com/airbytehq/airbyte/pull/85996) | Update dependencies |
| 0.4.13 | 2026-09-08 | [85433](https://github.com/airbytehq/airbyte/pull/85433) | Update dependencies |
| 0.4.12 | 2026-08-18 | [84549](https://github.com/airbytehq/airbyte/pull/84549) | Update dependencies |
| 0.4.11 | 2026-08-11 | [83899](https://github.com/airbytehq/airbyte/pull/83899) | Update dependencies |
| 0.4.10 | 2026-08-04 | [83394](https://github.com/airbytehq/airbyte/pull/83394) | Update dependencies |
| 0.4.9 | 2026-07-28 | [82865](https://github.com/airbytehq/airbyte/pull/82865) | Update dependencies |
| 0.4.8 | 2026-07-21 | [82368](https://github.com/airbytehq/airbyte/pull/82368) | Update dependencies |
| 0.4.7 | 2026-07-14 | [81795](https://github.com/airbytehq/airbyte/pull/81795) | Update dependencies |
| 0.4.6 | 2026-06-30 | [81036](https://github.com/airbytehq/airbyte/pull/81036) | Update dependencies |
| 0.4.5 | 2026-06-23 | [80394](https://github.com/airbytehq/airbyte/pull/80394) | Update dependencies |
| 0.4.4 | 2026-06-16 | [79829](https://github.com/airbytehq/airbyte/pull/79829) | Update dependencies |
| 0.4.3 | 2026-06-09 | [79257](https://github.com/airbytehq/airbyte/pull/79257) | Update dependencies |
| 0.4.2 | 2026-06-02 | [78639](https://github.com/airbytehq/airbyte/pull/78639) | Update dependencies |
| 0.4.1 | 2026-05-08 | [77895](https://github.com/airbytehq/airbyte/pull/77895) | Upgrade the base image to source-declarative-manifest 7.18.1 |
| 0.4.0 | 2026-05-08 | [77819](https://github.com/airbytehq/airbyte/pull/77819) | Add pagination, incremental sync, and EU region support |
| 0.3.19  | 2025-08-20 | [65113](https://github.com/airbytehq/airbyte/pull/65113) | Update logo                                 |
| 0.3.18  | 2025-05-10 | [60049](https://github.com/airbytehq/airbyte/pull/60049) | Update dependencies                         |
| 0.3.17  | 2025-05-03 | [58875](https://github.com/airbytehq/airbyte/pull/58875) | Update dependencies                         |
| 0.3.16  | 2025-04-19 | [57766](https://github.com/airbytehq/airbyte/pull/57766) | Update dependencies                         |
| 0.3.15  | 2025-04-05 | [57225](https://github.com/airbytehq/airbyte/pull/57225) | Update dependencies                         |
| 0.3.14  | 2025-03-29 | [56546](https://github.com/airbytehq/airbyte/pull/56546) | Update dependencies                         |
| 0.3.13  | 2025-03-22 | [55918](https://github.com/airbytehq/airbyte/pull/55918) | Update dependencies                         |
| 0.3.12  | 2025-03-08 | [55311](https://github.com/airbytehq/airbyte/pull/55311) | Update dependencies                         |
| 0.3.11  | 2025-03-01 | [54942](https://github.com/airbytehq/airbyte/pull/54942) | Update dependencies                         |
| 0.3.10  | 2025-02-22 | [54374](https://github.com/airbytehq/airbyte/pull/54374) | Update dependencies                         |
| 0.3.9   | 2025-02-15 | [51670](https://github.com/airbytehq/airbyte/pull/51670) | Update dependencies                         |
| 0.3.8   | 2025-01-11 | [51062](https://github.com/airbytehq/airbyte/pull/51062) | Update dependencies                         |
| 0.3.7   | 2025-01-04 | [50582](https://github.com/airbytehq/airbyte/pull/50582) | Update dependencies                         |
| 0.3.6   | 2024-12-21 | [49999](https://github.com/airbytehq/airbyte/pull/49999) | Update dependencies                         |
| 0.3.5   | 2024-12-14 | [49490](https://github.com/airbytehq/airbyte/pull/49490) | Update dependencies                         |
| 0.3.4   | 2024-12-12 | [48923](https://github.com/airbytehq/airbyte/pull/48923) | Update dependencies                         |
| 0.3.3   | 2024-11-04 | [48225](https://github.com/airbytehq/airbyte/pull/48225) | Update dependencies                         |
| 0.3.2   | 2024-10-28 | [47464](https://github.com/airbytehq/airbyte/pull/47464) | Update dependencies                         |
| 0.3.1   | 2024-08-16 | [44196](https://github.com/airbytehq/airbyte/pull/44196) | Bump source-declarative-manifest version    |
| 0.3.0   | 2024-08-15 | [44158](https://github.com/airbytehq/airbyte/pull/44158) | Refactor connector to manifest-only format  |
| 0.2.15  | 2024-08-12 | [43889](https://github.com/airbytehq/airbyte/pull/43889) | Update dependencies                         |
| 0.2.14  | 2024-08-10 | [43513](https://github.com/airbytehq/airbyte/pull/43513) | Update dependencies                         |
| 0.2.13  | 2024-08-03 | [43185](https://github.com/airbytehq/airbyte/pull/43185) | Update dependencies                         |
| 0.2.12  | 2024-07-27 | [42631](https://github.com/airbytehq/airbyte/pull/42631) | Update dependencies                         |
| 0.2.11  | 2024-07-20 | [42219](https://github.com/airbytehq/airbyte/pull/42219) | Update dependencies                         |
| 0.2.10  | 2024-07-13 | [41808](https://github.com/airbytehq/airbyte/pull/41808) | Update dependencies                         |
| 0.2.9   | 2024-07-10 | [41389](https://github.com/airbytehq/airbyte/pull/41389) | Update dependencies                         |
| 0.2.8   | 2024-07-09 | [41225](https://github.com/airbytehq/airbyte/pull/41225) | Update dependencies                         |
| 0.2.7   | 2024-07-06 | [40883](https://github.com/airbytehq/airbyte/pull/40883) | Update dependencies                         |
| 0.2.6   | 2024-06-29 | [40624](https://github.com/airbytehq/airbyte/pull/40624) | Update dependencies                         |
| 0.2.5 | 2024-06-28 | [38318](https://github.com/airbytehq/airbyte/pull/38318) | Make the connector compatible with Connector Builder |
| 0.2.4   | 2024-06-25 | [40369](https://github.com/airbytehq/airbyte/pull/40369) | Update dependencies                         |
| 0.2.3   | 2024-06-22 | [39953](https://github.com/airbytehq/airbyte/pull/39953) | Update dependencies                         |
| 0.2.2   | 2024-06-04 | [38980](https://github.com/airbytehq/airbyte/pull/38980) | [autopull] Upgrade base image to v1.2.1     |
| 0.2.1   | 2024-05-31 | [38812](https://github.com/airbytehq/airbyte/pull/38812) | [autopull] Migrate to base image and poetry |
| 0.2.0 | 2023-08-29 | [29385](https://github.com/airbytehq/airbyte/pull/29385) | Migrate TS CDK to Low code |
| 0.1.23 | 2021-11-09 | 126 | Add Customer.io source |

</details>
