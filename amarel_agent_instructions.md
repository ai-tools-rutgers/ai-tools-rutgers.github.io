# Agent Instructions for Rutgers Amarel

Use this document as the starting prompt or reference file for a coding agent
that will help a Rutgers user work on Amarel. It is written for terminal coding
agents from OpenAI, Anthropic, and similar providers.

## Goal

Help the user run research code on Amarel through a safe, repeatable workflow:

1. connect to Amarel over SSH;
2. stage files and scripts from the local machine;
3. run CPU or GPU work through Slurm, not on the login node;
4. collect logs and outputs back on the local machine; and
5. keep every remote action auditable.

## Required User Inputs

Ask the user for these details before doing remote work:

- Rutgers NetID;
- project name or task name;
- whether the job needs CPU only or GPU;
- expected runtime, memory, and number of CPU cores;
- where outputs should be stored locally.

Do not guess account-specific values such as NetID, allocation account,
partition, or project paths.

## SSH Setup

Prefer a named SSH host so future commands are simple:

```sshconfig
Host amarel
  HostName amarel.rutgers.edu
  User <NETID>
```

After the user confirms the SSH config, test only lightweight commands:

```bash
ssh amarel 'hostname'
ssh amarel 'pwd; whoami'
```

If SSH requires Duo, VPN, keys, or Rutgers-specific authentication, pause and
ask the user to complete that step.

## Off-Campus Access

If the user is not physically on the Rutgers campus network, ask them to connect
to the Rutgers VPN before trying Amarel. Rutgers VPN uses Cisco AnyConnect /
Cisco Secure Client. The setup entry point is:

```text
https://vpn.rutgers.edu
```

After the VPN is connected, retry only a lightweight SSH test before staging or
running any jobs.

## Agent Runtime Profile

For fully automated remote work, the agent environment needs permission to use
networked commands such as `ssh`, `scp`, and `rsync`.

Recommended hands-off profile for trusted sessions:

```toml
approval_policy = "never"
network_access = "enabled"
sandbox_mode = "danger-full-access"
```

More conservative profile:

```toml
approval_policy = "on-failure"
network_access = "enabled"
sandbox_mode = "workspace-write"
```

If the environment requires approvals, bundle remote work into one clear command
per task so the user is not prompted repeatedly.

## Golden Rules for Amarel

- Never run heavy compute on the login node.
- Use Slurm allocations for CPU and GPU work.
- Keep project files and large outputs in scratch storage.
- Show the user the remote commands before launching a job.
- Capture stdout, stderr, Slurm job IDs, and output paths.
- Pull important outputs back to the local machine after the run.

## Standard Remote Task Workflow

For each task, create a local task bundle:

```text
amarel_tasks/<task_id>/
  run.sh
  inputs/
  outputs/
```

The `run.sh` file should:

1. create or enter a scratch project directory;
2. load required modules;
3. create or activate the needed environment;
4. request a Slurm allocation or submit an `sbatch` script;
5. write logs and outputs to a known directory.

Sync the bundle to Amarel:

```bash
rsync -av amarel_tasks/<task_id>/ amarel:/scratch/<NETID>/<task_id>/
```

Run the task remotely:

```bash
ssh amarel 'cd /scratch/<NETID>/<task_id> && bash run.sh'
```

Pull results back:

```bash
rsync -av amarel:/scratch/<NETID>/<task_id>/outputs/ amarel_tasks/<task_id>/outputs/
```

## Interactive CPU Allocation Template

Use this for short CPU work:

```bash
salloc --partition=<partition> --cpus-per-task=<N> --mem=<MEMORY> --time=<HH:MM:SS>
srun --pty bash -l
```

## Interactive GPU Allocation Template

Use this for short GPU work:

```bash
salloc --partition=<gpu_partition> --gres=gpu:1 --cpus-per-task=<N> --mem=<MEMORY> --time=<HH:MM:SS>
srun --pty bash -l
```

## Batch Job Template

Prefer `sbatch` for longer work:

```bash
#!/usr/bin/env bash
#SBATCH --job-name=<job_name>
#SBATCH --output=outputs/%x-%j.out
#SBATCH --error=outputs/%x-%j.err
#SBATCH --time=<HH:MM:SS>
#SBATCH --cpus-per-task=<N>
#SBATCH --mem=<MEMORY>
#SBATCH --partition=<partition>

set -euo pipefail

mkdir -p outputs
module purge
module load <modules>

python <script>.py
```

For GPU work, add the appropriate GPU request:

```bash
#SBATCH --gres=gpu:1
```

Submit and inspect:

```bash
sbatch job.slurm
squeue -u "$USER"
```

## What the Agent Should Show Before Running

Before executing remote work, summarize:

- SSH host;
- local task directory;
- remote scratch directory;
- files to upload;
- exact Slurm resource request;
- command to run;
- expected output files;
- command that will retrieve outputs.

## What the Agent Should Return After Running

After the run, report:

- whether the command completed;
- job ID, if any;
- important stdout or stderr lines;
- local output paths;
- remote output paths;
- next debugging step if the run failed.
