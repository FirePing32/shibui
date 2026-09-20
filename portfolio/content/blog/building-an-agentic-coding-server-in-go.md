---
title: "Building an Agentic Coding Server in Go"
date: 2026-09-20T18:30:00+05:30
comment: false
tags: ["go", "agents", "llm", "architecture", "harness"]
---

# What if the agent loop lived behind the API instead of in your client?

Every coding agent you've used — Claude Code, Cursor, Aider — is the same shape
underneath. The model asks to read a file. Something reads it. The result goes
back. The model asks to edit. Repeat until it stops asking. That loop is where
most of the engineering lives, and it almost always lives in a client on your
laptop.

What happens if you move it to the other side of an HTTP endpoint? Not a
framework, not an SDK — an **OpenAI-compatible server** you can point any
existing client at. You send one chat completion request; the server
runs fifteen model calls and a pile of tool invocations; you get one answer
back.

The result is [**harness-api**](https://github.com/FirePing32/harness-api): about
12,000 lines of Go, another 11,000 of tests, and exactly one runtime dependency.
What follows are the design decisions that turned out to matter, most of which
were not the obvious ones.

---

## Getting started

### Running it

```sh
export HARNESS_UPSTREAM_BASE_URL=https://api.openai.com/v1
export HARNESS_UPSTREAM_API_KEY=sk-...
export HARNESS_UPSTREAM_MODEL=gpt-4.1

./harness-api
```

Then use the OpenAI SDK you already have:

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8080/v1", api_key="unused")

resp = client.chat.completions.create(
    model="gpt-4.1",
    messages=[{"role": "user", "content": "rename oldName to newName everywhere"}],
    extra_body={"harness": {"workspace": "/path/to/your/project"}},
)
print(resp.choices[0].message.content)
```

No new client library. No new protocol. The agent is a server.

### The shape of the system

![Architecture](./../assets/harness-api-architecture.svg)

---

## Why a harness matters at all

Start with the uncomfortable framing: **a harness cannot make a weak model
strong.** Agentic capability is dominated by the model, full stop.

What a harness controls is how much capability gets *left on the table* — and
that gap is large. The same model, same weights, same prompt, can swing
dramatically on agentic benchmarks depending on whether its tools return
instructive errors, whether its context gets managed sensibly, and whether it
can tell "you did something wrong" apart from "the tool is broken."

That reframing changes what you optimise. You're not building intelligence.
You're building the thing that stops intelligence being wasted.

---

## The tools

The tool set is deliberately small: `read`, `write`, `edit`, `glob`, `grep`,
`bash`. Adding more tools sounds like adding capability; mostly it adds ways for
the model to pick the wrong one.

### `edit` decides whether the whole thing works

It does exact byte-for-byte string replacement — no fuzzy matching, no
whitespace normalisation, no regex. That sounds unhelpfully strict, and it is the right
call, because a fuzzy match that picks the *wrong* location produces a change
nobody asked for and nobody notices.

The strictness only works if failure is instructive. So when a match fails:

```
No match for that text in src/config.go.

The closest similar line is 42:
    42 |     timeout = 30
                ^^^^ you passed 4 spaces here, the file has a tab
```

And when it matches too many times, it names the line numbers rather than
saying "ambiguous":

```
That text appears 3 times (lines 12, 45, 89). Include more surrounding
context so the match is unique.
```

The principle underneath: **tool error messages are implementation, not
decoration.** They are the model's only recovery signal. A message that says
what went wrong but not what to do next turns into a retry loop, and retry loops
are where agent runs go to die.

### Read-before-edit is a version check, not a flag

This is the detail most worth stealing.

The obvious way to stop a model editing a file it hasn't looked at is a boolean:
*has the model read this path?* It catches the common failure and misses a
nastier one.

![Ledger](./../assets/harness-api-ledger.svg)

The model reads a file, runs a build script that rewrites that file, then edits
it based on what it remembers. With a boolean, the edit sails through and the
script's work is silently destroyed. Nothing throws. The edit succeeds.

So instead of a flag, the server records a **content hash** per path. An edit is
authorised only if the file is still byte-for-byte what was observed. Any change
from any source — a shell command, a formatter, a concurrent process —
invalidates it and forces a re-read.

There's a subtle follow-on. When the conversation gets compacted (more on that
below) and the summariser drops the turn that held a file's contents, the model
no longer *has* those bytes — but the hash entry survives and keeps saying yes.
The invariant quietly degrades into a rubber stamp. So compaction marks affected
observations stale, and the error says *"you read this, but that part of the
conversation was summarised away"* rather than *"you never read this"*. Telling a
model it didn't do something it plainly did invites it to argue with the tool
instead of retrying.

### The path jail, and a race worth designing out

File tools must not escape the workspace. The traditional approach is: resolve
symlinks, compare against the root prefix, reject if outside.

That approach is a time-of-check-to-time-of-use race **by construction**.
Between your check and your `open()`, anything can swap a directory component
for a symlink pointing at `/etc`.

Go 1.24 added [`os.Root`](https://pkg.go.dev/os#Root), which resolves every path
component with `openat` relative to a held directory descriptor. It's a kernel
check, not a string comparison, and the race disappears — not through
carefulness, but because there is no window left to exploit.

The sharpest test plants a symlink *after* the path has been validated and
asserts the open still fails.

One honest caveat that belongs in any post like this: **`bash` is not jailed.**
`cd /etc && cat passwd` works. The path jail covers the file tools and nothing
else. That's documented loudly rather than implied away, because a security
boundary you *think* you have is worse than one you know you don't.

### Truncation should be asymmetric

Both file reads and command output need capping. The interesting bit is that
they should be capped from **opposite ends**.

- **File views drop the tail.** You want the top of the file, and a footer
  telling the model how to continue (`offset=2001`, or "search with grep first").
- **Shell output drops the head.** The error in a failed build is at the
  *bottom*, under four hundred lines of compilation progress.

Cap both the same way and you throw away the one thing that mattered. The full
output spills to a file in the workspace so the model can page through it.

### `bash` is stateless, and that deleted the hardest code

The obvious design is a persistent shell — keep a `bash` process alive, feed
it commands, read results back. This is genuinely hard: you need sentinel framing
to find command boundaries in a shared stream, detection and respawn when the
shell dies, pipe bookkeeping, and a story for interleaved output.

[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) does
something simpler: a fresh `bash -c` per call, with a `workdir` argument instead
of `cd`.

Adopting that deleted the hardest code in the project. What a persistent shell
would preserve — environment variables, working directory — either gets passed
explicitly or lives in the filesystem, which persists anyway.

One measurement worth sharing. On timeout, you must kill the whole process
group, or `npm test` orphans node workers that hold ports for hours. SIGINT
is the obvious signal. It doesn't work: **POSIX requires a non-interactive shell to
set SIGINT to *ignored* in background children**, so `something &` inside
`bash -c` cannot be interrupted by SIGINT at all — bash dies, the child keeps
running. SIGTERM, then SIGKILL after a grace period. That came out of a
controlled experiment against a real process group, rather than from adjusting
the test until it passed.

---

## Keeping the agent in bounds

### Guards can only deny, never permit

Every tool call passes a chain of guards. The contract is one line:

```go
type Guard interface {
    Name() string
    Check(Execution) string  // "" means no opinion
}
```

There is **no allow result**. A guard can refuse or abstain, nothing else.

That sounds like a limitation and it's the entire point: it makes the chain
*monotonic*. No guard can undo another's refusal, so adding one can only ever
make the system more cautious, and ordering affects which *message* the model
sees and nothing else. It removes a whole bug class where "is this permitted"
has a different answer depending on registration order.

### The denylist is ergonomics, not security

The built-ins are a repeated-identical-call detector, a destructive-command
denylist, and a timeout policy. The denylist is labelled in the docs as
**ergonomics, not a security boundary** — command text has unlimited ways to say
the same thing, and anything *trying* to get past it will. What it catches is
the accident: a path built badly so `rm -rf /tmp/build/` becomes `rm -rf /`.

A false positive here is worse than a miss, incidentally. A model can't tell a
policy refusal from a bug, so it rephrases the same command instead of adapting.
Patterns are anchored to command position so a dangerous word inside a quoted
string doesn't trip them.

---

## Managing the context window

### Prune before you summarise

Long runs overflow the context window. The obvious fix is to summarise older
turns with an extra model call.

That's the *second* thing you should do. A long agent run overflows the window
not because the conversation is long, but because **tool results are large**.
Twenty file reads at 40 KB each is most of a context window, and almost none of
it is still needed — the model read the file, made its edit, moved on.

So the first pass just drops the *bodies* of old tool results, replacing them
with a stub that names the file and says it can be re-read. It costs nothing, no
model call, and on a typical run it's enough. Only when that fails does the
summariser run.

Getting the order wrong spends a model call and loses detail to solve a problem
that deleting stale file contents would have solved for free.

Two details that are easy to get wrong:

**Never cut between an assistant's `tool_calls` and the results answering them.**
Strict providers reject the *entire request*, which surfaces as a 400 on the turn
*after* compaction with nothing pointing at compaction as the cause.

**Keep the original task verbatim.** The system prompt and the first user message
survive untouched. If a summary paraphrases the goal loosely, the agent carries
on with no idea what it was asked to do, and the run fails in a way that looks
like model incompetence rather than context loss.

### Token estimation that corrects itself

Compaction needs to know how close you are to the window. That needs a token
count, and this server talks to *whatever you point it at* — OpenAI's tokenizer
tells you nothing useful about Llama or Qwen.

Vendoring a tokenizer would be wrong for most providers. So instead: estimate
from byte count, and correct the ratio against reality. Every response carries
`usage.prompt_tokens` for a request whose byte count you know exactly — a free
labelled sample, every single turn.

An exponentially-weighted moving average over those samples converges on
whatever tokenizer the provider actually uses, within a few turns, with zero
dependencies. Implausible samples get discarded (a cached prompt, a provider
counting images), and while it's still unsure it deliberately reads *high* —
because underestimating means overflowing, which is a failed request, while
overestimating costs one unnecessary summarisation.

---

## Talking to any provider

### Quirks are data, not code

"Works with any OpenAI-compatible model" is a sentence that hides a lot of pain:

- `max_tokens` vs `max_completion_tokens`
- `system` vs `developer` role, or no system role at all
- empty content with tool calls: `null`, `""`, or omitted
- `parallel_tool_calls` accepted, or a 400
- JSON Schema dialect gaps that break vLLM and Ollama
- `reasoning_content` that must **never** be echoed back (DeepSeek 400s on it)
- SSE vs NDJSON framing, missing `[DONE]` terminators

There are more providers than anyone will write structs for. So compatibility is
a `Profile` struct — plain data — selected by configuration, with per-field JSON
overrides. A provider nobody has heard of is a config change, not a code change.

There's also an autodetect safety net: on a 400 matching a known pattern, apply
the corresponding fix, log what to pin, and retry. Capped at three adjustments
per model so a rejection nothing can be inferred from fails once instead of
looping forever.

### Streaming, and what it actually buys

`"stream": true` works, with a limitation worth stating plainly.

An agent run has several generations and only the last one is the answer. The
others are the model saying "let me check that file" before a tool call.
Concatenating them reads like a transcript of someone thinking out loud, so only
the final turn gets emitted.

The consequence: the final turn isn't known to be final until it arrives without
tool calls. So its content is produced *before* streaming begins, then sent in
pieces so progressive renderers behave normally. There is **no time-to-first-token
benefit** over a non-streaming request. What streaming buys is the connection
staying open, and structured progress events if you opt in:

```json
{"type": "tool_start", "turn": 3, "tool": "edit", "call_id": "c7",
 "args": {"path": "main.go"}, "summary": "edit main.go"}
```

Those ride on `choices[0].delta.harness`. Standard SDKs ignore unknown keys
*inside* `delta` — verified against `openai-python` — so it's safe to leave on
with a client that's never heard of it.

---

## Measuring any of it

Everything above is an argument. Arguments are cheap. So the repo includes an
eval harness, and its design choices are more opinionated than the server's.

### Checks are programs, not LLM judges

A judge lets a task grade prose, which
is tempting and wrong — judges disagree with themselves across runs, and a
regression detector that is itself noisy detects noise. Every task exits zero or
it doesn't.

### A failed measurement is not a failed task

A run that never reached the model
— rate limited, server down — is excluded from the pass-rate denominator and
reported separately, loudly. Otherwise a bad afternoon on your provider looks
exactly like a capability regression, and those call for opposite responses.

### It refuses to call small differences real

The comparison runs a Fisher exact
test before declaring a winner. At three repetitions per task:

| Config A | Config B | p | Verdict |
|---|---|---|---|
| 20/30 | 24/30 | 0.38 | nothing |
| 18/30 | 24/30 | 0.16 | nothing |
| 0/30 | 30/30 | 10⁻¹⁷ | real |

A six-run gap at n=30 is *still not significant*. That's not pessimism about the
tool; it's what thirty runs buys. A comparison tool that can't say so gets used
to justify changes that did nothing — and, worse, to reject changes that helped.

### The metric that matters most

It isn't pass rate. It's **tool error rate, per tool**.
If `edit` errors on a third of its calls, the fix is in `edit.go`, not in the
prompt — and without that number there's no way to tell those two hypotheses
apart.

---

## Five things worth knowing

In rough order of how surprising they were:

1. **Error messages are product surface.** Not logging, not diagnostics.
   The model's recovery behaviour is downstream of your error text.
2. **Matching a good reference beats inventing.** Reading DeepSeek Harness at
   source deleted more code than it added.
3. **Measure the platform instead of trusting the docs.** SIGINT vs SIGTERM,
   `setrlimit` not existing where the documentation implied, `ulimit` block
   sizes differing between shells — three separate times, the assumed answer
   was wrong.
4. **Monotonic beats configurable.** Deny-only guards can't be misordered into a
   security hole.
5. **Build the measurement before you believe the design.** Especially your own.

The code is on GitHub: [**FirePing32/harness-api**](https://github.com/FirePing32/harness-api).
MIT licensed. Go 1.26, one runtime dependency, `net/http` routing, `log/slog`
logging, no OpenAI SDK — the quirks layer needs byte-level request control and
the official SDKs fight unknown fields.

If you build something with it, or find a place where the reasoning above is
wrong, open an issue.
