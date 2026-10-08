# Call Records Feed

Every 5 minutes, Dwesk sends the new and changed call records to an endpoint on your server.
This keeps your records up to date after the first download.

| | |
| --- | --- |
| Direction | Dwesk → your system |
| Method | `POST` |
| Content-Type | `application/json` |
| How often | Every 5 minutes, even when nothing changed |
| Signed with | HMAC-SHA256, using a secret Dwesk gives you |

Read [Syncing Call Records](/guide/call-records-sync) first. The feed works together with
the [Call Records](/api/call-records) endpoint, which you use to fill in anything you miss.

## Getting set up

Like the other [webhooks](/webhooks/), you can't set this up yourself. Send the Dwesk team the
HTTPS URL where you want to receive the feed. We save it for your company with a signing
secret and a retry count, and send you the secret.

## Payload

```json
{
  "companyId": 1005,
  "from": "2026-10-08T09:03:00+05:30",
  "to": "2026-10-08T09:10:00+05:30",
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
  "complete": true,
  "cursor": "eyJwIjoiYyIsInQiOiIyMDI2LTEwLTA4VDA5OjEwOjAwIiwiaSI6MH0"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `companyId` | Long | Your company ID. |
| `from` | String | Start of the time period this delivery covers. |
| `to` | String | End of the time period. |
| `records` | Array | Records whose `updatedAt` is inside the period, oldest first. At most 1000. The fields are listed on [Call Records](/api/call-records#call-record). |
| `complete` | Boolean | `false` if more than 1000 records changed in the period, so some were left out. |
| `cursor` | String | A [Call Records](/api/call-records) cursor that points just after the last record in this delivery. |

Each period starts 2 minutes before the last one ended. So a record that changed near the
end of a period can come in two deliveries.

### When complete is false

If more than 1000 records changed in 5 minutes, Dwesk sends the first 1000 and sets
`complete` to `false`. Get the rest by calling [Call Records](/api/call-records) with this
delivery's `cursor` until `hasMore` is `false`. You can use that cursor straight away.

## Finding a missed delivery

Each time you finish a delivery, save its `to` and `cursor`. When the next one arrives:

- if its `from` is the same as or earlier than your saved `to`, you missed nothing;
- if its `from` is later than your saved `to`, you missed one or more deliveries. Call
  [Call Records](/api/call-records) with your saved `cursor` until `hasMore` is `false`, then
  save this delivery.

The full code is in [If you missed something](/guide/call-records-sync#if-you-missed-something).

Dwesk doesn't send a separate message when we had a problem or stopped retrying. The `from`
of the next delivery tells you.

## Checking the signature

Every delivery has two headers:

| Header | Value |
| --- | --- |
| `X-Dwesk-Timestamp` | The time the delivery was signed, in Unix seconds. |
| `X-Dwesk-Signature` | `sha256=` and then the hex HMAC-SHA256 of `<timestamp>.<raw body>`, made with your secret. |

Calculate the signature from the raw request body, before you parse the JSON, and compare it
in constant time. Reject the request if the timestamp is more than 5 minutes away from your
own clock. That stops someone from sending an old request again.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

function verify(rawBody: Buffer, timestamp: string, signature: string): boolean {
  const age = Math.abs(Date.now() / 1000 - Number(timestamp));
  if (!Number.isFinite(age) || age > 300) return false;

  const expected =
    "sha256=" +
    createHmac("sha256", process.env.DWESK_FEED_SECRET!)
      .update(`${timestamp}.`)
      .update(rawBody)
      .digest("hex");

  const a = Buffer.from(expected);
  const b = Buffer.from(signature);
  return a.length === b.length && timingSafeEqual(a, b);
}
```

## Handling a delivery

Reply quickly, then do the work afterwards:

```ts
app.post(
  "/webhooks/dwesk/call-records",
  express.raw({ type: "application/json" }),
  async (req, res) => {
    const ok = verify(
      req.body,
      req.header("X-Dwesk-Timestamp") ?? "",
      req.header("X-Dwesk-Signature") ?? "",
    );
    if (!ok) return res.sendStatus(401);

    res.sendStatus(200);

    await syncQueue.add(JSON.parse(req.body.toString("utf8")));
  },
);
```

Handle deliveries one at a time, in the order they arrive. If you handle two at once, both
can pass the missed-delivery check and save their `to` in the wrong order.

## Reply and retries

Reply with any `2xx` within 10 seconds. We don't read the body.

If you take longer, or reply with any other status, we try again. The number of retries is
set for your company, up to 3. They happen 10, 30 and 60 seconds after the failure, so they
are done before the next delivery. If the last retry also fails, we drop that delivery and
carry on with the next one. The missed-delivery check above is how you notice and fill the
gap.

::: warning Empty deliveries matter too
We send a delivery every 5 minutes even if nothing changed. Then `records` is `[]`. Save its
`to` and `cursor` like any other delivery. If you ignore empty deliveries, the next one will
look like you missed something, and you'll make a call you didn't need.
:::
