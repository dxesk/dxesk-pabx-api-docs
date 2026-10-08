# Call Record Counts

Get the number of call records for each day, split by status and by type. Use it to check
that the calls you saved match the ones Dwesk has. If they match, your dashboard shows the
same numbers as ours.

<div class="endpoint"><span class="method get">GET</span><span class="path">/call-records/counts</span></div>

| | |
| --- | --- |
| Base | Gateway. See [Base URLs](/guide/environments). |
| Auth | [API key](/guide/authentication#api-keys) |

## Parameters

| Parameter | In | Required | Type | Description |
| --- | --- | --- | --- | --- |
| `X-Transaction-Id` | Header | Yes | String | Your ID for this request. See [transaction IDs](/guide/authentication#transaction-ids). |
| `from` | Query | Yes | String | First day, `yyyy-MM-dd`. |
| `to` | Query | Yes | String | Last day, `yyyy-MM-dd`. This day is included. Up to 31 days after `from`. |

A record is counted on the day its `callStartTime` falls on, in Sri Lanka time. Records
without a `callStartTime` are not counted.

## Request

::: code-group

```bash [cURL]
curl 'https://gateway.dxesk.cloud/pabx/v1/call-records/counts?from=2026-10-01&to=2026-10-07' \
  -H 'X-API-Key: <YOUR_API_KEY>' \
  -H 'X-Transaction-Id: 3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64'
```

```ts [TypeScript]
import { randomUUID } from "node:crypto";

interface CallRecordCountsDay {
  date: string;
  total: number;
  byStatus: Partial<Record<CallStatus, number>>;
  byType: Partial<Record<CallType, number>>;
}

async function getCallRecordCounts(from: string, to: string) {
  const url = new URL(`${process.env.DWESK_GATEWAY_BASE}/call-records/counts`);
  url.searchParams.set("from", from);
  url.searchParams.set("to", to);

  const res = await fetch(url, {
    headers: {
      "X-API-Key": process.env.DWESK_API_KEY!,
      "X-Transaction-Id": randomUUID(),
    },
  });
  if (!res.ok) throw new Error(`call-records/counts ${res.status}`);

  return (await res.json()) as { days: CallRecordCountsDay[] };
}
```

:::

## Success response

`200 OK`

```json
{
  "days": [
    {
      "date": "2026-10-01",
      "total": 412,
      "byStatus": { "ANSWERED": 336, "TRANSFER": 12, "CONFERENCE": 2, "FAILED": 40, "VOICEMAIL": 22 },
      "byType": { "INCOMING": 120, "CONNECT_DIAL": 180, "QUEUE_DIAL": 60, "OUTGOING": 52 }
    },
    {
      "date": "2026-10-02",
      "total": 0,
      "byStatus": {},
      "byType": {}
    }
  ]
}
```

| Field | Type | Description |
| --- | --- | --- |
| `days` | Array | One item for every day from `from` to `to`, even days with no calls. |
| `days[].date` | String | `yyyy-MM-dd`. |
| `days[].total` | Integer | Number of records that started on this day. |
| `days[].byStatus` | Object | Number of records for each `callStatus`. Statuses with 0 records are left out. |
| `days[].byType` | Object | Number of records for each `callType`. Types with 0 records are left out. |

## Comparing with your records

Count your own records for each day and compare:

```ts
const { days } = await getCallRecordCounts("2026-10-01", "2026-10-07");

for (const day of days) {
  const mine = await db.callRecord.count({
    where: { callStartTime: { gte: `${day.date}T00:00:00+05:30`, lt: nextDay(day.date) } },
  });

  if (mine !== day.total) {
    console.warn(`${day.date}: Dwesk ${day.total}, local ${mine}`);
  }
}
```

If `total` matches but `byStatus` doesn't, you have the right calls but some are old
versions. Usually that means a newer record didn't replace the old one. Check
[A record can change](/guide/build-your-dashboard#a-record-can-change).

If your `total` is lower, call [Call Records](/api/call-records) with your last saved cursor.

::: warning Only compare days that are over
Today's numbers change as calls move from `RINGING` to their final status, so they won't
match exactly while calls are going on. Compare yesterday and earlier.
:::

## Error responses

| Status | `code` | Why |
| --- | --- | --- |
| `400` | `INVALID_RANGE` | `from` or `to` is missing or not `yyyy-MM-dd`, `to` is before `from`, or the range is more than 31 days. |
| `400` | `MISSING_TRANSACTION_ID` | The `X-Transaction-Id` header is missing or empty. |
| `400` | `INVALID_TRANSACTION_ID` | The transaction ID is longer than 64 characters or has characters that aren't allowed. |
| `401` | `UNAUTHORIZED` | The `X-API-Key` header is missing or the key is wrong. |
| `429` | `RATE_LIMITED` | You sent more than 60 requests in a minute. Wait `Retry-After` seconds. |
| `500`, `503` | `SERVER_ERROR` | Try again later. |
