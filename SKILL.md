---
name: colab-drive-agent
description: Operate and maintain the Google Colab Drive-based job runner stored in `/Users/yongxinliu/Career/CareerAtERAU/KG Transportation System Sequential/Colab Agent/colab_agent.ipynb`. Use when Codex needs to explain how to run the Colab Agent, keep the required Google Drive folder structure correct, submit single-file `.py` or `.ipynb` jobs or bundled job folders with `job.json`, interpret heartbeat and per-job logs, or use `control.md` to stop active jobs without stopping the watcher.
---

# Colab Drive Agent

Use this skill to help the user run the Colab Agent notebook and manage its Drive-backed queue correctly.

## Source Of Truth

- Notebook: [colab_agent.ipynb]
- Skill file: [SKILL.md]

Treat the notebook as the source of truth for actual runtime behavior.

## Required Drive Layout

The agent expects and the user is responsible for this exact structure under `MyDrive/Colab Agent`:

```text
Colab Agent/
  control.md
  heartbeat.json
  jobs/
    inbox/
    running/
    done/
    failed/
```

Tell the user to follow these rules:

- Submit new `.py` or `.ipynb` single-file jobs, or bundled job folders, only into `jobs/inbox/`.
- Do not manually place files into `jobs/running/`.
- Finished job folders belong in `jobs/done/` or `jobs/failed/`.
- Each completed job folder contains its own execution log.
- `heartbeat.json` is maintained by the watcher.
- `control.md` is the operator control file.
- For a bundled job folder with multiple runnable files, include `job.json` with an `entrypoint` field. That entrypoint may be either a `.py` file or an `.ipynb` file.

## Explain How To Run The Agent

When the user asks how to start the agent, tell them to:

1. Open [colab_agent.ipynb] in Google Colab.
2. Run the setup cells to mount Drive and initialize any missing folders.
3. Confirm the notebook is pointing at `MyDrive/Colab Agent`.
4. Start the watcher with `watch_forever()`.
5. Mirror the Colab Agent/ folder in user's Google Drive to the local, let AI know where it is in the local computer.
6. Leave the Colab runtime connected if continuous job execution is needed.

Warn them that if the runtime disconnects, the watcher stops and the notebook cells must be run again.

## Explain How To Submit Work

When the user asks how to submit a job, tell them to:

1. Put either a single `.py` file, a single `.ipynb` file, or a job folder into `MyDrive/Colab Agent/jobs/inbox/`.
2. If the job is a folder and it contains more than one runnable `.py` or `.ipynb` file, add `job.json` with an `entrypoint` field such as `{\"entrypoint\": \"main.py\"}` or `{\"entrypoint\": \"notebooks/run.ipynb\"}`.
3. Wait for the watcher to move the job into `jobs/running/` while executing it.
4. Check `jobs/done/` for success or `jobs/failed/` for failure or stop.
5. Read `job_result.json` and `execution.log` in the final job folder for the recorded outcome and full output.
6. If the job was a notebook, inspect the executed notebook artifact in that same final job folder.

Be explicit that the notebook executes `.py` scripts directly, `.ipynb` notebooks through `papermill`, and bundled job folders by resolving a single entrypoint file. Mention that the Colab setup installs `papermill` if it is missing. Do not describe it as a general shell-job runner.

When the user asks about `.py` progress, explain that the agent cannot reliably infer the currently executing source line from an arbitrary Python process. Tell the user to add explicit progress prints or write their own progress updates if they want live progress visibility for `.py` jobs.

## Explain GPU Behavior

When the user asks whether notebook jobs can use GPU, explain these rules:

- `papermill` does not enable GPU by itself; it runs the notebook inside the current Colab runtime.
- If the Colab runtime is GPU-backed, the notebook can use the GPU.
- Actual GPU usage still depends on the notebook code and libraries detecting and using the GPU.
- For PyTorch, the notebook still needs checks like `torch.cuda.is_available()` and must move work onto CUDA explicitly.
- For TensorFlow or other frameworks, the notebook must use their normal GPU-aware configuration.

Tell the user that a GPU-backed Colab session is necessary but not sufficient: the notebook code must also be written to use the GPU.

## Heartbeat Semantics

`heartbeat.json` is the liveness signal for the watcher.

When the user asks how to read the heartbeat, explain this expected behavior:

- It updates every 30 seconds while the watcher loop is alive.
- It includes a UTC timestamp.
- It reports watcher state.
- It reports pending and running job counts.
- It reports whether `control.md` currently requests stop.
- It reports runtime telemetry such as memory, disk, load average, and GPU information when available.
- The watcher console can print a one-line runtime summary alongside heartbeat messages.

Give this interpretation:

- Fresh timestamp: the Colab watcher is still alive.
- Stale timestamp: the Colab runtime likely disconnected or the watcher cell stopped.
- State `watching`: the watcher is alive and allowed to launch jobs.
- State `stop-requested`: the watcher is alive but will not launch new jobs while `control.md` contains `stop`.

When the user asks about runtime health, explain that the heartbeat may include:

- RAM totals, used bytes, and usage percent.
- Disk totals and free space for the Drive-mounted workspace.
- Load average when the runtime exposes it.
- GPU device name, memory usage, utilization, and temperature when `nvidia-smi` is available.

When the user asks about watcher console output, explain that:

- Heartbeat lines can include a brief RAM and GPU summary.
- Idle polling lines can also include the same brief runtime summary.
- Notebook jobs can print periodic progress lines while `papermill` is executing cells.

## Explain Saved Outputs

When the user asks whether generated files are preserved, explain these rules:

- Each job runs inside its own job workspace under `jobs/running/<job_id>/...`.
- When the job finishes, that whole job workspace is moved into `jobs/done/<job_id>/` or `jobs/failed/<job_id>/`.
- Files written with relative paths inside the job workspace are preserved with the job result.
- This includes artifacts such as `.pkl`, `.csv`, `.json`, images, and notebook-generated files.
- Files written to unrelated absolute paths outside the job workspace are not automatically tracked or copied back.

Tell the user to write outputs to relative paths inside the job workspace if they want those files to be saved with the job.

## Explain Control File Semantics

`control.md` is a simple operator control file.

Tell the user these rules:

- If it contains `stop`, the active Python job should be terminated.
- The watcher must remain alive and keep writing heartbeats.
- While `stop` remains present, the watcher must not start new jobs.
- Changing `control.md` back to a non-stop value such as `run` allows new jobs to start again.

Describe `control.md` as a pause-and-stop switch for jobs, not a shutdown switch for the watcher.

## Troubleshooting

When troubleshooting for the user, check these cases:

- No new jobs are starting: check whether `control.md` contains `stop`.
- Heartbeat is stale: the Colab runtime or watcher cell likely stopped.
- Jobs stay in `inbox/`: verify the watcher cell is running in Colab.
- A job was interrupted: check `jobs/failed/<job_id>/execution.log` and `jobs/failed/<job_id>/job_result.json`.
- A notebook job failed: inspect `execution.log`, `job_result.json`, and the executed notebook artifact in the final job folder.
- Notebook progress looked stalled: compare the watcher console progress line with `execution.log` in the job folder.
- A `.py` job shows no visible progress: check whether the script prints progress messages; if not, explain that the user needs to add them explicitly.
- A bundled job folder failed before launch: check whether it had multiple runnable files without `job.json` and `entrypoint`.
- The agent appears alive but idle: inspect `heartbeat.json` for state and queue counts.

## Maintenance Guidance

- Preserve the exact folder names unless the notebook is updated to match.
- Preserve support for both single-file jobs and bundled folder jobs unless the user explicitly narrows scope.
- Keep heartbeat and control-file behavior consistent with the notebook.
- If behavior changes in the notebook, update this skill to match the new operating model.
- Prefer concise operator instructions over notebook implementation details unless the user is explicitly editing the agent.
