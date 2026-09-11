# witness — command recorder

Record what you actually did while debugging a switch, then export the handful of
steps that mattered — with the state changes they caused — straight into the bug
report.

The problem it solves: a repro section gets written from memory two hours after
the evidence scrolled off screen. Everything the assignee needs to reproduce the
bug was on screen at 11:47; this keeps it, and throws away the 35 commands that
were wrong turns.

## Use it

```
./witness start vtb-aviral-01         # record: a rec> prompt, commands run on the DUT
./witness trim                        # keep/drop pass over the recording
./witness export --paste              # markdown timeline → wastebin URL for the ticket
./witness demo                        # replay a recording with no network at all
```

`start` writes `~/.cmdrec/<testbed>-<stamp>/events.jsonl`. Every other subcommand
takes a session directory, an `events.jsonl` path, or nothing (the most recent
recording). Try it without a testbed:

```
./witness export fixture.jsonl
```

Inside the `rec>` prompt: any line runs on the DUT, `mark <note>` flags the
command that just ran, `stop` ends the recording.

## The one design call

**Don't record a terminal — be the terminal.** `rec>` is a REPL that shells each
line out to `nh tb ssh <testbed> -c "<cmd>"` rather than wrapping a PTY and
watching you type. That deletes prompt detection, ANSI parsing, pager handling,
exit-code scraping and every line of credential code, and it hands you the
command text, output, exit code and duration for free because you are the one
running it. `nh tb ssh` already falls back through SONiC and FBOSS credentials,
so both NOSes work with no extra code here.

The cost is honest: you type inside `rec>` instead of a normal SSH session. A PTY
wrapper is the obvious follow-up.

## events.jsonl is the whole interface

Capture writes it, curate reads it, neither side imports the other — which is why
the export path can be built and demoed against `fixture.jsonl` without a
testbed. `fixture.jsonl` is the spec; it is also the fallback demo.

```json
{"event": "command", "seq": 11, "t": "2026-09-11T14:22:07Z", "nos": "sonic",
 "cmd": "config interface shutdown Ethernet24", "exit": 0, "ms": 812, "out": "", "marked": true}
{"event": "state_diff", "seq": 12, "caused_by": 11,
 "changes": [["CONFIG_DB", "PORT|Ethernet24",       "admin_status", "up", "down"],
             ["APPL_DB",   "PORT_TABLE:Ethernet24", "admin_status", "up", "down"],
             ["STATE_DB",  "PORT_TABLE|Ethernet24", "oper_status",  "up", "up"]],
 "anomaly": "oper_status did not follow admin_status"}
{"event": "marker", "seq": 13, "t": "2026-09-11T14:22:19Z", "about": 11, "note": "oper_status stuck up"}
```

A `meta` event at the top carries testbed, NOS version and platform for the
header. A step can be flagged either inline (`"marked": true`) or by a later
`marker` event — the recorder only ever appends, so it uses the marker; the
reader accepts both. Unknown event types are ignored, and a truncated last line
is tolerated, so a recording killed mid-write still loads.

## How a step earns its place in the export

Kept: anything that changed state, anything you marked, and the first read-only
command after a change (the one that corroborates it). Dropped: non-zero-exit
typos, dead-end reads, and consecutive repeats of the same command. `trim` is the
manual override, and it never mutates `events.jsonl` — the keep list lands in
`curated.json` beside it, so trimming is reversible in both directions.

## State diffing

`sudo sonic-db-dump` over a key allowlist (`PORT|*`, `PORT_TABLE:*`,
`PORT_TABLE|*`), once before the command and again 1.5 s after it, in one SSH
round trip each.

The bug this was built for is a *non*-change, which a plain diff hides. So when a
port's `admin_status` moves, its `STATE_DB` `oper_status` is carried into the
diff even when it stayed put — that unchanged row is the evidence, and the one
anomaly rule we ship fires on exactly that shape.

## Known shortcuts

Both are one-day compromises with a known proper version; say so out loud rather
than having them found for you.

- Commands are typed inside `rec>` rather than a normal SSH session → PTY wrapper.
- The settle window is a fixed 1.5 s sleep → SPINE's adaptive window, driven by
  redis keyspace notifications.
- One hardcoded anomaly rule, and the diff is anchored on a small key allowlist
  rather than whole-DB snapshots.

## Next

FBOSS in the same timeline (`nh tb ssh --nos fboss` already authenticates; parse
`show port` into the same change shape), Jira attachment via `jiralib`, anomaly
rules in a YAML file, and auto-record on test failure from a mint or sonic-mgmt
hook.
