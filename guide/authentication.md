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
| `flowId` | Recording export |
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

[Call Records](/api/call-records) and [Call Record Counts](/api/call-record-counts) run on
the Dwesk API gateway. Send your API key in the `X-API-Key` header:

```bash
curl 'https://gateway.dxesk.cloud/pabx/v1/call-records' \
  -H 'X-API-Key: <YOUR_API_KEY>'
```

We give you the API key once the integration is agreed. You also get the gateway address and
the signing secret for the [Call Records Feed](/webhooks/call-records-feed). Each key belongs
to one company and only returns that company's records.

Keep the key safe, the same way as a password. Only use it from your own server, and keep it
in a secret manager or an environment variable. If it leaks, ask your Dwesk contact to cancel
it and give you a new one.

If the key is missing or wrong, you get HTTP `401`:

```json
{ "code": "UNAUTHORIZED", "message": "Missing or invalid API key" }
```
