# HPC Batch Script Builder

A single-file, browser-based tool that generates a Windows batch (`.bat`) script for running HPC jobs over PuTTY. Fill in a few fields, and it produces a ready-to-run script that **uploads** your input files from Windows to a Linux HPC, **runs** your commands there, and **retrieves** the results back to Windows.

**Live demo:** https://sensational-syrniki-d4d50d.netlify.app/

No installation, no build step, no external dependencies — it's one self-contained HTML file that runs entirely in the browser. You can also download it and open it offline.

---

## Why

Submitting a job to a remote HPC from a Windows machine usually means repeating the same sequence by hand every time: `pscp` the inputs up, `plink` in to launch the solver, wait, `pscp` the results back, clean up. This tool turns that sequence into a parametrized form so you can regenerate a correct, consistent `.bat` for each job or study without hand-editing paths and escaping.

---

## Features

- **Live preview** — the `.bat` updates as you type, with basic syntax highlighting.
- **File pickers** for both directions (Windows → Linux upload, Linux → Windows download).
- **Run subfolder toggle** — isolates each run in a numbered subfolder (`%Run_Num%`), useful for parallel runs.
- **tmux mode** — launches the job in a detached tmux session and polls until it finishes, so a dropped connection doesn't kill the run.
- **Optional cleanup** of the remote run folder when finished.
- **Extra verbatim commands** appended to the end of the script (e.g. rename / move results).
- **Copy** or **download** the generated script.

---

## Requirements

To *run* the generated script you'll need, on the Windows side:

- [PuTTY](https://www.putty.org/) — specifically `plink.exe` and `pscp.exe`
- A private key in `.ppk` format
- SSH access to your HPC login node

On the HPC side, `tmux` must be available if you use tmux mode.

The generator itself (the web page) needs only a browser.

---

## Usage

1. Open the [live demo](https://sensational-syrniki-d4d50d.netlify.app/) (or the local `index.html`).
2. Fill in the fields:
   - **Connection & PuTTY paths** — SSH user, host, `.ppk` key, and the paths to `plink.exe` / `pscp.exe`.
   - **Copy files Windows → Linux** — pick files or add wildcard patterns (e.g. `*.py`, `*.inp`).
   - **Linux working directory** — the remote working directory, plus an optional run subfolder.
   - **Commands to run on Linux** — one command per line (see the note on scheduler syntax below).
   - **Copy files Linux → Windows** — the result files to pull back and where to save them.
   - **Advanced** — tmux mode, remote cleanup, `pause`, extra commands, and the output filename.
3. Click **Download script**.
4. Put the `.bat` in the same folder as your input files and double-click it (or run it from a terminal).

> The generated script assumes the input files sit **next to the `.bat`** (`%~dp0`).

---

## What the generated script does

The script declares its settings as variables at the top, then runs this pipeline:

1. **Create** the remote working directory (`mkdir -p`).
2. **Upload** the selected input files with `pscp` (Windows → Linux).
3. **Run** your commands on Linux — either inside a detached **tmux** session with a wait loop, or as a single blocking `plink` call.
4. **Download** the result files with `pscp` (Linux → Windows).
5. *(optional)* **Delete** the remote run folder.

Example of the variable block it emits:

```bat
REM --- Paths ---
set "Run_Num=1"
set "REMOTE_WORKDIR=remote_working_directory_path"
set "RUN_DIR=%REMOTE_WORKDIR%/%Run_Num%"
set "LOCAL_DIR=%~dp0"
```

---

## Notes & limitations

**Job-submission syntax varies.** The exact command to submit a job depends on your cluster's scheduler (**LSF**, **PBS**) or a site **wrapper script**, so the commands in the tool are only an example. Refer to your internal / company documentation for the correct submission commands.

**File paths from the picker.** For browser-security reasons, the file picker can only read the **file name**, not the full path. The generated script therefore assumes the selected files are in the same folder as the `.bat`. Use manual entry for wildcard patterns.

**Why tmux?** A tmux session keeps the job alive on the HPC even if your PuTTY/SSH connection drops. The script launches it detached and polls until it finishes, so a network hiccup on the Windows side won't kill the run.

**Foreground assumption.** This script assumes the job runs in the **foreground** (e.g. Abaqus's `interactive` option, or a `:wait` queue) so the session stays occupied until the job completes and the wait logic can detect it. If the job is sent to the **background** (the submit command returns immediately while the solver keeps running under the scheduler), the session won't hold onto it and the wait / cleanup steps won't work — a separate script is needed for that case.

**Run subfolder for parallel runs.** Enabling the run subfolder gives each run its own numbered directory (`%REMOTE_WORKDIR%/%Run_Num%`). This keeps multiple jobs running in parallel from reading and writing the same files at once — each run is isolated by its run number.

---

## Running locally

It's a static page, so there's nothing to build:

```bash
git clone <your-repo-url>
cd <your-repo>
# open index.html in a browser, or serve it:
python -m http.server 8000   # then visit http://localhost:8000
```

## Deployment

Any static host works (Netlify, GitHub Pages, Vercel, S3, …). This repo is deployed on Netlify. For GitHub Pages, enable Pages on the branch/folder containing `index.html`.

---

## Tech

Plain HTML, CSS, and vanilla JavaScript in a single file. No frameworks, no build tooling, no external requests — which is what lets it run fully offline.

---

## License

MIT License
