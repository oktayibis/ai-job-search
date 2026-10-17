# Local web console

A browser front end over the framework's files and slash commands. It adds no workflow logic of its own: tables read and edit the same files the commands use, and every action button runs the existing slash command headlessly through `claude -p`.

```bash
python3 ui/server.py          # http://127.0.0.1:8765, standard library only
```

Requirements: Python 3.10+, the `claude` CLI on `PATH` and logged in, and [bun](https://bun.sh) for the portal search CLIs.

## Profiles: several job searches side by side

One console can run independent job searches for several people, for example one per family member. Each search is a **profile**: a plain folder (no git) that the console creates and manages.

```
~/JobSearch/                       # JOBUI_HOME
├── profiles.json                  # registry: slug, name, path, settings
└── profiles/
    ├── anna-berg/                 # one self-contained workspace per person
    │   ├── CLAUDE.md, .claude/, .agents/, tools/, cv/, cover_letters/ ...   # framework copy
    │   ├── documents/             # uploaded CV, LinkedIn PDF, diplomas, references, setup_answers.md
    │   ├── job_search_tracker.csv
    │   ├── job_scraper/seen_jobs.json
    │   └── ui/                    # this profile's runs, viewed-jobs state, role briefs
    └── ...
```

### How isolation works

- **One folder per person.** Every Claude run starts with its working directory set to one profile's folder, so `CLAUDE.md`, the skills, the tracker and the documents it sees all belong to that person. Profiles live outside the app checkout, so a run in one profile cannot reach another profile's files.
- **Separate memory.** Claude Code keeps project memory per folder path, so each profile also gets its own memory.
- **A copy of the framework, not links.** A new profile gets the framework files as they are in the app's `master` commit (`JOBUI_TEMPLATE_REF`), read from git rather than the working tree, so the app checkout's own data never leaks into a profile. The files are copied, not symlinked, because `tools/*.py` locate the repo root from their own path. Each portal CLI gets its own `bun install` (about 4 s).
- **Per-profile locks.** Runs, file locks and "one setup at a time" are scoped per profile, so a scrape for one person never blocks another.
- **Shared accounts.** Connectors such as Gmail belong to the Claude account, not to a profile, so every profile would see the same mailbox. Gmail sync is therefore **off** for new profiles and switched on per profile on the Profile page.

### Setting up a new profile

1. Sidebar → profile switcher → **New job search** → name.
2. The setup form, with a draft saved as you type:
   1. Identity
   2. CV and documents (drag-and-drop PDF/TeX/TXT/MD up to 15 MB, or paste the CV text)
   3. Languages and CV language
   4. Market, city and commute radii, plus target roles, skills and sectors (optional: left blank, Claude derives them from the CV and LinkedIn and lists them in its summary for review)
   5. Goals, deal-breakers and optional salary and behavioral answers
   6. Review
3. **Start setup**. The server writes the answers to `documents/setup_answers.md` and runs `/setup`:
   - It uses the uploaded documents (Path A), or the pasted CV (Path B).
   - It takes every answer from the file instead of asking, and applies additive changes without confirmation.
   - It asks in the chat only about genuine conflicts (a document contradicting an answer, or two documents disagreeing).
   - It also fills the CV and cover-letter contact blocks.
4. When the run finishes, the Profile page shows the setup checklist:
   - **Ready** means no template placeholders are left in the profile files.
   - **Partial** lists the files that still need work and offers **Finish setup**.

The Profile page also has:
- re-running only the search configuration (`/setup --section search`)
- `/expand`
- portal on/off switches
- the Gmail switch
- a `.zip` backup (profiles have no git history)

### Why profile files are written through a staging folder

Headless `claude -p` write-protects any path inside a `.claude/` folder, and permission allow rules don't lift that protection. The profile files `/setup` and `/apply` personalize live in `.claude/skills/`, so the console tells each run to write them to `ui/staging/<path relative to .claude/>`. When the run ends, the server installs them.

- Only known profile files are installed (`CLAUDE.md` and the personal skill files). Anything else in staging is ignored.
- Each replaced file is backed up first, under `ui/backups/<run id>/`.

### Updating the framework

After you pull a newer app version, **Update framework** on the Profile page copies the framework files into the profile:
- Personal files are never overwritten: `CLAUDE.md`, the job-application-assistant profile files, `search-queries.md`, the example CV and cover letter, `documents/`, `job_scraper/` and generated output.
- A portal's `enabled:` choice is kept.
- When the profile's methodology files carry an older `framework_version`, the page offers a Claude **merge** run. It takes the new methodology and keeps the personal content.

### Existing personalized checkouts

A checkout personalized before profiles existed is registered as an **external** profile in place, named from its `CLAUDE.md` heading. Nothing is copied or moved, and it keeps working as before alongside the new profiles.

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `JOBUI_HOME` | `~/JobSearch` | Where `profiles.json` and the profile folders live |
| `JOBUI_TEMPLATE_REF` | `master` | Git ref new profiles and framework updates are copied from |
| `JOBUI_AUTO_BRIEF` | `1` | `0` turns off the role briefs written after a scrape |
| `CLAUDE_UI_ARGS` | `--permission-mode acceptEdits --allowedTools ...` | Extra flags for every `claude -p` run |

## Security notes

- Binds to `127.0.0.1` only, with Host/Origin checks against DNS rebinding and cross-site requests.
- Profile slugs are validated. Uploads go only to `documents/{cv,linkedin,diplomas,references,projects}`, with an extension allow-list, a size cap, safe file names and no overwrites.
- File serving is confined to the active profile's `cv/`, `cover_letters/` and `documents/`.

## Tests

```bash
python3 -m unittest tests.test_ui_server tests.test_ui_profiles
```

`tests/test_ui_profiles.py` builds a throwaway git repo as the app and uses a fake `claude` that records its working directory and arguments. It checks the following:
- profiles never share data
- runs start in their own profile
- framework updates keep personal files
- uploads stay confined
- staged files install with a backup
