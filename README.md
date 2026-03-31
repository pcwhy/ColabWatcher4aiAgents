# Colab Drive Agent for Local AI agents to leverage Google Colab computational power.

`colab-drive-agent` is a local skill plus Colab workflow for running Python scripts and Jupyter notebooks from a Google Drive-backed job queue.

This folder is intended to travel well in GitHub: it contains the skill instructions for future Codex sessions and sits alongside the Colab notebook/operator assets used by the workflow.

## What It Does

- Watches a Google Drive folder from a live Google Colab session, the session should running "colab_agent.ipynb". **Please remember to uncomment the last line in the last code cell**
- Accepts single-file `.py` jobs
- Accepts single-file `.ipynb` jobs
- Accepts bundled job folders with `job.json` and an `entrypoint`
- Writes `heartbeat.json` so the session can be monitored
- Supports `control.md` as a stop/pause signal
- Saves logs and outputs inside each job result folder

## Key Files

- [SKILL.md](./SKILL.md): agent-facing operating instructions
- [agents/openai.yaml](./agents/openai.yaml): UI metadata for the skill
- [../colab_agent.ipynb](../colab_agent.ipynb): the Colab notebook that runs the watcher
- `Colab Agent` in Google Drive: runtime queue root used by the notebook

## Drive Layout

The Colab notebook expects this structure under `MyDrive/Colab Agent`:

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

## Job Types

### Single-file script

Drop a Python file into:

```text
jobs/inbox/my_job.py
```

### Single-file notebook

Drop a notebook into:

```text
jobs/inbox/my_notebook.ipynb
```

Notebook jobs are executed through `papermill`.

### Bundled folder job

Use a folder when the job needs helper files:

```text
jobs/inbox/my_bundle/
  job.json
  main.py
  helper.py
```

Example `job.json`:

```json
{
  "entrypoint": "main.py"
}
```

The `entrypoint` may also point to a notebook, for example:

```json
{
  "entrypoint": "notebooks/run.ipynb"
}
```

## How To Run

1. Open [../colab_agent.ipynb](../colab_agent.ipynb) in Google Colab. **Make sure you enabled GPU Runtime**
2. Run the setup cells to mount Google Drive, **check the code so that you know which folder in your Google Drive you are mounting**, the code will initialize folders.
3. Mirror the **../Colab Agent** folder to your local PC. Let AI know where it is in the local computer and let AI read this file or the Skill.md.
4. Make sure the `watch_forever()` function is running with heartbeats generated.
5. Submit jobs into `jobs/inbox/` of the **Colab Agent/** folder in your Google Drive.

## Outputs

Each job is executed inside its own job workspace under `jobs/running/<job_id>/`.

When the job finishes, that whole workspace is moved into either:

- `jobs/done/<job_id>/`
- `jobs/failed/<job_id>/`

Typical artifacts include:

- `execution.log`
- `job_result.json`
- executed notebook output such as `name.executed.ipynb`
- any relative-path outputs produced by the job, such as `.pkl`, `.csv`, images, or model artifacts

If a script or notebook writes files outside its job workspace, those files are not automatically tracked by the agent.

## Heartbeat And Control

- `heartbeat.json` is updated periodically while the watcher is alive
- `control.md` can be used to place the system in `stop` mode
- in `stop` mode, active jobs are terminated and new jobs are not launched
- the watcher should remain alive and continue writing heartbeat updates

Heartbeat runtime information may include:

- memory usage
- disk usage
- load average
- GPU name, utilization, temperature, and memory usage when available

## GPU Behavior

If the Colab runtime is GPU-backed, jobs can use the GPU, but only if the job code itself is written to use it.

`papermill` does not enable GPU on its own; it simply runs the notebook in the current Colab runtime.

## Progress Reporting

- Notebook jobs can show execution progress through `papermill`
- `.py` jobs do not expose line-by-line progress automatically
- if you want visible progress for `.py` jobs, add explicit `print(...)` progress messages in the script

## Intended Use Of This Folder

This folder is best treated as:

- a reusable skill for future Codex sessions
- a GitHub-shareable operator guide
- a local companion to the Colab notebook that actually runs the Drive watcher
