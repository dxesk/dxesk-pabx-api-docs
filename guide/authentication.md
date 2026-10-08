# Authentication

The middleware and web endpoints use HTTP Basic authentication. The call records endpoints
run on the Dwesk API gateway and use an [API key](#api-keys) instead. There are no tokens to
refresh and no OAuth.

## Building the header

Base64-encode `username:password` and send it in the `Authorization` header:

::: code-group

```bash [cURL]
curl -X POST 'https://dxesk-asr-node7-server-farm-6.dxesk.cloud/dwesk-middleware/api/middleware/api/outbound-call' \
  -H 'Authorization: Basic <BASE64_USER_COLON_PASS>' \
  -d 'toNumber=0771234567'
```

```ts [TypeScript]
const token = Buffer.from(`${username}:${password}`).toString("base64");

const res = await fetch(url, {
  method: "POST",
  headers: { Authorization: `Basic ${token}` },
});
```

```ts [TypeScript (browser / edge)]
const token = btoa(`${username}:${password}`);

const res = await fetch(url, {
  method: "POST",
  headers: { Authorization: `Basic ${token}` },
});
```

:::

Most HTTP clients build the header for you. In cURL, `-u username:password` does the same
thing as the header above.

::: danger Never authenticate from a browser
Basic credentials are reversible. Anyone who sees the header can Base64-decode it back to
your password. Call the Dwesk API only from a server you control, and keep credentials in
environment variables or a secret manager, never in frontend code, a committed `.env`, or
a shared Postman collection.
:::

## Getting credentials

The Dwesk team issues credentials, `companyId`, `serviceId`, and `agentId` per tenant when
your account is provisioned, along with the hostnames your tenant is served from. See
[Base URLs](/guide/environments).

Ask your Dwesk account contact. You will receive:

| Value | Used by |
| --- | --- |
| Username and password | Every endpoint |
| `companyId` | Outbound call, queue upload, direct mapping, recordings |
| `serviceId` | Outbound call, content upload |
| `agentId` | Content upload |
| `queueId` | Queue upload |
| `flowId` | Recording export, IVR call summary, IVR reports |
| Hotline number | Direct mapping |
| Middleware and web hostnames | Every endpoint |

## Failure response

A bad or missing header returns HTTP `401`, with code `001` on the outbound-call endpoint
and `007` on queue upload:

```json
{
  "status": "001",
  "message": "Unauthorized"
}
```

If you get a `401` with credentials you believe are correct, check that you are calling
the hostname issued to your tenant. See [Base URLs](/guide/environments).

## Webhook authentication

Webhooks travel in the opposite direction, so your endpoint authenticates Dwesk rather
than the reverse. You give the Dwesk team a webhook URL and, optionally, an API key. They
store both against your company, and Dwesk sends the key back on each delivery as an
`x-api-key` header. [Webhooks Overview](/webhooks/) covers verification.

## API keys

[Call Records](/api/call-records), [Call Record Counts](/api/call-record-counts),
[IVR Call Summary](/api/ivr-call-summary), [IVR Call Recording](/api/ivr-recordings) and
[IVR Reports](/api/ivr-reports) run on the Dwesk API gateway. Send your API key in the
`X-API-Key` header, and a [transaction ID](#transaction-ids) in the `X-Transaction-Id` header:

```bash
curl 'https://gateway.dxesk.cloud/pabx/v1/call-records' \
  -H 'X-API-Key: <YOUR_API_KEY>' \
  -H 'X-Transaction-Id: 3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64'
```

Ask your Dwesk contact for an API key. With it you get your gateway base URL and the signing
secret for the [Call Records Feed](/webhooks/call-records-feed). Each key belongs to one
company and only returns that company's data.

Keep the key safe, the same way as a password. Only use it from your own server, and keep it
in a secret manager or an environment variable. If it leaks, ask your Dwesk contact to cancel
it and give you a new one.

If the key is missing or wrong, you get HTTP `401`:

```json
{ "code": "UNAUTHORIZED", "message": "Missing or invalid API key", "transactionId": "3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64" }
```

## Transaction IDs

Every request you send to the gateway must have a transaction ID in the `X-Transaction-Id`
header. It is your own reference for that request. If something goes wrong, give us the
transaction ID and we can find exactly that request in our logs.

| Rule | Detail |
| --- | --- |
| Format | Any text or number, up to 64 characters. Letters, digits, `-`, `_`, `.` and `:` only. |
| Recommended | A UUID, for example from `crypto.randomUUID()`. |
| Unique | Use a new one for every request, including when you try a request again. |
| Keep it | Log it on your side together with what you sent. |

We send the same value back in the `X-Transaction-Id` response header, and in the
`transactionId` field of every error body. Error responses also tell you when it is missing or
wrong:

| Status | `code` | Why |
| --- | --- | --- |
| `400` | `MISSING_TRANSACTION_ID` | There is no `X-Transaction-Id` header, or it is empty. |
| `400` | `INVALID_TRANSACTION_ID` | It is longer than 64 characters or has other characters. |

```ts
import { randomUUID } from "node:crypto";

const res = await fetch(url, {
  headers: {
    "X-API-Key": process.env.DWESK_API_KEY!,
    "X-Transaction-Id": randomUUID(),
  },
});
```

When you contact us about a problem, send the transaction ID, the time of the request and the
endpoint you called.

