# Building Your Dashboard

This guide explains how to put the numbers from the Dwesk PBX dashboard into your own
system: today's calls, answered and failed calls, answer rate, average times, busiest
hours, weekly and monthly trends, and the list of recent calls.

## How it works

Dwesk doesn't send you finished numbers. We send you the calls themselves, and your system
counts them.

Each call is one **call record**. It says who called, which agent took the call, when the
phone rang, when the agent answered, when the call ended, and how it ended. Every number on
the Dwesk dashboard is counted from these records, and we publish the exact rule for each one.
If you have the same records and use the same rules, you
get the same numbers as our dashboard.

Because you have the calls and not just totals, you can also count things our dashboard
doesn't: per team, for any date range, or together with data from your CRM.

You set it up in four steps:

| Step | What happens | How often | You use |
| --- | --- | --- | --- |
| 1. Fetch your history | You get every call your company has made so far | Once | [Call Records](/api/call-records) |
| 2. Keep it up to date | Dwesk sends you new and changed calls | Every 5 minutes | [Call Records Feed](/webhooks/call-records-feed) |
| 3. Work out the numbers | Your system counts the calls | Whenever your dashboard refreshes | [Dashboard Metrics](/reference/dashboard-metrics) |
| 4. Check your totals | You compare your counts with Dwesk's | For example once a day | [Call Record Counts](/api/call-record-counts) |

::: info Getting access
These endpoints use an API key, not the username and password used on the rest of this site.
Ask your Dwesk contact for access. You get your API key, your gateway base URL and your feed
secret. Every request also needs a transaction ID.
[Authentication](/guide/authentication#api-keys) explains both.

Build your integration to this guide. If something is unclear, or you need it to work
differently, contact us.
:::

## Step 1: Fetch your history

Trends go back up to 24 months, and the monthly overview compares this month with last
month, so your dashboard needs past calls as well as today's. Fetch them all once, before
anything else.

Call [Call Records](/api/call-records) without a cursor. Each response gives you a
`nextCursor`. Call again with it, and keep going until `hasMore` is `false`. You then have
every call for your company, from the very first one, oldest first.

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
fetch can take a while.

## Step 2: Keep it up to date

After the first fetch, Dwesk sends new and changed calls to your
[feed endpoint](/webhooks/call-records-feed) every 5 minutes. Save them, and your dashboard
stays at most about 5 minutes behind ours.

### A record can change

We create a call record as soon as the call starts and update it as the call goes on. So you
may first get a call with status `RINGING`, which your dashboard counts under "Calls ringing".
A few minutes later you get the same call again with status `ANSWERED` and an end time, and it
moves to "Answered calls". Both versions have the same `channelId`.

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
If you do, one call ends up as several rows and every number on your dashboard comes out too
high. We often send the same record more than once, on purpose, so that no change is missed.
Saving one row per `channelId` makes that safe.
:::

### Spotting a missed delivery

Each delivery covers a time period, from `from` to `to`, and includes a `cursor`. Each time
you finish handling a delivery, save its `to` and its `cursor`. When the next one arrives,
compare its `from` with the `to` you saved:

| Check | What it means | What to do |
| --- | --- | --- |
| `from` is the same as or earlier than your saved `to` | You missed nothing | Save the records, then save the new `to` and `cursor` |
| `from` is later than your saved `to` | You missed one or more deliveries | Call the Call Records endpoint with your saved `cursor` until `hasMore` is `false`, then save this delivery |

Each delivery starts 2 minutes before the previous one ended. So normally `from` is a little
earlier than your saved `to`, and a few records arrive twice. That's fine, because you save
one row per `channelId`.

### If you missed something

The same check works whatever went wrong:

| What happened | What you will see |
| --- | --- |
| Your server was down and Dwesk stopped retrying | The next delivery's `from` is later than your saved `to` |
| Dwesk had a problem and skipped some deliveries | Same as above |
| Your server replied `200` but didn't save the data | Same as above, because you never saved the new `to` |
| You lost your database | You have no saved cursor. Start again from Step 1 |

You never need to ask us to send anything again. Call the Call Records endpoint with your last
saved cursor and you get every call that changed after it, including the ones from deliveries
you missed.

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

## Step 3: Work out the numbers

Now count your saved calls. Each part of the Dwesk dashboard has its own section in
[Dashboard Metrics](/reference/dashboard-metrics):

| On the Dwesk dashboard | How to work it out |
| --- | --- |
| Total, answered and failed calls, answer rate, calls ringing, inbound and outbound, voicemail, transfers, conferences | [Today's summary](/reference/dashboard-metrics#today-s-summary) |
| Average handle time, ring to connect time, wait time | [Average times](/reference/dashboard-metrics#average-times) |
| Calls per hour chart and busiest hour | [Hourly distribution](/reference/dashboard-metrics#hourly-distribution) |
| What happened to the calls in one hour | [One hour in detail](/reference/dashboard-metrics#one-hour-in-detail) |
| Weekly and monthly trend chart | [Call trends](/reference/dashboard-metrics#call-trends) |
| Calls per month and growth from last month | [Monthly overview](/reference/dashboard-metrics#monthly-overview) |
| Recent calls list | [Recent call records](/reference/dashboard-metrics#recent-call-records) |

The same page also has [TypeScript code](/reference/dashboard-metrics#reference-implementation)
for these calculations that you can copy.

Recalculate right after you save each feed delivery. Our own dashboard updates every 10
seconds, so yours will be up to about 5 minutes behind it.

## Step 4: Check your totals

[Call Record Counts](/api/call-record-counts) tells you how many calls Dwesk has for each day,
split by status and by type. Count your own saved calls the same way and compare. If a day
doesn't match, your dashboard numbers for that day will be wrong too. Fix it yourself, without
contacting us:

- If you have fewer calls than Dwesk, call the Call Records endpoint with your last saved
  cursor until `hasMore` is `false`.
- If the totals match but the statuses don't, a newer version of a call didn't replace the old
  one. Check [A record can change](#a-record-can-change).
- If it still doesn't match, fetch everything again from Step 1, without a cursor. Because you
  save one row per `channelId`, this fixes your data without making duplicates.

Only compare days that are over. Today's numbers keep changing while calls are going on.

## Time zone

All times are in Sri Lanka time (Asia/Colombo), written as ISO 8601 with `+05:30` at the end.
"Today", hours, weeks and months on the dashboard are all counted in Sri Lanka time. If your
servers use UTC, convert the times first. Otherwise calls near midnight will land on a
different day than on our dashboard.

## What you can't build from call records

A call record is about one call. It doesn't say anything about the PBX itself, so a few things
can't come from it:

| Not included | Why |
| --- | --- |
| Gateway, engine or trunk status | These are about the PBX, not about a call. |
| Hold time | The PBX doesn't record hold time separately. |
| Your answer rate target | This is your own setting. Keep it in your own system. |
| Agent names and emails | Records only have the agent's phone number, `agentCli`. Match it to your own list of agents. |
| IVR flow details | Use [IVR Call Summary](/api/ivr-call-summary) and [IVR Reports](/api/ivr-reports). |

"Calls ringing" works, but it is only as fresh as the last delivery, so it can be up to 5
minutes behind.
