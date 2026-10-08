# Syncing Call Records

Dwesk does not send you dashboard numbers. We send you the call records, and you calculate
the numbers yourself. Every number on the Dwesk PBX dashboard comes from these records.
[Dashboard Metrics](/reference/dashboard-metrics) shows how each one is calculated. If you
have the same records as we do, you will get the same numbers.

You use three things:

| What | Who calls who | What it is for |
| --- | --- | --- |
| [Call Records](/api/call-records) | You call Dwesk | Download all your records the first time, and fill in anything you missed |
| [Call Records Feed](/webhooks/call-records-feed) | Dwesk calls you | Get new and changed records every 5 minutes |
| [Call Record Counts](/api/call-record-counts) | You call Dwesk | Check that your totals match ours |

::: info You get access after we agree the integration
These endpoints use an API key, not the username and password used on the rest of this site.
Once the integration is agreed, Dwesk gives you the API key, the gateway address and the feed
secret. See [Authentication](/guide/authentication#api-keys).
:::

## A record can change

We create a call record when the call starts, and we update it as the call goes on. So you
may first get a call with status `RINGING`, and a few minutes later get the same call again
with status `ANSWERED` and an end time. Both versions have the same `channelId`.

Save one row per `channelId`. When a record arrives:

- if you don't have that `channelId` yet, insert it;
- if you already have it, replace it, unless the one you have is newer (compare `updatedAt`).

```ts
async function saveRecord(r: CallRecord) {
  const stored = await db.callRecord.findUnique({ where: { channelId: r.channelId } });

  if (stored && new Date(stored.updatedAt) > new Date(r.updatedAt)) return;

  await db.callRecord.upsert({
    where: { channelId: r.channelId },
    create: r,
    update: r,
  });
}
```

::: danger Don't insert every record as a new row
If you do, one call ends up as several rows and all your counts come out too high. We often
send the same record more than once (this is on purpose, see below). Saving one row per
`channelId` makes that safe.
:::

## Step 1: download everything

Call [Call Records](/api/call-records) without a cursor. Each response gives you a
`nextCursor`. Call again with it, and keep going until `hasMore` is `false`. You now have
every record for your company, from the very first call, oldest first.

```ts
let cursor = await loadCursor();
let page;

do {
  page = await getCallRecords({ cursor, limit: 1000 });
  for (const r of page.records) await saveRecord(r);
  cursor = page.nextCursor;
  await saveCursor(cursor);
} while (page.hasMore);
```

Save the cursor after every page. If your job stops halfway, start again from the saved
cursor, not from the beginning.

You don't need to wait between pages. If your company has a lot of history, this first
download can take a while.

## Step 2: receive changes every 5 minutes

After that, Dwesk sends the new and changed records to your
[feed endpoint](/webhooks/call-records-feed) every 5 minutes. Each delivery covers a time
period, from `from` to `to`, and includes a `cursor`.

Each time you finish handling a delivery, save its `to` and its `cursor`. When the next one
arrives, compare its `from` with the `to` you saved:

| Check | What it means | What to do |
| --- | --- | --- |
| `from` is the same as or earlier than your saved `to` | You missed nothing | Save the records, then save the new `to` and `cursor` |
| `from` is later than your saved `to` | You missed one or more deliveries | Call [Call Records](/api/call-records) with your saved `cursor` until `hasMore` is `false`, then save this delivery |

Each delivery starts 2 minutes before the previous one ended. So normally `from` is a little
earlier than your saved `to`, and a few records arrive twice. That's fine, because you save
one row per `channelId`.

## If you missed something

The same check works whatever went wrong:

| What happened | What you will see |
| --- | --- |
| Your server was down and Dwesk stopped retrying | The next delivery's `from` is later than your saved `to` |
| Dwesk had a problem and skipped some deliveries | Same as above |
| Your server replied `200` but didn't save the data | Same as above, because you never saved the new `to` |
| You lost your database | You have no saved cursor. Start again from Step 1 |

You never need to ask us to send anything again. Call [Call Records](/api/call-records) with
your last saved cursor and you get every record that changed after it, including the ones
from deliveries you missed.

```ts
async function onFeedDelivery(batch: CallRecordsFeedBatch) {
  const saved = await loadSyncState();

  if (saved && new Date(batch.from) > new Date(saved.to)) {
    let cursor = saved.cursor;
    let page;
    do {
      page = await getCallRecords({ cursor, limit: 1000 });
      for (const r of page.records) await saveRecord(r);
      cursor = page.nextCursor;
    } while (page.hasMore);
  }

  for (const r of batch.records) await saveRecord(r);

  let cursor = batch.cursor;
  if (!batch.complete) {
    let page;
    do {
      page = await getCallRecords({ cursor, limit: 1000 });
      for (const r of page.records) await saveRecord(r);
      cursor = page.nextCursor;
    } while (page.hasMore);
  }

  await saveSyncState({ to: batch.to, cursor });
}
```

## Step 3: check your totals

[Call Record Counts](/api/call-record-counts) tells you how many records Dwesk has for each
day, split by status and by type. Count your own records the same way and compare. If a day
doesn't match, you know which day to download again, and you don't need to contact us.

Only compare days that are over. Today's numbers keep changing while calls are going on.

## Time zone

All times are in Sri Lanka time (Asia/Colombo), written as ISO 8601 with `+05:30` at the end.
"Today", hours, weeks and months are all counted in Sri Lanka time. If your servers use UTC,
convert the times first. Otherwise calls near midnight will land on a different day than on
our dashboard.

## What the records don't include

A call record is about one call. It doesn't say anything about the PBX itself, so you can't
calculate these from it:

| Not included | Why |
| --- | --- |
| Gateway, engine or trunk status | These are about the PBX, not about a call. |
| Hold time | The PBX doesn't record hold time separately. |
| Your answer rate target | This is your own setting. Keep it in your own system. |
| Agent names and emails | Records only have the agent's phone number, `agentCli`. Match it to your own list of agents. |
| IVR flow summaries | Not part of call records. |

You can calculate "calls in queue", but it is only as fresh as the last delivery, so it can
be up to 5 minutes behind.
