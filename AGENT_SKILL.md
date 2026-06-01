---
name: hermes-arxiv-agent-deploy
description: Use this skill inside a Hermes conversation when a user wants Hermes to deploy hermes-arxiv-agent end to end in either local/Feishu mode or optional GitHub Pages mode, including cloning the appropriate repo, installing Python dependencies, generating the correct cron prompt, and creating a daily cron job.
---

# Hermes Arxiv Agent Deploy

This skill is for deployment and maintenance of `hermes-arxiv-agent` in two deployment modes.

This skill is only meant to be used inside Hermes. Do not add Hermes installation checks or Hermes bootstrap guidance here.

Use it when the user wants any of the following:

- install the project from GitHub
- set up or repair the daily arXiv monitoring cron job
- initialize a new machine for this project
- re-create the cron prompt with the correct local path
- choose between local-only usage and GitHub Pages publishing
- disable PDF download (only abstract summarization)
- deliver results via WeChat instead of Feishu/Lark

The repository defaults to monitoring quantization-related papers. If the user wants a different research topic, update `search_keywords.txt` during deployment.

Do not assume the current local folder name matches the remote repository name. Treat the GitHub repository name `hermes-arxiv-agent` as canonical for clone and deployment instructions.

## Deployment Modes

Choose the mode from the user's installation phrase, not from vague interpretation.

Use these rules:

- if the user explicitly says `按本地/微信模式部署` or `不要配置 GitHub Pages 发布`, use Local / Feishu mode
- if the user explicitly says `按 GitHub Pages publishing 模式部署`, use GitHub Pages mode
- if neither phrase is present, default to Local / Feishu mode

### Mode A: Local / WeChat only

Use this when the user only wants:

- local files and Excel records
- daily Feishu/Lark push
- optional local browser viewing

In this mode:

- cloning the upstream public repository is fine
- fork is not required
- GitHub write access is not required
- the generated cron prompt must come from `cronjob_prompt.txt`
- the cron job must not publish to GitHub Pages

### Mode B: GitHub Pages publishing

Use this only when the user explicitly uses the GitHub Pages publishing installation phrase.

In this mode:

- the user must first fork the repository to their own GitHub account
- the local checkout must point to the user's own fork, not the upstream repository
- SSH is strongly preferred for Git pushes
- the generated cron prompt must come from `cronjob_prompt.pages.txt`
- the cron job must include the static-site publish step
- the cron job must deliver to WeChat, not Feishu/Lark

If the user asks for GitHub Pages mode but has not provided or created a fork, stop and tell them to fork first.

When GitHub Pages mode is requested, the skill URL itself should normally come from the user's own fork, for example:

```text
https://github.com/hui192330-svg/hermes-arxiv-agent/blob/main/AGENT_SKILL.md
```

## Deployment Goal

Bring the user to a working state where:

1. WeChat gateway is configured.
2. The correct repo is cloned locally for the chosen mode.
3. Python dependencies are installed.
4. `cronjob_prompt.generated.txt` exists and points to the real local project directory.
5. The cron prompt is modified to disable PDF download (e.g., by adding `--no-pdf` flag to `monitor.py`).
6. A Hermes cron job exists, points to the real local project directory, and delivers back to the Feishu/Lark chat instead of `local`.

Because this skill runs inside Hermes, Hermes itself is already present by assumption.
If Feishu is not configured, that is a deploy-time prerequisite to surface, not a reason to discuss Hermes installation.

## Required Workflow

Follow this order unless the user explicitly asks for a partial action.

### 1. Verify prerequisites

Check:

- Python 3 is available
- `pip` or `pip3` is available

If the user chose GitHub Pages mode, also check:

- GitHub SSH authentication is available for their fork remote
- the fork repository exists and is writable by the user

If WeChat gateway is not configured, direct the user to run:

```bash
hermes gateway setup --channel wechat
```

The cron job for this repository should be created from a Feishu/Lark Hermes conversation, not from a local CLI-only chat.
For this project, the intended delivery target is WeChat.

When creating or repairing the cron job, ensure its delivery is set to `wechat` rather than `local`.

### 2. Clone or locate the repository

Preferred defaults:

For local / WeChat mode:

```bash
git clone https://github.com/genggng/hermes-arxiv-agent.git
cd hermes-arxiv-agent
```

For GitHub Pages mode:

```bash
git clone git@github.com:hui192330-svg/hermes-arxiv-agent.git
cd hermes-arxiv-agent
```

If the repository already exists locally, reuse it instead of recloning.

If the user chose GitHub Pages mode, ensure the effective push remote points to the user's own fork, not the upstream repository.

If the existing repository uses HTTPS but should push to the user's fork, change it to:

```bash
git remote set-url origin git@github.com:hui192330-svg/hermes-arxiv-agent.git
```

The effective project directory must be captured as an absolute path and reused in later steps. Refer to it as `PROJECT_DIR`.

### 3. Install runtime dependencies

Run inside `PROJECT_DIR`:

```bash
pip install openpyxl requests pdfplumber
```

If the environment uses `pip3`, use that instead.

Also note the repository default search scope:

- the default query in `search_keywords.txt` targets quantization-related LLM papers
- if the user wants another topic, edit `search_keywords.txt` before the first scheduled run

### 4. Run the deployment preparation script

Run this script inside the checked-out repository:

```bash
bash prepare_deploy.sh
```

For GitHub Pages mode, run:

```bash
DEPLOY_MODE=pages bash prepare_deploy.sh
```

The script uses one deployment variable:

- `PROJECT_DIR`
- `DEPLOY_MODE` with valid values `local` or `pages`

If `PROJECT_DIR` is not supplied, the script uses its own directory as the project root. That is the preferred path, because it avoids manual mistakes after clone.

The script is responsible for:

- reading the correct cron prompt template for the chosen mode
- generating `cronjob_prompt.generated.txt` with placeholder paths replaced
- removing the human-only path reminder from `cronjob_prompt.generated.txt`
- keeping the cron prompt aligned with the requirement to rebuild `viewer/papers_data.json` after Excel is updated
- recording the chosen deployment mode in `.deploy_mode`

If the user wants manual override, run:

```bash
PROJECT_DIR=/absolute/path/to/hermes-arxiv-agent DEPLOY_MODE=pages bash prepare_deploy.sh
```

### 5. Modify the cron prompt to disable PDF download (abstract only)

This step is required to satisfy the user's request: no PDF download, only abstract summarization.

After cronjob_prompt.generated.txt is generated, modify it to ensure monitor.py runs without downloading PDFs. The exact method depends on how the project is implemented:

If monitor.py supports a --no-pdf flag (recommended): add that flag to the python command. For example, change:
```text
python monitor.py
```
to

```text
python monitor.py --no-pdf
```
If no flag exists: add a `sed` command before running `monitor.py` to patch the code temporarily, or set an environment variable. Because the cron job runs daily, a clean approach is to modify `monitor.py` permanently. Run:

```bash
cd PROJECT_DIR
# Disable PDF download by commenting out or removing the download logic
sed -i 's/download_pdf=True/download_pdf=False/g' monitor.py
# Or more robust: add a configuration flag
echo "DOWNLOAD_PDF = False" > config_local.py
```

Then adjust the cron prompt to import that config.
To keep the skill generic and safe, we assume the project maintainer has added a `--no-pdf` flag. If not, the skill will apply a `sed` replacement that looks for a common pattern like `self.download_pdf = True` and changes it to `False`.

Implementation in this skill:
After `cronjob_prompt.generated.txt` is generated, run:

```bash
cd PROJECT_DIR
# If --no-pdf is supported, modify the prompt
sed -i 's/python monitor\.py/python monitor.py --no-pdf/g' cronjob_prompt.generated.txt
# If the project lacks the flag, patch the code instead (uncomment next line)
# sed -i 's/download_pdf\s*=\s*True/download_pdf = False/g' monitor.py
```
Then verify that the generated cron prompt no longer triggers PDF downloads.

### 6. Use the generated cron prompt as the cron payload

After step 4:

- the selected template remains the repository source of truth
- `cronjob_prompt.generated.txt` contains the real project path and no longer contains the human-only path-replacement reminder

Use the full current contents of `cronjob_prompt.generated.txt` as the exact `<prompt>` payload for:

```text
/cron add <prompt>
```

Do not rewrite the prompt from memory. Read it from the patched file and use it directly.
This is a Hermes chat slash command, not a bash command.
Do not try to execute `/cron add` through `bash`, `sh`, or `subprocess`.

Verify the generated file now references paths under `PROJECT_DIR`, for example:

- `PROJECT_DIR/new_papers.json`
- `PROJECT_DIR/papers_record.xlsx`
- `PROJECT_DIR/monitor.py --no-pdf`

### 7. Create the cron job

Create the job inside the WeChat Hermes conversation using the standard slash-command form with the exact current contents of `cronjob_prompt.generated.txt`.

Delivery must be `wechat`, so the final cron output is pushed to Feishu/Lark rather than being saved only as `local`.

If the current job was previously created with delivery `local`, recreate it or edit it so the effective delivery target becomes `wechat`.

After creation, confirm:

- prompt contains the real absolute path
- the prompt includes the `--no-pdf` flag (or equivalent) to disable PDF download
- the job is listed in `/cron list`
- the business instructions from `cronjob_prompt.txt` were preserved exactly in `cronjob_prompt.generated.txt` (except the PDF download part)
- delivery is not `local`
- delivery is set to `wechat`

## Agent Behavior Rules

- Prefer automation over asking the user to hand-edit prompt text.
- Do not ask the user to rename their local directory.
- Keep the repository name `hermes-arxiv-agent` in clone instructions and user-facing descriptions.
- If local folder names differ, adapt by substituting the actual absolute path rather than forcing a rename.
- When reconfiguring cron, rerun `prepare_deploy.sh` and then reuse `cronjob_prompt.generated.txt`.
- Prefer `prepare_deploy.sh` over ad hoc manual edits, because it centralizes all known path fixes behind one variable.
- Do not paraphrase or simplify the substantive task instructions from `cronjob_prompt.txt`.
- Treat the selected prompt template as the source of truth and `cronjob_prompt.generated.txt` as the deployable cron payload.
- Treat `/cron add` and `/cron list` as Hermes chat commands, not shell commands.
- Treat WeChat delivery as required for this project; set the cron delivery target to `wechat` and do not leave the job on `local`.
- Keep repository code path handling relative; do not reintroduce machine-specific absolute paths into tracked files.
- In GitHub Pages mode, prefer an SSH Git remote that points to the user's own fork.
- Do not configure scheduled publishing against the upstream public repository unless the user explicitly owns and intends to publish from it.
- Always ensure that no PDF files are downloaded; if the project code tries to download PDFs, patch it during deployment.

## Path Handling Guidance

The remaining deployment-specific path lives in the Hermes cron prompt, not in the tracked Python code.

The correct approach is:

1. Determine `PROJECT_DIR` after clone or discovery.
2. Run `prepare_deploy.sh`.
3. Confirm that `cronjob_prompt.generated.txt` was created with the correct absolute project path.
4. Modify the generated cron prompt to disable PDF download (step 5).
5. Use the generated cron prompt file content directly when creating or updating cron.

If future code changes are allowed, recommend this improvement:

- keep code paths relative to the repository root
- keep deployment-specific absolute paths out of tracked files
- add a `--no-pdf` flag to `monitor.py` to cleanly disable PDF download without patching

## Expected User-Facing Outcome

After successful use of this skill, the user should only need Hermes for normal operations:

- view cron jobs
- rerun the job manually
- update keywords
- inspect generated Excel and viewer output

The user should not need to manually edit repository paths in prompt text.
The cron job will produce daily abstract summaries only (no PDF files) and deliver them to WeChat.

