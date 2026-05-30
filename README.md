# Valency RCN Logistics Control Tower

A Vercel-hosted dashboard for the Valency RCN logistics PCH workbook.
Anyone with the URL sees the latest data — no manual upload, no login.

## How it works

```
   Excel file in your local OneDrive folder
              │ (OneDrive desktop keeps it in sync with the cloud)
              ▼
    you run:  ./scripts/sync.sh
              │
              ├─ copies file ──►  data/pch.xlsx
              └─ git add / commit / push ──►  origin/main
                                                 │
                                                 ▼ webhook
                                          Vercel auto-deploys
                                                 │
                                                 ▼
                                    Browser opens your Vercel URL
                                       fetch /data/pch.xlsx
                                       parse & render
```

That's it. The dashboard is a pure static site (no server, no API, no
database). The Excel workbook is just another static asset, refreshed by
your local sync script and pushed to GitHub.

## Project layout

```
valency-onedrive-sync/
├── index.html               # the dashboard
├── data/
│   └── pch.xlsx             # the latest workbook (created by sync.sh)
├── scripts/
│   ├── sync.sh              # macOS / Linux: local copy + git add/commit/push
│   ├── sync.ps1             # Windows (PowerShell) equivalent of sync.sh
│   ├── sync.env.example     # template — copy to sync.env and edit
│   └── sync.env             # (gitignored) your local config
├── package.json
├── vercel.json
├── .gitignore
├── .gitattributes
└── README.md
```

---

# One-time setup

## 1. Put this folder under git and push to GitHub

```bash
cd valency-onedrive-sync
git init
git add .
git commit -m "init: Valency logistics control tower"
git branch -M main
git remote add origin git@github.com:<you>/valency-logistics-tower.git  # or https URL
git push -u origin main
```

> If the repo already exists on GitHub and is empty, the commands above are
> all you need. If GitHub created an initial commit (README/license), run
> `git pull --rebase origin main` before pushing.

## 2. Connect the repo to Vercel

1. Go to <https://vercel.com/new>.
2. Import the GitHub repo you just pushed.
3. **Framework preset: Other.** Build command: leave empty. Output
   directory: leave empty. (It's a pure static site.)
4. Click **Deploy**.

Vercel will give you a URL like `https://valency-logistics-tower.vercel.app`.
On every push to `main`, Vercel rebuilds and redeploys in ~20–60 seconds.

## 3. Tell the sync script where your OneDrive file lives

```bash
cp scripts/sync.env.example scripts/sync.env
# Open scripts/sync.env in any editor and set PCH_SOURCE to the absolute
# path of the Excel file inside your local OneDrive folder.
```

On macOS the path usually looks like:

```
/Users/<you>/Library/CloudStorage/OneDrive-<Company>/Logistics/PCH.xlsx
```

> Tip: open the file once from the OneDrive folder in Finder so OneDrive
> actually downloads it locally (otherwise it's an "online-only" placeholder
> and the script can't read it).

`scripts/sync.env` is in `.gitignore`, so absolute paths never get committed.

---

# Daily use

Every time a department finishes their edits in the OneDrive Excel and you
want the dashboard to reflect that:

**macOS / Linux:**

```bash
cd valency-onedrive-sync
./scripts/sync.sh
```

**Windows (PowerShell):**

```powershell
cd valency-onedrive-sync
.\scripts\sync.ps1
```

> If PowerShell blocks the script due to execution policy, run it as:
> `powershell -ExecutionPolicy Bypass -File .\scripts\sync.ps1`
> (or `pwsh` instead of `powershell` if you're on PowerShell 7+).

What you'll see:

```
▶ Plan
  Source:       /Users/.../OneDrive-Acme/Logistics/PCH.xlsx  (2.4 MB)
  Destination:  data/pch.xlsx
  Repo:         /Users/.../valency-onedrive-sync
  Branch:       main
  Push:         yes

▶ Copying workbook into repo
  Wrote data/pch.xlsx

▶ Syncing git branch main
  Pulling latest from origin/main (rebase)…

▶ Committing
  data: sync PCH.xlsx (2026-05-30 11:40:01 IST)

▶ Pushing to origin/main

✓ Done. Vercel should redeploy in ~20–60 seconds.
```

Open (or refresh) your Vercel URL — the top banner flips to a green
**"Loaded at HH:MM · N shipments · sheet 'PCH'"**.

### Useful flags

macOS / Linux (`sync.sh`):

| Command                                  | What it does                                                   |
| ---------------------------------------- | -------------------------------------------------------------- |
| `./scripts/sync.sh`                      | Normal run: copy → commit → push to main                       |
| `./scripts/sync.sh /other/path.xlsx`     | One-off override of the source file                            |
| `./scripts/sync.sh --no-push`            | Copy and commit locally, but don't push (useful for review)    |
| `./scripts/sync.sh --dry-run`            | Print the plan and exit. Nothing on disk or git is touched.    |

Windows (`sync.ps1`):

| Command                                                  | What it does                                                   |
| -------------------------------------------------------- | -------------------------------------------------------------- |
| `.\scripts\sync.ps1`                                     | Normal run: copy → commit → push to main                       |
| `.\scripts\sync.ps1 'C:\path\to\other.xlsx'`             | One-off override of the source file                            |
| `.\scripts\sync.ps1 -NoPush`                             | Copy and commit locally, but don't push                        |
| `.\scripts\sync.ps1 -DryRun`                             | Print the plan and exit. Nothing on disk or git is touched.    |

Both scripts read the same `scripts/sync.env` file and behave identically.

The script:

- Refuses files >100 MB (GitHub's hard limit).
- Warns on files >50 MB.
- Rebases your local main onto `origin/main` before committing.
- Skips the commit if the workbook bytes are identical to what's already on
  main (no-op deploys).
- Marks `.xlsx` as binary in `.gitattributes` so git never tries to diff
  or merge it.

---

# Troubleshooting

| Symptom                                                               | What to do                                                                                                                                                                                                |
| --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Banner: `No workbook deployed yet. Run ./scripts/sync.sh…`            | First-time setup. Run the sync script once, wait for Vercel to redeploy, refresh the page.                                                                                                                |
| Banner: `Could not load /data/pch.xlsx (HTTP 404)`                    | Same as above, or the file was deleted. Check the latest Vercel build's "Files" tab to confirm `data/pch.xlsx` is present.                                                                                |
| Script: `Source file not found`                                       | OneDrive may have the file in "online-only" mode. Open it once in Finder/Explorer to force a download, then re-run.                                                                                       |
| Script: `is not a git repository` / `no 'origin' remote configured`   | Run the one-time setup steps in §1.                                                                                                                                                                       |
| Script: `File is XXX MB — exceeds GitHub's 100 MB file limit`         | Either shrink the workbook (remove old sheets) or switch to Git LFS (`brew install git-lfs && git lfs install && git lfs track "*.xlsx"`).                                                                  |
| Vercel build fails                                                    | Open the deployment in the Vercel dashboard and read the build log. There's no build step for this project; failures are almost always missing files or `vercel.json` syntax.                              |
| New data not showing after refresh                                    | Check the Vercel dashboard — is the latest commit deployed yet? Builds take ~20–60s. Hard refresh (⌘⇧R / Ctrl+Shift+R) once it's green.                                                                  |

The dashboard's **Manual upload** button (top right of the banner) is still
fully functional — use it if you ever need to view a workbook that isn't
the one on main (e.g. a prior month's file from your downloads).

---

# Updating the dashboard itself

The dashboard JavaScript, CSS, and HTML all live in a single file:
`index.html`. Edit it, commit, push — Vercel redeploys. The sync layer is
contained in three places inside that file (CSS block, the `syncPanel`
markup at the top of `<body>`, and the `Live workbook layer` JS block). All
other logic — column mapping, charts, Vee Patron chatbot, CSV/PDF exports —
is unchanged from the original v142 file.
