<!--
SPDX-FileCopyrightText: 2025, 2026 Eric van der Vlist <vdv@dyomedea.com>

SPDX-License-Identifier: GPL-3.0-or-later OR MIT
-->

<img src="./assets/icon.svg" alt="cs-grafting logo" title="cs-grafting" width="256" style="float:right;max-width:256px;height:auto" />

# .devcontainer — scion maintainer notes

🌱 Quick metaphor (attention)  
This scion is to a repository what a plant scion is to a stock: graft the scion (this Codespace/devcontainer template) into a repo to give it an instant, reproducible development environment.

What’s in the scion (high level)
- devcontainer.json, Dockerfile, docker-compose.yml — container & Codespaces config
- bin/graft.sh — installer/updater script (graft)
- .vscode/ — editor configuration, snippets and stubs (template-managed)
- docs/ — scion-maintainer docs (upgrade guides, internals)
- assets/ — icons and images used in the scion docs

Scion structure (short)
```
.devcontainer/
├── README.md             # this file (short)
├── docs/                 # extended scion docs (upgrade guides, internals)
├── assets/               # icons, images used in scion docs
├── devcontainer.json
├── Dockerfile
├── .cs_env               # base environment variables
├── .cs_env.d/            # additional env files (Debian .d style)
├── bin/
│   └── graft.sh          # run this to graft the scion into a repo
├── sbin/
│   ├── bootstrap.sh      # container startup script
│   ├── bootstrap.sh.d/   # bootstrap hooks (sourced in order)
│   └── merge-env.sh      # merges .cs_env and .cs_env.d/*
├── tmp/                  # temporary files (gitignored)
├── var/                  # runtime data (gitignored)
└── wp-content/           # WordPress content customizations
```

🔁 Upgrade & maintainer quick guide
- Interactive (inside a Codespace) — recommended:
  - `cs_install` — initial install (alias to bin/graft.sh)
  - `cs_update`  — interactive update (alias to bin/graft.sh)
- From a workstation:
  ```bash
  curl -L -o ~/Downloads/graft.sh \
    https://raw.githubusercontent.com/evlist/codespaces-grafting/stable/.devcontainer/bin/graft.sh
  chmod +x ~/Downloads/graft.sh
  cd /path/to/your-repo
  bash ~/Downloads/graft.sh      # or: bash ~/Downloads/graft.sh --dry-run
  ```
- Dry-run recommended: `bash bin/graft.sh --dry-run`

Update semantics (summary)
- New scion file → added and scion (previous) saved as `.orig`.
- Local edits preserved when they differ from `.orig` (interactive choices: keep, replace, backup+replace, save scion sample as `.dist`, or 3‑way merge).
- Scion samples saved as `.dist` when keeping local changes.
- Like Debian's dpkg: `.orig` = previous version, new version compared against local edits.

Troubleshooting tips
- If `git check-ignore` shows required paths are ignored, fix `.gitignore` or run interactively — non‑dry‑run installs abort to avoid hiding scion files.
- Use `.devcontainer/docs/` for step‑by‑step maintainer procedures and to record merging/upgrading policies.

Notes on naming and scope
- Display name: `codespaces-grafting` (short alias: `cs-grafting`). Current implementation and examples target WordPress, but the pattern is generic and reusable for other repositories.

## License

This project is dual-licensed:

- GPL-3.0-or-later OR
- MIT

You may choose either license. See the LICENSE file and LICENSES/ directory for full texts.