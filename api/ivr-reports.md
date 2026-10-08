# IVR Reports

Get an IVR report as a CSV file. These are the same reports as the IVR Flow Reports page in
the Dwesk portal. The portal builds an Excel file and you download it later. This endpoint
sends you the CSV straight away.

<div class="endpoint"><span class="method get">GET</span><span class="path">/ivr/reports</span></div>

| | |
| --- | --- |
| Base | Gateway. See [Base URLs](/guide/environments). |
| Auth | [API key](/guide/authentication#api-keys) |
| Returns | `text/csv` |

## Parameters

| Parameter | In | Required | Type | Description |
| --- | --- | --- | --- | --- |
| `X-Transaction-Id` | Header | Yes | String | Your ID for this request. See [transaction IDs](/guide/authentication#transaction-ids). |
| `report` | Query | Yes | String | `FLOW_REPORT`, `AGENT_CALLS` or `AGENT_CALL_SUMMARY`. See [reports](#reports). |
| `from` | Query | Yes | String | First day, `yyyy-MM-dd`. |
| `to` | Query | Yes | String | Last day, `yyyy-MM-dd`. This day is included. Up to 31 days after `from`. |
| `flowId` | Query | For `FLOW_REPORT` | Long | The IVR flow. |
| `type` | Query | No | String | For `FLOW_REPORT` only. `ALL_DIAL_CALLS` (default), `CONNECT_AGENT_CALLS` or `IVR_UPLOAD_CALLS`. |
| `filter` | Query | No | String | For `FLOW_REPORT` with `type=ALL_DIAL_CALLS` only. Default `ALL_DIALS`. See [filters](#filters). |
| `missedCallWindow` | Query | No | Integer | For `filter=CONNECT_AGENT_MISSED_CALLS` only. Minutes, `0` to `1440`. Default `0`. See [missed calls](#missed-calls). |
| `agentCli` | Query | For `AGENT_CALL_SUMMARY` | String | The agent's number, with or without the first `0`. Optional for `AGENT_CALLS`, where leaving it out gives you all agents. |
| `direction` | Query | No | String | For `AGENT_CALL_SUMMARY` only. `ALL` (default), `INCOMING` or `OUTGOING`. |

Days are in Sri Lanka time and cover the whole day.

### Reports

| `report` | What it covers | Columns |
| --- | --- | --- |
| `FLOW_REPORT` | Calls through one IVR flow. `type` and `filter` pick which ones. | Depend on `type` and `filter`, see below |
| `AGENT_CALLS` | Calls handled by one agent, or all agents | [Agent calls](#agent-calls) |
| `AGENT_CALL_SUMMARY` | One agent's calls, with wait, ring and talk times. Only if this report is turned on for your company. | [Agent call summary](#agent-call-summary) |

### Filters

For `FLOW_REPORT` with `type=ALL_DIAL_CALLS`. Some filters only work if the feature is turned
on for your company.

| `filter` | Rows |
| --- | --- |
| `ALL_DIALS` | Every dial |
| `CONNECT_AGENT_ALL_CALLS` | Dials with `dial_type` `CONNECT_AGENT` |
| `CONNECT_AGENT_SUCCESS_CALLS` | Same, with `status` `SUCCESS` |
| `CONNECT_AGENT_FAILED_CALLS` | Same, with `status` `FAILED` |
| `CONNECT_AGENT_MISSED_CALLS` | Missed callers, one row per missed call. Different columns, see [missed calls](#missed-calls). |
| `QUEUE_DIAL_ALL` | Dials with `dial_type` `QUEUE_DIAL` |
| `QUEUE_DIAL_SUCCESS` | Same, with `status` `SUCCESS` |
| `QUEUE_DIAL_FAILED` | Same, with `status` `FAILED` |
| `OUT_CALL_ALL` | Dials with `dial_type` `AGENT_OUT_DIAL` |
| `OUT_CALL_SUCCESS` | Same, with `status` `SUCCESS` |
| `OUT_CALL_FAILED` | Same, with `status` `FAILED` |
| `DIRECT_CALL_MAPPING` | Dials with `dial_type` `DIRECT_CALL_MAPPING` |
| `VOICE_MAIL` | Dials where the caller left a voicemail |
| `TRANSFER` | Dials that were transferred |
| `CONFERENCE` | Dials that became a conference |

## Request

::: code-group

```bash [cURL]
curl -o flow-667.csv 'https://gateway.dxesk.cloud/pabx/v1/ivr/reports?report=FLOW_REPORT&flowId=667&type=ALL_DIAL_CALLS&filter=CONNECT_AGENT_SUCCESS_CALLS&from=2026-10-01&to=2026-10-07' \
  -H 'X-API-Key: <YOUR_API_KEY>' \
  -H 'X-Transaction-Id: 3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64'
```

```ts [TypeScript]
import { randomUUID } from "node:crypto";
import { writeFile } from "node:fs/promises";

async function getIvrReport(params: Record<string, string>, file: string) {
  const url = new URL(`${process.env.DWESK_GATEWAY_BASE}/ivr/reports`);
  for (const [k, v] of Object.entries(params)) url.searchParams.set(k, v);

  const res = await fetch(url, {
    headers: {
      "X-API-Key": process.env.DWESK_API_KEY!,
      "X-Transaction-Id": randomUUID(),
    },
  });
  if (!res.ok) {
    const err = (await res.json()) as GatewayError;
    throw new Error(`${res.status} ${err.code}: ${err.message}`);
  }

  await writeFile(file, await res.text());
}

await getIvrReport(
  { report: "FLOW_REPORT", flowId: "667", from: "2026-10-01", to: "2026-10-07" },
  "./flow-667.csv",
);
```

:::

## Success response

`200 OK`

| Header | Value |
| --- | --- |
| `Content-Type` | `text/csv; charset=utf-8` |
| `Content-Disposition` | `attachment; filename="<report>_<from>_<to>.csv"` |

```csv
call_id,hotline_number,dial_type,caller_number,agent_cli,dial_start_time,dial_end_time,dial_duration_seconds,dial_duration_units,call_start_time,call_end_time,call_duration_seconds,call_duration_units,status,agents_not_answered,voicemail_available,transfer_available,conference_available,details
88120,0112345678,CONNECT_AGENT,0764897587,779994374,2026-10-07 09:00:59,2026-10-07 09:04:13,194,4,2026-10-07 09:01:09,2026-10-07 09:04:13,184,4,SUCCESS,,NO,NO,NO,"Agent Ringing : 0779994374
Agent Answered : 0779994374"
```

The file is a standard CSV:

- The first line is the column names. Every other line is one row. There are no title or
  total lines.
- Text with a comma, a quote or a line break is inside double quotes. A quote inside the text
  is written twice (`""`). Use a CSV library to read it. `details` often has line breaks.
- Times are Sri Lanka time, written `yyyy-MM-dd HH:mm:ss`. An empty value means that step
  didn't happen.
- `YES` and `NO` are used for yes or no columns.
- `agent_cli` is the agent's number without the first `0`. Lists of agents are separated by `;`.
- Rows are newest first.

If nothing matches, you get `200` with only the column names line.

## Columns

### Flow report: all dial calls

For `type=ALL_DIAL_CALLS` with any filter except `CONNECT_AGENT_MISSED_CALLS`.

| Column | Description |
| --- | --- |
| `call_id` | Dwesk's ID for this dial. Same as `id` in [`IvrDialCall`](/api/ivr-call-summary#ivrdialcall). |
| `hotline_number` | The hotline the customer called. |
| `dial_type` | What kind of dial it was. See [dial types](/api/ivr-call-summary#dial-types). |
| `caller_number` | The customer's number. |
| `agent_cli` | The agent the call reached. |
| `dial_start_time`, `dial_end_time` | When the dial started and ended. |
| `dial_duration_seconds` | `dial_end_time` minus `dial_start_time`, in whole seconds. |
| `dial_duration_units` | Billing units for the dial. See [duration units](#duration-units). |
| `call_start_time`, `call_end_time` | When the agent was connected and when that call ended. |
| `call_duration_seconds` | `call_end_time` minus `call_start_time`, in whole seconds. |
| `call_duration_units` | Billing units for the connected call. |
| `status` | `SUCCESS` or `FAILED`. |
| `agents_not_answered` | Agents who were rung but didn't answer. Empty when `status` is `SUCCESS`. |
| `voicemail_available` | `YES` if the caller left a voicemail. |
| `transfer_available` | `YES` if the call was transferred. |
| `conference_available` | `YES` if the call became a conference. |
| `details` | Dwesk's log of the call. For people to read only. |

#### Duration units

| Duration | Units |
| --- | --- |
| No duration, or a missing time | `0` |
| Less than 60 seconds | `1` |
| 60 seconds or more | Whole minutes + 1. So 60 to 119 seconds is `2`, 120 to 179 is `3`. |

This is how Dwesk counts it today: a call of exactly 60 seconds is 2 units.

### Missed calls

For `type=ALL_DIAL_CALLS&filter=CONNECT_AGENT_MISSED_CALLS`.

| Column | Description |
| --- | --- |
| `call_id` | ID of the last attempt in this missed call. |
| `hotline_number` | The hotline the customer called. |
| `caller_number` | The customer's number. |
| `attempts` | How many times they called in this missed call. |
| `first_attempt_time` | Start of the first attempt. |
| `last_attempt_time` | Start of the last attempt. |
| `waiting_seconds` | Time spent on all the attempts together, in whole seconds. |
| `agents_not_answered` | Every agent who was rung and didn't answer, across all attempts. |
| `voicemail_left` | `YES` if a voicemail was left on any attempt. |
| `details` | Dwesk's log of the last attempt. |

This is how Dwesk decides what a missed call is. Only `CONNECT_AGENT` dials are looked at.

With `missedCallWindow=0`, each `FAILED` dial where an agent was rung is one missed call, with
1 attempt.

With `missedCallWindow` above 0, a caller's attempts are grouped:

1. Take the caller's dials in time order. A dial joins the group of the dial before it if it
   started within `missedCallWindow` minutes of that dial.
2. A `SUCCESS` dial means the caller got through. That dial and the ones before it in the
   group are not missed. Dials after it start a new group.
3. A group is one missed call if its last dial is `FAILED` and an agent was rung on at least
   one of its dials.

A missed call is listed on the day of its last attempt. The report looks one day past `to`, so
a caller who got through just after midnight isn't counted as missed.

The portal's Excel file also shows the number of missed calls per agent. To get it, count how
many rows each agent appears in under `agents_not_answered`.

### Flow report: connect agent calls

For `type=CONNECT_AGENT_CALLS`.

| Column | Description |
| --- | --- |
| `call_id` | Same as `id` in [`IvrAgentCall`](/api/ivr-call-summary#ivragentcall). |
| `hotline_number` | The hotline the customer called. |
| `agent_cli` | The agent. |
| `customer_number` | The customer's number. |
| `start_time`, `end_time` | When the call started and ended. |
| `status` | `SUCCESS` or `FAILED`. |
| `details` | Dwesk's log of the call. |

### Flow report: IVR upload calls

For `type=IVR_UPLOAD_CALLS`.

| Column | Description |
| --- | --- |
| `campaign_id` | The upload the call came from. |
| `call_id` | Dwesk's ID for this call in the upload. |
| `from_number`, `to_number` | Who called whom. |
| `scheduled_time` | When the call was planned for. |
| `executed_time` | When Dwesk picked it up to dial. |
| `start_time`, `end_time` | When the call started and ended. |
| `result` | What happened to the call. |

### Agent calls

For `report=AGENT_CALLS`.

| Column | Description |
| --- | --- |
| `call_id` | Dwesk's ID for this call. |
| `agent_cli` | The agent. |
| `customer_number` | The customer's number. |
| `call_type` | What kind of dial it was. Same values as [dial types](/api/ivr-call-summary#dial-types). |
| `flow_or_queue_id` | The flow or queue the call came through. |
| `start_time`, `end_time` | When the call started and ended. |
| `duration_seconds` | `end_time` minus `start_time`, in whole seconds. |
| `status` | `SUCCESS` or `FAILED`. |
| `details` | Dwesk's log of the call. |

The portal's Excel file has three total lines at the top. You can work them out from the rows:

| Total | How |
| --- | --- |
| Total | Number of rows, and the sum of `duration_seconds` |
| Success | Rows with `status` `SUCCESS`, and their `duration_seconds` |
| Failed | All other rows, and their `duration_seconds` |

### Agent call summary

For `report=AGENT_CALL_SUMMARY`. These rows are [call records](/api/call-records), so the
times and statuses mean the same thing.

| Column | Description |
| --- | --- |
| `call_id` | Same as `id` in a call record. |
| `call_type` | `CONNECT_DIAL` or `OUTGOING`. |
| `call_status` | Same values as `callStatus` in a call record. |
| `caller_number` | Who called. For `OUTGOING` this is the agent. |
| `agent_cli` | The agent who took the call. Can be empty on `OUTGOING`. |
| `call_start_time` | When the call reached the PBX. |
| `ring_start_time` | When the agent's phone started ringing. |
| `connect_time` | When the agent answered. |
| `call_end_time` | When the call ended. |
| `wait_seconds` | `ring_start_time` minus `call_start_time`. `0` if either is empty. |
| `ring_seconds` | `connect_time` minus `ring_start_time`. If the agent didn't answer, `call_end_time` minus `ring_start_time`. `0` if there is no ring time. |
| `talk_seconds` | `call_end_time` minus `connect_time`, only when `call_status` is `ANSWERED`. Otherwise `0`. |
| `total_handling_seconds` | `wait_seconds` + `ring_seconds` + `talk_seconds`. |

All durations are whole seconds. A negative result is written as `0`.

`direction` picks the rows:

| `direction` | Rows |
| --- | --- |
| `INCOMING` | `CONNECT_DIAL` calls where `agent_cli` is the agent |
| `OUTGOING` | `OUTGOING` calls where `caller_number` is the agent |
| `ALL` | Both |

Queue calls (`QUEUE_DIAL`) are not in this report.

The portal's Excel file has a summary at the top. You can work it out from the rows:

| Summary | How |
| --- | --- |
| Total calls | Number of rows |
| Answered calls | Rows with `call_status` `ANSWERED` |
| Failed calls | Rows with `call_status` `FAILED` |
| Answer rate (%) | Answered ÷ total × 100, 2 decimals. `0` if there are no rows. |
| Average ring time | Sum of `ring_seconds` ÷ total calls, 2 decimals |
| Average wait time | Sum of `wait_seconds` ÷ total calls, 2 decimals |
| Average talk time | Sum of `talk_seconds` ÷ answered calls, 2 decimals. `0` if none were answered. |
| Total talk time | Sum of `talk_seconds` |

`TRANSFER` and `CONFERENCE` calls count in the total but not as answered or failed. This is
different from the [Dashboard Metrics](/reference/dashboard-metrics), where they count as
answered.

## Error responses

Errors come back as JSON, not CSV. The body has a `code`, a `message` and your `transactionId`.

| Status | `code` | Why |
| --- | --- | --- |
| `400` | `INVALID_PARAMETER` | A parameter is missing, wrong, or doesn't go with this `report`. `message` says which one. |
| `400` | `INVALID_RANGE` | `from` or `to` isn't `yyyy-MM-dd`, `to` is before `from`, or the range is more than 31 days. |
| `400` | `MISSING_TRANSACTION_ID` | The `X-Transaction-Id` header is missing or empty. |
| `400` | `INVALID_TRANSACTION_ID` | The transaction ID is longer than 64 characters or has characters that aren't allowed. |
| `401` | `UNAUTHORIZED` | The `X-API-Key` header is missing or the key is wrong. |
| `403` | `FEATURE_NOT_ENABLED` | This report, `type` or `filter` isn't turned on for your company. |
| `404` | `FLOW_NOT_FOUND` | Your company has no flow with this `flowId`. |
| `404` | `AGENT_NOT_FOUND` | Your company has no agent with this `agentCli`. |
| `429` | `RATE_LIMITED` | You sent more than 60 requests in a minute. Wait `Retry-After` seconds. |
| `500`, `503` | `SERVER_ERROR` | Try again later. |

```json
{ "code": "INVALID_PARAMETER", "message": "flowId is required when report is FLOW_REPORT", "transactionId": "3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64" }
```

::: tip Large reports
A busy flow can have many rows in 31 days. If a request is slow or times out, ask for fewer
days at a time and join the files.
:::
