<!--
SPDX-FileCopyrightText: 2025, 2026 Eric van der Vlist <vdv@dyomedea.com>

SPDX-License-Identifier: GPL-3.0-or-later OR MIT
-->

# MAINTAINER — scion operational notes

This document restores the detailed maintainer guidance that used to live in the longer READMEs. It's intended for people who maintain or evolve the scion (the `.devcontainer/` template) and for repository maintainers who will graft the scion into their projects.

TODO: update this file with the latest updates.

Keep this file with the scion so maintainer guidance travels with the template and doesn't pollute the stock repository.

Summary
- Purpose: explain preflight checks, environment file conventions, updater semantics (.orig/.dist/.bak), dry‑run behavior and common commands.
- Audience: scion maintainers and repository maintainers who accept updates.
- Assumption: the runtime installer (`graft.sh`) honors UPSTREAM_SCION and derived GRAFT_URL; it does not source workspace `.cs_env` automatically.

Contents
- Preflight & .gitignore checks
- Environment files (.devcontainer/.cs_env and ./ .cs_env)
- Updater semantics (.orig, .dist, .bak.*) and interactive choices
- Dry‑run and automation (flags and CI usage)
- Common commands and aliases
- Troubleshooting
- Renaming / migration notes

---

## Preflight & .gitignore checks

Before writing files into a stock repository the installer runs a preflight to detect whether any required template paths would be ignored by the repo's `.gitignore`. This is important because writing files that are then ignored can silently hide essential template content.

Recommended preflight sequence (illustrative)
1. Generate the list of destination paths you intend to create (relative to repo root).
2. Pipe that list to `git check-ignore -v --stdin` to discover any ignores:

```bash
# example: produce a newline-separated list of files/directories to write,
# then check which would be ignored by the target repo.
printf "%s\n" ".devcontainer/" ".vscode/" ".devcontainer/bin/graft.sh" | git check-ignore -v --stdin
```

Behavior
- Non‑dry‑run installs should abort if any required paths are ignored. This prevents accidentally leaving repositories without needed template files.
- Dry‑run should report ignored paths and continue without changing files so maintainers can fix `.gitignore` before committing.

Action for maintainers
- When adding new required files to the scion, update the installer so the preflight list includes the new paths.
- Document any special-case files that may be intentionally ignored and provide guidance on how to materialize them (e.g., copy from `.devcontainer/.cs_env` to `./.cs_env`).

---

## Environment files (.devcontainer/.cs_env and ./ .cs_env)

Rationale
- Many projects use `.env` already; to avoid overwriting or conflicting, scion-specific configuration lives in `.devcontainer/.cs_env` (the sample) and can be copied by repo owners to `./.cs_env` in their repo root when they want to override defaults.

Important conventions
- The scion ships a sample `.devcontainer/.cs_env`. This is not automatically sourced by the installer at runtime.
- The installer reads environment variables (e.g., UPSTREAM_SCION) from the environment. In Codespaces/CI you can export these variables for the session.
- You may include a sample `.devcontainer/.cs_env` with defaults for maintainers to copy, e.g.:
  ```bash
  UPSTREAM_SCION="evlist/codespaces-grafting@stable"
  # Optional hard override:
  # GRAFT_URL="https://raw.githubusercontent.com/evlist/codespaces-grafting/stable/.devcontainer/bin/graft.sh"
  ```

Operational practice
- In Codespaces the environment is commonly configured via repository-level Secrets / Environment — maintainers can set UPSTREAM_SCION there.
- For one-off installs from a workstation, the README shows a copy/paste URL (convenient). For automation, prefer setting UPSTREAM_SCION or GRAFT_URL in CI.

Why we don't auto-source `./.cs_env`
- The script should be usable in minimal environments (CI, fresh clones). Auto-sourcing a workspace file can introduce surprising behavior; instead, we honor already-exported environment variables. Repo maintainers who want to use `./.cs_env` simply export values or run a small wrapper that exports them and calls the installer.

---

## UPSTREAM_SCION and GRAFT_URL (how the installer selects the scion)

- UPSTREAM_SCION format: owner/repo@ref (example: `evlist/codespaces-grafting@stable`)
  - The installer derives a raw URL for the graft installer as:
    `https://raw.githubusercontent.com/<owner/repo>/<ref>/.devcontainer/bin/graft.sh`
- Resolution priority inside the installer:
  1. An already-exported GRAFT_URL (explicit raw URL) — highest precedence
  2. Otherwise derive GRAFT_URL from UPSTREAM_SCION as shown above
  3. Ultimately fall back to the baked-in default UPSTREAM_SCION if none provided

Examples
- Use explicit raw URL:
  ```bash
  export GRAFT_URL="https://raw.githubusercontent.com/evlist/codespaces-grafting/stable/.devcontainer/bin/graft.sh"
  curl -L -o ~/Downloads/graft.sh "$GRAFT_URL"
  ```
- Use UPSTREAM_SCION (installer derives URL):
  ```bash
  export UPSTREAM_SCION="evlist/codespaces-grafting@stable"
  # installer uses derived GRAFT_URL; you can still curl with the derived URL if needed
  ```

---

## Updater semantics and artifact conventions

The installer/updater preserves local edits and uses conservative semantics inspired by package manager config handling.

Conventions
- `.orig` — baseline copy of the file as it was when the scion was first applied. Used as the merge ancestor.
- `.dist` — an upstream sample saved when repository maintainers elect to keep their local version (matches the common ".example" or ".dist" pattern).
- `.bak.<timestamp>` — timestamped backups created before destructive replacements.

High-level rules
- New upstream file (not present locally):
  - Add the file, create a `.orig` baseline next to it.
- Upstream changed, local unmodified (matches `.orig`):
  - Replace the local file with upstream and update `.orig`.
- Local differs from `.orig`:
  - Present interactive choices:
    - Keep local (no change).
    - Replace local (overwrite and create `.bak.<ts>`).
    - Backup + replace (save `.bak.<ts>` then replace).
    - Save upstream sample as `<filename>.dist` (keep local, but keep upstream in a `.dist` copy).
    - Attempt a 3‑way merge using the `.orig` baseline (if toolable).
- Always preserve `.orig` to allow future 3-way merges and to track what was originally installed.

Interactive guidance
- In interactive Codespace runs the script should show a short diff and the options above.
- For non-interactive or CI driven updates, provide flags (e.g., `--yes` to accept a sensible default behavior) and require `--dry-run` in CI to verify what would change before applying.

---

## Dry‑run and automation

Flags and examples
- `--dry-run` — report intended actions and exit without modifying files. Use in CI to detect accidental ignores or policy violations.
- `--ref <branch-or-tag>` — allow selecting a specific upstream ref (branch, tag, or commitish). Defaults to the scion's chosen default (e.g., `stable` or `main`).
- `--yes` — for scripted runs: accept non-destructive defaults where sensible. Use with care.

CI usage
- Recommended pattern:
  1. Run `bash bin/graft.sh --ref <ref> --dry-run` as a pre-flight check in CI.
  2. Inspect results and only run an apply step with `--yes` in controlled automation (or create a pull request for manual review).

Exit codes
- Define and document exit codes in the script (0 success, >0 failure types). Ensure CI treats non-zero exit as failure.

---

## Common commands and aliases

Aliases provided in the Codespace image (convenience for maintainers)
- `cs_install` — alias to `bin/graft.sh` for the initial install (interactive).
- `cs_update` — alias to `bin/graft.sh` for updates (interactive).

Common commands
- Initial install (workstation):
  ```bash
  curl -L -o ~/Downloads/graft.sh https://raw.githubusercontent.com/evlist/codespaces-grafting/main/.devcontainer/bin/graft.sh
  chmod +x ~/Downloads/graft.sh
  cd /path/to/your-repo
  bash ~/Downloads/graft.sh
  ```
- Upgrade interactively (inside Codespace):
  ```bash
  cs_update
  ```
- Dry-run (preview):
  ```bash
  bash bin/graft.sh --dry-run
  ```

---

## Troubleshooting

Common issues and remedies

1. Files are missing after install (likely ignored)
   - Symptom: template files not present after install (or are present locally but not committed).
   - Check:
     ```bash
     printf "%s\n" ".devcontainer/" ".devcontainer/bin/graft.sh" | git check-ignore -v --stdin
     ```
   - Fix: update `.gitignore` or adjust installer preflight; run installer interactively after adjustments.

2. Markdown preview blank in browser (Codespace web UI quirk)
   - Workaround: open in a Chromium-based browser or view README on the GitHub repo page.

3. Codespace returns 401 or missing envs
   - Ensure required environment variables are set in the Codespace (Secrets / Repository settings) or copy `.devcontainer/.cs_env` to `./.cs_env` and export the necessary vars locally.

4. Upstream rename/migration confusion
   - If the scion repo is renamed, set `UPSTREAM_SCION` to the new `owner/repo@ref` in Codespaces/CI or update the template sample `.devcontainer/.cs_env`.
   - The installer derives `GRAFT_URL` from `UPSTREAM_SCION`; no code change needed if env is updated.

---

## Renaming & migration notes

If you rename the scion repository (for example to `evlist/codespaces-grafting`) follow these steps:
1. Update the template sample: `.devcontainer/.cs_env` to point to the new `owner/repo@ref`.
2. Update README examples (the explicit raw URL examples) to use the new `raw.githubusercontent.com` location for convenience.
3. Encourage repository maintainers to set `UPSTREAM_SCION` in the Codespace or CI environment to the new `owner/repo@ref` so no immediate code changes are required.
4. Optionally, update the baked-in default `UPSTREAM_SCION` in `bin/graft.sh` to the new value.

Note: GitHub creates redirects for renamed repositories, so old raw URLs may continue to work for a while. However, for long-term clarity update explicit URLs in documentation.

---

## Appendix: sample .devcontainer/.cs_env (recommended)

Add this sample to the scion to help maintainers override defaults easily:

```bash
# .devcontainer/.cs_env (sample)
# Format: owner/repo@ref
UPSTREAM_SCION="evlist/codespaces-grafting@stable"
# Optional explicit override:
# GRAFT_URL="https://raw.githubusercontent.com/evlist/codespaces-grafting/stable/.devcontainer/bin/graft.sh"
```

---

If you want, I can:
- Produce a ready-to-commit PR that adds this `MAINTAINER.md` into `.devcontainer/docs/`.
- Or produce a smaller `MAINTAINER.md` if you prefer a shorter checklist-only document.

Which would you like next?