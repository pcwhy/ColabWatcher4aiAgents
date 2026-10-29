# Project instructions — Using Foundational Models

## Do not use foreground `sleep` to wait

Do not use foreground commands (`sleep 30`, `while ...; do sleep; done`, `watch`, `tail -f`) to wait for any external event. Use a single check instead of polling; when a real wait is unavoidable, use a background task or ask the user directly.

Rationale (measured on this machine, macOS 15.8 / OpenCode):

- The `sleep` process itself costs almost no CPU; the cost comes from the **waiting mechanism around it**: while a tool call is open, the OpenCode server continuously polls the subprocess state and output streams, the TUI keeps drawing its progress indicator, and the terminal and WindowServer redraw along with it.
- One 15-second foreground `sleep`, measured: the server process gained 1.37 s CPU (≈ 9% of one core), the TUI gained 0.21 s; there are also clear redraw spikes at the start and end of the wait (Terminal / WindowServer momentarily 40%+).
- In other words, the waste is strictly proportional to the wait duration: waiting 10 minutes is about 55 CPU-seconds of pure idling, and the session is blocked the whole time, unable to do anything.

Alternatives:

- Check the current state only once (`cat` / `stat` / `ls`) and judge freshness from an existing timestamp — do not sample repeatedly.
- If a real wait is needed, put it in the background, or tell the user to check back later.
- For "check again later", use one background command that notifies on completion, instead of blocking in the foreground.

## Only one Colab watcher — jobs are strictly serial

The user runs **one** Colab watcher. Only one job executes at a time; everything else queues in `inbox/` — **there is no parallelism**.

Operating rules that follow from this:

- **Submit only one job at a time**, and wait for it to reach `done/` or `failed/` before submitting the next. Do not submit several at once, and especially not interdependent jobs: the later one would start queuing before the earlier one's outputs exist.
- Before submitting, check `running/` and `inbox/` to confirm the queue is empty.
- Size jobs for **20–25 minutes** (the notebook's hard limit `JOB_TIMEOUT_SECONDS` = 30 minutes; on timeout it is terminated and placed in `failed/`). Since it is serial, **fewer jobs means lower total latency** — do not split a 25-minute task into five 5-minute jobs just to isolate failures.
- Each job writes its own intermediate outputs to relative paths (checkpoints, partial CSVs, logs), so a single job failure does not require restarting from scratch.
- After submitting, use a background notification — do not poll in the foreground.

(The earlier note "3–5 jobs per batch" was written before knowing there is only one watcher; it is obsolete.)

## Environment notes

- Google Drive is mapped as a streaming cache: `~/Google Drive` → `~/Library/CloudStorage/GoogleDrive-gmliuyongxin@gmail.com`. After a remote change, the local side may briefly return stale content; to judge whether a file is updated, look at both the timestamp inside the content and `mtime`, not just a single read.
- Colab job queue: `~/Google Drive/My Drive/Colab Agent`; see the skill `.opencode/skills/colab-drive-agent/` for the operating procedures.
