# Error Codes

::: danger Codes are scoped to their endpoint
The same numeric code means different things on different endpoints, and on some endpoints
it means two different things depending on the message. `006` is a reused transaction ID on
the outbound call and a past start time on queue upload. Do not write a shared error-code
map. Branch on which endpoint returned the code, then read the `message`.
:::

## HTTP status is always 200

Every middleware endpoint on this site returns a plain response body with HTTP `200`,
including on failure. Nothing signals an error through the status line. Read `status` from
the body.

```ts
const res = await fetch(url, opts);
const json = await res.json();

// res.ok is true here even when json.status is "011"
```

## Success conventions

Three endpoint families use three different conventions:

| Endpoints | Success looks like |
| --- | --- |
| Outbound call, queue upload | `"status": "001"` |
| Content upload | `"status": 200` (number), message in `msg` |
| Callback routing | `"status": 0` (number) |
| Recording export | `"status": "SUCCESS"`, with `"statusCode": "0000"` |

```ts
function isSuccess(json: { status: string | number }): boolean {
  return (
    json.status === "001" ||
    json.status === 200 ||
    json.status === 0 ||
    json.status === "SUCCESS"
  );
}
```

## POST /api/outbound-call

| Code | Message | Cause |
| --- | --- | --- |
| `001` | Unauthorized | Basic auth failed, or the username is not a known PBX agent. |
| `002` | Missing parameter | On `CAMPAIGN`, one of `companyId`, `transactionId`, `startTime`, `contentId`, `toNumber` is absent. |
| `003` | Start time is in the past | `startTime` is before current server time. |
| `004` | company not found | Invalid `companyId`. |
| `005` | Invalid serviceId | `serviceId` is not registered against this company. |
| `006` | Transaction Id unavailable | This `transactionId` has been used before. |
| `007` | company hotline number not found | No hotline configured for the company. |
| `008` | Invalid response from external API | The downstream scheduling call failed. |
| `008` | Internal server error | Unhandled failure. |
| `009` | Content not found | The DRM lookup returned nothing. |
| `009` | Invalid content ID | A referenced content ID exists but is not usable. |
| `011` | Invalid toNumber | Does not match `0` followed by 9 digits. |
| `012` | Missing dtmfOptions for SURVEY | Required for `SURVEY`. |
| `013` | Missing contentMapping for SURVEY | Required for `SURVEY`. |
| `014` | Missing contentMapping for key | A DTMF or reserved key has no mapping. |
| `015` | Duplicate DTMF keys not allowed | A key appears twice in `dtmfOptions`. |
| `018` | File mapping validation failed | `contentMapping` is not in `key:contentId` form. |
| `018` | Internal server error | Unhandled failure in the scheduling service. |

## POST /api/queue/start-upload

| Code | Message | Cause |
| --- | --- | --- |
| `005` | Missing parameter | A required parameter was omitted. |
| `006` | Start time is in the past | `startTime` is before current server time. |
| `007` | Unauthorized | Basic auth failed. |
| `008` | Invalid queue or company | `companyId` or `queueId` not found. |
| `009` | CSV contains invalid number: `<entry>` | A row failed validation. |
| `400` | Unhandled Error | Unhandled failure during processing. |
| `500` | Failed to parse CSV | The file could not be read as CSV. |
| `500` | Check Lead Ids Again | No usable lead IDs were found. |

`400` and `500` here are values of the `status` string, not HTTP statuses.

## POST /api/content/v1/add

| status | msg | Cause |
| --- | --- | --- |
| `2` | Internal Server Error! | Unhandled failure. |
| `3` | Session Not Found! | The request was not authenticated. |
| `4` | Empty File! | No `attachment` was sent. |
| `5` | Empty Service ID! | No `serviceId` was sent. |
| `6` | Service Not Found! | `serviceId` does not belong to the agent's company. |
| `7` | (relayed from DRM) | DRM rejected the upload with a 401. |

## Direct agent mapping

`status: 1` is a rejected request. `status: 3` is a server-side failure.

| Status | Message |
| --- | --- |
| 1 | Invalid Company ID |
| 1 | Invalid Hotline Number or Hotline does not belong to this Company |
| 1 | Invalid Agent Number. Must be 9 or 10 digits. |
| 1 | Invalid Customer Number. Must be 9 or 10 digits or 'ANY'. |
| 1 | Agent Number not found in PBX records for this Company. |
| 1 | A mapping already exists for this Agent and Customer within the requested time period. |
| 1 | Transaction ID not found |
| 1 | Mapping is already inactive |
| 3 | Failed to create mapping |
| 3 | Internal Server Error |
| 3 | Invalid Request |

## GET /download/call-recordings

| statusCode | Cause |
| --- | --- |
| `0000` | Success. |
| `0008` | Missing, non-Basic, or malformed `Authorization`, or the username is not a known PBX agent. |
| `0009` | Unhandled failure while building the export. |

## Gateway endpoints

The gateway endpoints ([Call Records](/api/call-records),
[Call Record Counts](/api/call-record-counts), [IVR Call Summary](/api/ivr-call-summary),
[IVR Call Recording](/api/ivr-recordings) and [IVR Reports](/api/ivr-reports)) work
differently from the endpoints above. They use normal HTTP status codes, and an error body
looks like this. The recording and report endpoints also send errors as JSON, not as audio or
CSV.

```json
{ "code": "INVALID_CURSOR", "message": "Cursor is not valid for this company", "transactionId": "3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64" }
```

| Status | `code` | Cause |
| --- | --- | --- |
| `400` | `INVALID_CURSOR` | The cursor was changed, cut off, or belongs to another company. |
| `400` | `INVALID_LIMIT` | `limit` is less than 1 or more than 1000. |
| `400` | `INVALID_RANGE` | Wrong `from` or `to`, or a range of more than 31 days. |
| `400` | `INVALID_PARAMETER` | A parameter is missing or wrong. `message` says which one. |
| `400` | `MISSING_TRANSACTION_ID` | `X-Transaction-Id` is missing or empty. |
| `400` | `INVALID_TRANSACTION_ID` | `X-Transaction-Id` is longer than 64 characters or has characters that aren't allowed. |
| `401` | `UNAUTHORIZED` | `X-API-Key` is missing or wrong. |
| `403` | `FEATURE_NOT_ENABLED` | The summary type, report, filter or recordings aren't turned on for your company. |
| `404` | `FLOW_NOT_FOUND` | Your company has no flow with this `flowId`. |
| `404` | `AGENT_NOT_FOUND` | Your company has no agent with this `agentCli`. |
| `404` | `RECORDING_NOT_FOUND` | No call with this ID for your company, or it has no recording. |
| `429` | `TOO_EARLY` | You already have everything and called before `nextPollAfter`. |
| `429` | `RATE_LIMITED` | More than 60 requests in a minute. |
| `500`, `503` | `SERVER_ERROR` | Try again later with the same parameters. |

Both `429` responses include a `Retry-After` header, in seconds.

`transactionId` is the `X-Transaction-Id` you sent, or `null` if you didn't send one. Give it
to us when you report a problem. See [Transaction IDs](/guide/authentication#transaction-ids).

## Webhook receiver errors

These are codes your own system returns, not Dwesk's. Dwesk retries `400`, `500`, and
`503`, using the retry settings configured for your company. See [Webhooks](/webhooks/).

`NS-00115` is a validation error from a receiver that requires conditional fields
unconditionally. See
[Conditional Validation](/webhooks/connect-agent-call-end#conditional-validation).

## Common causes

A `401`, or `status` `001` or `007` with credentials you believe are correct, usually means
you are calling a hostname other than the one issued to your tenant. See
[Base URLs](/guide/environments).

A `404` on content upload means you used the middleware host. Content upload is served by
the web host.

An `011` means `toNumber` did not match `0` followed by exactly 9 digits. A 9-digit number
without the leading zero fails this check.

Start-time rejections on a freshly built timestamp happen because the server compares
against its own local time, not UTC and not yours. Schedule at least a minute ahead.

A `005` on a request that includes every field is usually capitalisation. The queue upload
file field is `File`, not `file`.
