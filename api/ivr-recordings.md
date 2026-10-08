# IVR Call Recording

Download the recording of one call from the [IVR Call Summary](/api/ivr-call-summary).

<div class="endpoint"><span class="method get">GET</span><span class="path">/ivr/recordings/{kind}/{id}</span></div>

| | |
| --- | --- |
| Base | Gateway. See [Base URLs](/guide/environments). |
| Auth | [API key](/guide/authentication#api-keys) |

You don't build this URL yourself. Every item in the IVR Call Summary that has a recording
comes with a `recordingUrl`. Call that URL with your API key.

## Parameters

| Parameter | In | Required | Type | Description |
| --- | --- | --- | --- | --- |
| `X-Transaction-Id` | Header | Yes | String | Your ID for this request. See [transaction IDs](/guide/authentication#transaction-ids). |
| `kind` | Path | Yes | String | `out-dial`, `direct-call`, `connect-agent`, `transfer` or `conference`. |
| `id` | Path | Yes | Long | The `id` of the summary item. |

## Request

::: code-group

```bash [cURL]
curl -o call-90211.wav 'https://gateway.dxesk.cloud/pabx/v1/ivr/recordings/connect-agent/90211' \
  -H 'X-API-Key: <YOUR_API_KEY>' \
  -H 'X-Transaction-Id: 3f2b8c1e-6d4a-4f9b-9a7e-2c5d8e1f0a64'
```

```ts [TypeScript]
import { randomUUID } from "node:crypto";
import { writeFile } from "node:fs/promises";

async function downloadRecording(recordingUrl: string, file: string) {
  const res = await fetch(recordingUrl, {
    headers: {
      "X-API-Key": process.env.DWESK_API_KEY!,
      "X-Transaction-Id": randomUUID(),
    },
  });
  if (!res.ok) throw new Error(`recording ${res.status}`);

  await writeFile(file, Buffer.from(await res.arrayBuffer()));
}
```

:::

## Success response

`200 OK` with the audio file as the body.

| Header | Value |
| --- | --- |
| `Content-Type` | Usually `audio/wav`. Can be `audio/mpeg` or `audio/x-gsm` for older recordings. |
| `Content-Disposition` | `attachment; filename="<file name>"` |

The link doesn't expire. It only works with your API key, so you can't put it straight into a
browser or audio player. Download the file on your server first, then serve it from there.

## Error responses

Errors come back as JSON, not audio.

| Status | `code` | Why |
| --- | --- | --- |
| `400` | `INVALID_PARAMETER` | `kind` isn't one of the values above, or `id` isn't a number. |
| `400` | `MISSING_TRANSACTION_ID` | The `X-Transaction-Id` header is missing or empty. |
| `400` | `INVALID_TRANSACTION_ID` | The transaction ID is longer than 64 characters or has characters that aren't allowed. |
| `401` | `UNAUTHORIZED` | The `X-API-Key` header is missing or the key is wrong. |
| `403` | `FEATURE_NOT_ENABLED` | Recordings aren't turned on for your company. |
| `404` | `RECORDING_NOT_FOUND` | There is no call with this `id` for your company, or it has no recording. |
| `429` | `RATE_LIMITED` | You sent more than 60 requests in a minute. Wait `Retry-After` seconds. |
| `500`, `503` | `SERVER_ERROR` | Try again later. |

To download many recordings at once for a date range, use
[Recording Export](/api/recordings) instead.
