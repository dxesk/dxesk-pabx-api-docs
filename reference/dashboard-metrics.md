# Dashboard Metrics

This page shows how every number on the Dwesk PBX dashboard is calculated, so you can show the
same numbers on your own dashboard. They all come from the [call records](/api/call-records)
you saved. Use the same rules on your records and you get the same numbers as ours. If you
haven't set up the records yet, start with [Building Your Dashboard](/guide/build-your-dashboard).

## Basic rules

These rules apply to every number on this page.

| Rule | Detail |
| --- | --- |
| Time zone | Sri Lanka time (Asia/Colombo, `+05:30`). Days, hours, weeks and months are all counted in this time zone. |
| Which day a call is on | The date of its `callStartTime`. Records without a `callStartTime` are not counted anywhere. |
| "Today" | From 00:00:00 today up to, but not including, 00:00:00 tomorrow. |
| Percentages | `count ÷ total × 100`, rounded to 1 decimal place. If the total is 0, the result is `0%`. |
| Durations | Worked out in whole seconds. Any part of a second is dropped. Averages are also rounded down to a whole second. |
| Missing times | If a time needed for a duration is `null`, that record is skipped for that average only. It is still counted everywhere else. |

The times already include the `+05:30` offset, so you can read the local date and hour
straight from the text. For `2026-10-08T09:00:59+05:30`, `day` is `"2026-10-08"` and `hour`
is `9`:

```ts
const day = r.callStartTime.slice(0, 10);
const hour = Number(r.callStartTime.slice(11, 13));
```

## Groups

Most numbers count the records in one of these groups.

| Group | A record is in the group if |
| --- | --- |
| Inbound | `callType` is `INCOMING`, `CONNECT_DIAL` or `QUEUE_DIAL` |
| Outbound | `callType` is `OUTGOING` |
| Answered | `callStatus` is `ANSWERED`, `TRANSFER` or `CONFERENCE` |
| Failed | `callStatus` is `FAILED` |
| Voicemail | `callStatus` is `VOICEMAIL` |
| Transfer | `callStatus` is `TRANSFER` |
| Conference | `callStatus` is `CONFERENCE` |
| Ringing | `callStatus` is `RINGING` |

A transferred or conference call counts as answered, because an agent picked it up first. It
is also counted in its own group. So one transferred call adds 1 to Answered and 1 to
Transfer.

A `WAITING` record counts in the totals and in Inbound or Outbound, but not in any status
group.

## Today's summary

These use only today's records.

| Number | How to calculate it |
| --- | --- |
| Total calls | Count of all records. |
| Answered calls | Count of records in Answered. |
| Answer rate | Answered calls ÷ Total calls, as a percentage. |
| Failed calls | Count of records in Failed. |
| Failure rate | Failed calls ÷ Total calls, as a percentage. |
| Calls ringing | Count of records in Ringing. Shown on the Incoming Calls card. When it is 0 the card says "Idle". |
| Inbound calls | Count of records in Inbound. |
| Inbound share | Inbound calls ÷ Total calls, as a percentage. |
| Outbound calls | Count of records in Outbound. |
| Outbound share | Outbound calls ÷ Total calls, as a percentage. |
| Voicemail outcomes | Count of records in Voicemail. |
| Transfers | Count of records in Transfer. |
| Conferences | Count of records in Conference. |
| Handoff events | Transfers + Conferences. |

Inbound share and Outbound share add up to 100%, unless some records have no `callType`.

### Average times

| Number | Which records | Time for each record |
| --- | --- | --- |
| Avg handle time | Answered, with a `callEndTime` | `callEndTime − callStartTime` |
| Ring to connect | Any status, with both `agentRingStartTime` and `agentConnectTime` | `agentConnectTime − agentRingStartTime` |
| Avg wait | Failed, with both `agentRingStartTime` and `callEndTime` | `callEndTime − agentRingStartTime` |

Add up the times in seconds, divide by the number of records, and round down. If no records
qualify, the result is 0.

::: warning Avg handle time starts before the agent answers
It is measured from `callStartTime`, when the call reached the PBX, not from when the agent
answered. Time in IVR menus and ringing is included. If you only want talk time, use
`callEndTime − agentConnectTime`, but then your number won't match our dashboard.
:::

::: info Avg wait only uses failed calls
It shows how long callers who never reached an agent waited, from when an agent's phone
started ringing until the caller hung up. Answered calls are not included.
:::

The dashboard shows durations like `45s`, `3m` or `3m 12s`.

## Hourly distribution

The dashboard shows one bar for each hour of today, during your working hours.

1. Your working hours are the ones you gave us when we set up your PABX. If you didn't give
   any, they are 08 to 18.
2. The bars start at your start hour and stop at your end hour, or at the current hour if
   that is earlier. Both the first and last hour get a bar. Before your start hour there
   are no bars.
3. Each bar is the number of today's records whose `callStartTime` is in that hour. An hour
   with no calls still gets a bar, with 0.
4. Calls outside your working hours are counted in today's summary, but they don't get a
   bar.

| Number | How to calculate it |
| --- | --- |
| Calls in the hour | The count for that bar. |
| Percent of peak | The bar's count ÷ the biggest bar's count × 100, rounded to a whole number. 0% if every bar is 0. |
| Busiest hour | The bar with the biggest count. If two bars are equal, the earlier hour. |
| Calls in busiest hour | The count for that bar. |

Hours are shown as `8AM`, `12PM`, `1PM` and so on. Hour 0 is `12AM`.

## One hour in detail

When you pick an hour of today on the timeline, take today's records whose `callStartTime`
is in that hour, and count:

| Tile | Group |
| --- | --- |
| Incoming | Inbound |
| Answered | Answered |
| Outgoing | Outbound |
| Failed | Failed |
| Voicemail | Voicemail |
| Transfer | Transfer |
| Conference sessions | Conference |

The percentage on each tile is its count ÷ all records that started in that hour.

These are the same groups as today's summary, just for one hour. If you add up a tile over
all 24 hours, you get the same number as in today's summary.

## Call trends

The trend chart counts all records, whatever their status or type, in weeks or months.

| Range | Bars |
| --- | --- |
| 13 weeks | This week and the 12 weeks before it |
| 26 weeks | This week and the 25 weeks before it |
| 12 months | This month and the 11 months before it |
| 24 months | This month and the 23 months before it |

Weeks follow ISO 8601: a week starts on Monday, and week 1 is the week that has the first
Thursday of the year. The label is `W` and the week number, for example `W41`. The last
days of December can be in week 1 of the next year, and the first days of January can be in
week 52 or 53 of the year before. Use an ISO week function from your language or library,
rather than counting days from January 1.

Months are normal calendar months, shown like `Oct 2026`.

A week or month with no calls still gets a bar, with 0.

| Number | How to calculate it |
| --- | --- |
| Calls per bar | Count of records whose `callStartTime` is in that week or month. |
| Calls in range | All the bars added together. |
| Average per bar | Calls in range ÷ number of bars, rounded to a whole number. |
| Busiest period | The bar with the biggest count. If two are equal, the earlier one. |

This week and this month are not over yet, so their bars keep growing during the day.

## Monthly overview

| Number | How to calculate it |
| --- | --- |
| Calls per month | Count of records whose `callStartTime` is in that month. |
| Growth from last month | (This month − last month) ÷ last month × 100, rounded to 1 decimal place, with a + or − sign, like `+80.0%` or `-12.5%`. If last month had 0 calls, don't show a growth figure. |

## Recent call records

The call list shows records newest first, sorted by `callStartTime`.

| Column | What to show |
| --- | --- |
| Call | `callerNumber` |
| Direction | Inbound or Outbound, from the groups above |
| Call ID | `id` |
| Start | `callStartTime` |
| Agent | `agentCli`, matched to your own list of agents. Empty if `null`. |
| Duration | `callEndTime − callStartTime`, shown as `mm:ss`. Shows `N/A` while `callEndTime` is `null`. |
| Status | `callStatus` |

## Reference implementation

This code calculates today's summary, the hourly bars and one hour in detail from a list of
saved records. You can use it to check your own code.

```ts
type Rec = CallRecord;

const INBOUND = new Set(["INCOMING", "CONNECT_DIAL", "QUEUE_DIAL"]);
const ANSWERED = new Set(["ANSWERED", "TRANSFER", "CONFERENCE"]);

const day = (t: string) => t.slice(0, 10);
const hour = (t: string) => Number(t.slice(11, 13));
const secs = (a: string, b: string) =>
  Math.trunc((Date.parse(b) - Date.parse(a)) / 1000);
const pct = (n: number, d: number) => (d ? Math.round((n / d) * 1000) / 10 : 0);

function avgSeconds(values: number[]): number {
  if (!values.length) return 0;
  return Math.floor(values.reduce((s, v) => s + v, 0) / values.length);
}

function groupCounts(rs: Rec[]) {
  const count = (f: (r: Rec) => boolean) => rs.filter(f).length;
  return {
    total: rs.length,
    inbound: count((r) => INBOUND.has(r.callType)),
    outbound: count((r) => r.callType === "OUTGOING"),
    answered: count((r) => ANSWERED.has(r.callStatus)),
    failed: count((r) => r.callStatus === "FAILED"),
    voicemail: count((r) => r.callStatus === "VOICEMAIL"),
    transfer: count((r) => r.callStatus === "TRANSFER"),
    conference: count((r) => r.callStatus === "CONFERENCE"),
    ringing: count((r) => r.callStatus === "RINGING"),
  };
}

export function todaySummary(all: Rec[], today: string) {
  const rs = all.filter((r) => r.callStartTime && day(r.callStartTime) === today);
  const c = groupCounts(rs);

  return {
    ...c,
    answerRate: pct(c.answered, c.total),
    failureRate: pct(c.failed, c.total),
    inboundShare: pct(c.inbound, c.total),
    outboundShare: pct(c.outbound, c.total),
    handoffs: c.transfer + c.conference,
    avgHandleSeconds: avgSeconds(
      rs
        .filter((r) => ANSWERED.has(r.callStatus) && r.callEndTime)
        .map((r) => secs(r.callStartTime!, r.callEndTime!)),
    ),
    avgRingToConnectSeconds: avgSeconds(
      rs
        .filter((r) => r.agentRingStartTime && r.agentConnectTime)
        .map((r) => secs(r.agentRingStartTime!, r.agentConnectTime!)),
    ),
    avgWaitSeconds: avgSeconds(
      rs
        .filter((r) => r.callStatus === "FAILED" && r.agentRingStartTime && r.callEndTime)
        .map((r) => secs(r.agentRingStartTime!, r.callEndTime!)),
    ),
  };
}

export function hourlyBars(
  all: Rec[],
  today: string,
  currentHour: number,
  startHour = 8,
  endHour = 18,
) {
  const last = Math.min(endHour, currentHour);
  const bars: { hour: number; count: number }[] = [];
  for (let h = startHour; h <= last; h++) bars.push({ hour: h, count: 0 });

  for (const r of all) {
    if (!r.callStartTime || day(r.callStartTime) !== today) continue;
    const bar = bars.find((b) => b.hour === hour(r.callStartTime!));
    if (bar) bar.count++;
  }

  const max = Math.max(0, ...bars.map((b) => b.count));
  const busiest = bars.reduce<(typeof bars)[number] | null>(
    (best, b) => (!best || b.count > best.count ? b : best),
    null,
  );

  return {
    bars: bars.map((b) => ({
      ...b,
      percentOfPeak: max ? Math.round((b.count / max) * 100) : 0,
    })),
    busiestHour: busiest?.hour ?? null,
    busiestHourCalls: busiest?.count ?? 0,
  };
}

export function hourBreakdown(all: Rec[], today: string, h: number) {
  const rs = all.filter(
    (r) => r.callStartTime && day(r.callStartTime) === today && hour(r.callStartTime) === h,
  );
  const c = groupCounts(rs);
  const share = (n: number) => pct(n, c.total);

  return {
    incoming: { count: c.inbound, percent: share(c.inbound) },
    answered: { count: c.answered, percent: share(c.answered) },
    outgoing: { count: c.outbound, percent: share(c.outbound) },
    failed: { count: c.failed, percent: share(c.failed) },
    voicemail: { count: c.voicemail, percent: share(c.voicemail) },
    transfer: { count: c.transfer, percent: share(c.transfer) },
    conference: { count: c.conference, percent: share(c.conference) },
  };
}
```

If you have a lot of records, use the same rules in SQL instead of loading every record into
memory. Group by the local date and hour of `callStartTime`, and filter by the groups above.
