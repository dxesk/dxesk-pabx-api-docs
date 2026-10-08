# IVR Call Summary

Get the calls that went through one of your IVR flows, one page at a time. This is the same
data as the IVR Call Summary page in the Dwesk portal.

<div class="endpoint"><span class="method get">GET</span><span class="path">/ivr/call-summary</span></div>

| | |
| --- | --- |
| Base | Gateway. See [Base URLs](/guide/environments). |
| Auth | [API key](/guide/authentication#api-keys) |

## Parameters

| Parameter | In | Required | Type | Description |
| --- | --- | --- | --- | --- |
| `X-Transaction-Id` | Header | Yes | String | Your ID for this request. See [transaction IDs](/guide/authentication#transaction-ids). |
| `flowId` | Query | Yes | Long | The IVR flow. Dwesk gives you your flow IDs. |
| `type` | Query | Yes | String | Which calls to return. See [summary types](#summary-types). |
| `from` | Query | Yes | String | First day, `yyyy-MM-dd`. |
| `to` | Query | Yes | String | Last day, `yyyy-MM-dd`. This day is included. Up to 31 days after `from`. |
| `page` | Query | No | Integer | Page number, starting at `1`. Default `1`. |
| `size` | Query | No | Integer | Items per page. Default `50`, maximum `500`. |

Days are in Sri Lanka time and cover the whole day, from `00:00:00` to `23:59:59`.

### Summary types

| `type` | What you get | Item shape | Recordings |
| --- | --- | --- | --- |
| `ALL_DIAL_CALLS` | Every dial made from the flow: connect agent, queue, out-dial and direct mapping | [`IvrDialCall`](#ivrdialcall) | No |
| `OUT_DIAL_CALLS` | Calls your agents made out through the flow | [`IvrOutDialCall`](#ivroutdialcall) | Yes |
| `DIRECT_CALL_MAPPING` | Calls put straight through to a mapped number | [`IvrOutDialCall`](#ivroutdialcall) | Yes |
| `CONNECT_AGENT_CALLS` | Calls the flow put through to an agent | [`IvrAgentCall`](#ivragentcall) | Yes |
| `TRANSFER_CALLS` | Agent calls that were transferred | [`IvrAgentCall`](#ivragentcall) | Yes |
| `CONFERENCE_CALLS` | Agent calls that became a conference | [`IvrAgentCall`](#ivragentcall) | Yes |
| `IVR_UPLOAD_CALLS` | Calls made from numbers you uploaded to the flow | [`IvrUploadCall`](#ivruploadcall) | No |

Some types only work if the feature is turned on for your company. If it isn't, you get
`403 FEATURE_NOT_ENABLED`.

## Request

::: code-group

```bash [cURL]
curl 'https://gateway.dxesk.cloud/pabx/v1/ivr/call-summary?flowId=667&type=CONNECT_AGENT_CALLS&from=2026-10-01&to=2026-10-07&page=1&size=50' \
  -H 'X-API-Key: <YOUR_API_KEY>' \
  -H 'X-Transaction-Id: 3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64'
```

```ts [TypeScript]
import { randomUUID } from "node:crypto";

async function getIvrCallSummary(opts: {
  flowId: number;
  type: IvrSummaryType;
  from: string;
  to: string;
  page?: number;
  size?: number;
}) {
  const url = new URL(`${process.env.DWESK_GATEWAY_BASE}/ivr/call-summary`);
  for (const [k, v] of Object.entries(opts)) {
    if (v !== undefined) url.searchParams.set(k, String(v));
  }

  const res = await fetch(url, {
    headers: {
      "X-API-Key": process.env.DWESK_API_KEY!,
      "X-Transaction-Id": randomUUID(),
    },
  });
  if (!res.ok) throw new Error(`ivr/call-summary ${res.status}`);

  return (await res.json()) as IvrCallSummaryPage;
}
```

:::

## Success response

`200 OK`

```json
{
  "flowId": 667,
  "type": "CONNECT_AGENT_CALLS",
  "from": "2026-10-01",
  "to": "2026-10-07",
  "page": 1,
  "size": 50,
  "totalItems": 1,
  "totalPages": 1,
  "items": [
    {
      "id": 90211,
      "hotlineNumber": "0112345678",
      "agentCli": "779994374",
      "customerNumber": "0764897587",
      "status": "SUCCESS",
      "details": "Agent Ringing : 0779994374\nAgent Answered : 0779994374",
      "startTime": "2026-10-07T09:00:59+05:30",
      "endTime": "2026-10-07T09:04:13+05:30",
      "recordingUrl": "https://gateway.dxesk.cloud/pabx/v1/ivr/recordings/connect-agent/90211"
    }
  ]
}
```

| Field | Type | Description |
| --- | --- | --- |
| `flowId`, `type`, `from`, `to` | | Your request, sent back. |
| `page` | Integer | This page number. |
| `size` | Integer | Items per page. |
| `totalItems` | Integer | Items on all pages together. |
| `totalPages` | Integer | Number of pages. `0` if there are no items. |
| `items` | Array | The calls, newest first. The shape depends on `type`. |

Keep asking for the next page while `page` is less than `totalPages`.

## Item shapes

All times are Sri Lanka time, ISO 8601 with `+05:30`. A time is `null` if that step didn't
happen. `agentCli` is the agent's number without the first `0`, the same as in
[Call Records](/api/call-records#call-record). Other phone numbers are sent as Dwesk stored
them. If number masking is on for your company, some digits are hidden.

### IvrDialCall

For `ALL_DIAL_CALLS`.

| Field | Type | Description |
| --- | --- | --- |
| `id` | Long | Dwesk's ID for this dial. |
| `dialType` | String | What kind of dial it was. See [dial types](#dial-types). |
| `callerNumber` | String | The customer's number. |
| `hotlineNumber` | String \| null | The hotline the customer called. |
| `agentCli` | String \| null | The agent the call reached. `null` if it reached nobody. |
| `status` | String | `SUCCESS` if the call was connected, `FAILED` if it wasn't. Treat any other value as neither. |
| `finished` | Boolean | `false` while the call is still going on. |
| `dialStartTime` | String | When the flow started this dial. |
| `dialEndTime` | String \| null | When this dial last changed. Once `finished` is `true`, this is when it ended. |
| `callStartTime` | String \| null | When an agent was connected. |
| `callEndTime` | String \| null | When the connected call ended. |
| `ringCount` | Integer \| null | How many times agents were rung. |
| `agentsNotAnswered` | String[] | Agents who were rung but didn't answer, without the first `0`. Always empty when `status` is `SUCCESS`. |
| `voicemailAvailable` | Boolean | The caller left a voicemail. |
| `transferAvailable` | Boolean | The call was transferred. |
| `conferenceAvailable` | Boolean | The call became a conference. |
| `details` | String | Dwesk's step-by-step log of the call, one step per line. It is for people to read. Don't parse it, because the wording can change. |

#### Dial types

| `dialType` | Meaning |
| --- | --- |
| `CONNECT_AGENT` | Put through to an agent by Connect Agent. |
| `QUEUE_DIAL` | Placed in a queue to wait for a free agent. |
| `AGENT_OUT_DIAL` | An agent called out through the flow. |
| `DIRECT_CALL_MAPPING` | Put straight through to a mapped number. |
| `NORMAL_DIAL` | Any other dial. Also used when Dwesk didn't record a type. |

Other flow menu actions can show up too: `EXT_QUEUE_DIAL`, `EXT_AGENT_DIAL`, `AGENT_QUEUE`,
`CONTENT_DIAL`, `ACTIVATION_DIAL`, `DEACTIVATION_DIAL` and `COMMA_SEPERATED_DIAL`. If you
get a value that isn't listed here, keep it as it is.

### IvrOutDialCall

For `OUT_DIAL_CALLS` and `DIRECT_CALL_MAPPING`.

| Field | Type | Description |
| --- | --- | --- |
| `id` | Long | Dwesk's ID for this call. |
| `fromNumber` | String | Who made the call. For `OUT_DIAL_CALLS` this is the agent. For `DIRECT_CALL_MAPPING` it is the caller. |
| `toNumber` | String | The number that was called. |
| `status` | String | `SUCCESS` or `FAILED`. Treat any other value as neither. |
| `startTime` | String | When the call started. |
| `endTime` | String \| null | When the call ended. |
| `details` | String | Dwesk's log of the call. For people to read only. |
| `recordingUrl` | String \| null | See [Recordings](#recordings). |

### IvrAgentCall

For `CONNECT_AGENT_CALLS`, `TRANSFER_CALLS` and `CONFERENCE_CALLS`.

| Field | Type | Description |
| --- | --- | --- |
| `id` | Long | Dwesk's ID for this call. |
| `hotlineNumber` | String \| null | The hotline the customer called. |
| `agentCli` | String \| null | The agent. |
| `customerNumber` | String | The customer's number. |
| `status` | String | `SUCCESS` or `FAILED`. Treat any other value as neither. |
| `startTime` | String | When the call started. For transfers and conferences, when the transfer or conference started. |
| `endTime` | String \| null | When it ended. |
| `details` | String | Dwesk's log of the call. For people to read only. |
| `recordingUrl` | String \| null | See [Recordings](#recordings). For transfers and conferences this is the recording of the transfer or conference part. |

### IvrUploadCall

For `IVR_UPLOAD_CALLS`.

| Field | Type | Description |
| --- | --- | --- |
| `campaignId` | String | The upload the call came from. |
| `callId` | Integer | Dwesk's ID for this call in the upload. |
| `fromNumber` | String | The number the call was made from. |
| `toNumber` | String | The number that was called. |
| `scheduledTime` | String \| null | When the call was planned for. |
| `executedTime` | String \| null | When Dwesk picked it up to dial. |
| `startTime` | String \| null | When the call started. |
| `endTime` | String \| null | When the call ended. |
| `result` | String \| null | What happened to the call. |

## Recordings

`recordingUrl` is set when `status` is `SUCCESS` and Dwesk has a recording for the call.
Otherwise it is `null`. Recordings must also be turned on for your company. If they aren't,
`recordingUrl` is always `null`.

Download it with [IVR Call Recording](/api/ivr-recordings), using the same API key. Use the
URL exactly as you got it.

## Error responses

The body has a `code`, a `message` and your `transactionId`.

| Status | `code` | Why |
| --- | --- | --- |
| `400` | `INVALID_PARAMETER` | A parameter is missing or wrong. `message` says which one. |
| `400` | `INVALID_RANGE` | `from` or `to` isn't `yyyy-MM-dd`, `to` is before `from`, or the range is more than 31 days. |
| `400` | `MISSING_TRANSACTION_ID` | The `X-Transaction-Id` header is missing or empty. |
| `400` | `INVALID_TRANSACTION_ID` | The transaction ID is longer than 64 characters or has characters that aren't allowed. |
| `401` | `UNAUTHORIZED` | The `X-API-Key` header is missing or the key is wrong. |
| `403` | `FEATURE_NOT_ENABLED` | This `type` isn't turned on for your company. |
| `404` | `FLOW_NOT_FOUND` | Your company has no flow with this `flowId`. |
| `429` | `RATE_LIMITED` | You sent more than 60 requests in a minute. Wait `Retry-After` seconds. |
| `500`, `503` | `SERVER_ERROR` | Try again later. |

```json
{ "code": "INVALID_PARAMETER", "message": "type must be one of ALL_DIAL_CALLS, OUT_DIAL_CALLS, DIRECT_CALL_MAPPING, CONNECT_AGENT_CALLS, TRANSFER_CALLS, CONFERENCE_CALLS, IVR_UPLOAD_CALLS", "transactionId": "3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64" }
```

A page past the last one is not an error. You get `200` with an empty `items`.
