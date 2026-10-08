# Call Records

Get your company's call records, one page at a time. These are the calls behind every number
on the Dwesk PBX dashboard, so this is how you fill your own dashboard. Without a cursor you
start from your first call. With a cursor you get only the records that changed after that
cursor.

<div class="endpoint"><span class="method get">GET</span><span class="path">/call-records</span></div>

| | |
| --- | --- |
| Base | Gateway. See [Base URLs](/guide/environments). |
| Auth | [API key](/guide/authentication#api-keys) |

Read [Building Your Dashboard](/guide/build-your-dashboard) first. It explains when to call
this endpoint and how to save what you get back.

## Parameters

| Parameter | In | Required | Type | Description |
| --- | --- | --- | --- | --- |
| `X-Transaction-Id` | Header | Yes | String | Your ID for this request. See [transaction IDs](/guide/authentication#transaction-ids). |
| `cursor` | Query | No | String | The `nextCursor` from your last response, or the `cursor` from a [feed delivery](/webhooks/call-records-feed). Leave it out to start from your first call. |
| `limit` | Query | No | Integer | How many records per page. Default `500`, maximum `1000`. |

A cursor is a bookmark. Save it and send it back exactly as you got it. Don't try to read it
or make your own, because what's inside can change. Cursors never expire.

## Request

::: code-group

```bash [cURL]
curl 'https://gateway.dxesk.cloud/pabx/v1/call-records?limit=500' \
  -H 'X-API-Key: <YOUR_API_KEY>' \
  -H 'X-Transaction-Id: 3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64'
```

```ts [TypeScript]
import { randomUUID } from "node:crypto";

interface CallRecordsPage {
  records: CallRecord[];
  nextCursor: string;
  hasMore: boolean;
  nextPollAfter: string | null;
}

async function getCallRecords(opts: {
  cursor?: string;
  limit?: number;
}): Promise<CallRecordsPage> {
  const url = new URL(`${process.env.DWESK_GATEWAY_BASE}/call-records`);
  if (opts.cursor) url.searchParams.set("cursor", opts.cursor);
  if (opts.limit) url.searchParams.set("limit", String(opts.limit));

  const res = await fetch(url, {
    headers: {
      "X-API-Key": process.env.DWESK_API_KEY!,
      "X-Transaction-Id": randomUUID(),
    },
  });

  if (res.status === 429) {
    const wait = Number(res.headers.get("Retry-After") ?? "60");
    await new Promise((r) => setTimeout(r, wait * 1000));
    return getCallRecords(opts);
  }
  if (!res.ok) throw new Error(`call-records ${res.status}`);

  return (await res.json()) as CallRecordsPage;
}
```

:::

## Success response

`200 OK`

```json
{
  "records": [
    {
      "id": 387395,
      "channelId": "1759906859.48213",
      "companyId": 1005,
      "callerNumber": "764897587",
      "agentCli": 779994374,
      "callType": "CONNECT_DIAL",
      "callStatus": "ANSWERED",
      "callStartTime": "2026-10-08T09:00:59+05:30",
      "agentRingStartTime": "2026-10-08T09:01:04+05:30",
      "agentConnectTime": "2026-10-08T09:01:09+05:30",
      "callEndTime": "2026-10-08T09:01:13+05:30",
      "connected": true,
      "callFlowId": 667,
      "updatedAt": "2026-10-08T09:01:13+05:30"
    }
  ],
  "nextCursor": "eyJwIjoiYyIsInQiOiIyMDI2LTEwLTA4VDA5OjAxOjEzIiwiaSI6Mzg3Mzk1fQ",
  "hasMore": false,
  "nextPollAfter": "2026-10-08T09:06:13+05:30"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `records` | Array | Up to `limit` records. Empty if nothing changed. |
| `nextCursor` | String | Send this the next time you call. It is always there, even when `records` is empty. |
| `hasMore` | Boolean | `true` means there are more records. Call again right away. |
| `nextPollAfter` | String \| null | Only set when `hasMore` is `false`. If you call with `nextCursor` before this time, you get `429`. |

### Call record

| Field | Type | Description |
| --- | --- | --- |
| `id` | Long | Dwesk's record ID. This is the call ID shown on the Dwesk dashboard. |
| `channelId` | String | Unique for each call. Use it as your key. |
| `companyId` | Long | Your company ID. |
| `callerNumber` | String | The customer's number. For `OUTGOING` calls this is the number of the agent who made the call. |
| `agentCli` | Long \| null | The agent's phone number, without the first `0`. `null` if no agent took part. |
| `callType` | String | Which way the call went. See [call types](#call-types). |
| `callStatus` | String | What has happened to the call so far. See [call statuses](#call-statuses). |
| `callStartTime` | String \| null | When the call reached the PBX. |
| `agentRingStartTime` | String \| null | When an agent's phone started ringing. |
| `agentConnectTime` | String \| null | When the agent answered. |
| `callEndTime` | String \| null | When the call ended. `null` while the call is still going. |
| `connected` | Boolean \| null | Whether the call was connected to an agent. |
| `callFlowId` | Number \| null | The IVR flow or queue the call went through. |
| `updatedAt` | String | When this version of the record was made. If you have two versions of a call, keep the one with the later `updatedAt`. |

All times are Sri Lanka time, ISO 8601 with `+05:30`.

::: tip Match agentCli to your own agents
Records don't include agent names or emails. `agentCli` is the agent's phone number as a
number, so `0779994374` comes as `779994374`. Keep your own list of agent numbers and look
the agent up from it.
:::

### Call types

| Value | Meaning | Direction |
| --- | --- | --- |
| `INCOMING` | A call to your hotline. | Inbound |
| `CONNECT_DIAL` | An inbound call put through to an agent by Connect Agent. | Inbound |
| `QUEUE_DIAL` | An inbound call that went into a queue to wait for a free agent. | Inbound |
| `OUTGOING` | A call made by one of your agents. | Outbound |

### Call statuses

| Value | Meaning | Can it still change? |
| --- | --- | --- |
| `WAITING` | The caller is in a queue, waiting for a free agent. | Yes |
| `RINGING` | An agent's phone is ringing. | Yes |
| `ANSWERED` | An agent answered. | No |
| `TRANSFER` | An agent answered, then transferred the call. | No |
| `CONFERENCE` | An agent answered, then added another person to the call. | No |
| `VOICEMAIL` | The caller left a voicemail. | No |
| `FAILED` | The call ended without reaching an agent. | No |

A `WAITING` or `RINGING` record will be sent again when it changes. If you ever get a status
that isn't in this list, save the record anyway and treat it like one that can still change.

## Paging and how often to call

Pages come oldest first. During your first fetch, keep calling while `hasMore` is `true`.
You don't need to wait between calls.

When `hasMore` is `false`, you have everything, and the response includes `nextPollAfter`, 5
minutes later. If you call with that `nextCursor` before then, you get `429`. Normally you
won't need to call at all, because the [feed](/webhooks/call-records-feed) sends you the
changes. You call this endpoint when you've missed something.

You can use a cursor from a feed delivery straight away.

::: info You may get the same record twice
Each call looks back a little before your cursor, so you can get a record you already have.
We do this on purpose, so a record that was saved a few seconds late on our side isn't
missed. Save one row per `channelId` and it won't matter. See
[A record can change](/guide/build-your-dashboard#a-record-can-change).
:::

## Error responses

This endpoint uses normal HTTP status codes. The body has a `code`, a `message` and your `transactionId`.

| Status | `code` | Why |
| --- | --- | --- |
| `400` | `INVALID_CURSOR` | The cursor was changed, cut off, or belongs to another company. Go back to an earlier cursor you saved. |
| `400` | `INVALID_LIMIT` | `limit` is less than 1 or more than 1000. |
| `400` | `MISSING_TRANSACTION_ID` | The `X-Transaction-Id` header is missing or empty. |
| `400` | `INVALID_TRANSACTION_ID` | The transaction ID is longer than 64 characters or has characters that aren't allowed. |
| `401` | `UNAUTHORIZED` | The `X-API-Key` header is missing or the key is wrong. |
| `429` | `TOO_EARLY` | You already have everything and called before `nextPollAfter`. Wait `Retry-After` seconds. |
| `429` | `RATE_LIMITED` | You sent more than 60 requests in a minute. Wait `Retry-After` seconds. |
| `500`, `503` | `SERVER_ERROR` | Try the same request again later, with the same cursor. |

```json
{ "code": "TOO_EARLY", "message": "Caught up. Next poll allowed at 2026-10-08T09:06:13+05:30", "transactionId": "3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64" }
```

A failed request never moves you forward or back. Trying again with the same cursor is always
safe.
