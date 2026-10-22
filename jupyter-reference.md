# Jupyter Reference (CachyOS, fish, uv)

Personal reference for the headless JupyterLab setup: Python + R kernels, token auth, systemd user service, Brave PWA, VS Code. Commands are written for **fish**. Where bash/zsh differ, it is noted.

---

## 1. Layout at a glance

| Thing | Location |
|---|---|
| Jupyter project (uv) | `~/Workspace/ds-env/jupyter-workspace` |
| Project files | `pyproject.toml`, `uv.lock`, `.python-version`, `.venv/` |
| Python | uv-managed 3.12 (pinned in `.python-version`) |
| Service file | `~/.config/systemd/user/jupyterlab.service` |
| Token | `~/.config/jupyterlab/token` (chmod 600) |
| Token as env file | `~/.config/jupyterlab/env` (chmod 600, contains `JUPYTER_TOKEN=...`) |
| Python kernel spec | `~/.local/share/jupyter/kernels/ds-env` ("Python (ds-env)") |
| R kernel spec | `~/.local/share/jupyter/kernels/ir` |
| R user library | `~/.local/share/R/library` (set via `R_LIBS_USER` in `~/.Renviron`) |
| URL | `http://127.0.0.1:8888/lab` |

`~/Workspace/ds-env` is a plain parent folder. Only `jupyter-workspace` is a uv project (created with `--bare --no-workspace`, so it never attaches to a parent project).

---

## 2. How the pieces fit (the mental model)

- **uv** creates `.venv` inside the project and installs packages into it. `uv run <cmd>` runs a command inside that environment without activating it.
- **JupyterLab** is a server. The systemd service launches `.venv/bin/jupyter-lab` directly, so the venv's packages are what the server sees.
- **A kernel** is a separate process that actually runs your code. Jupyter finds kernels by reading **kernelspec** folders (a `kernel.json` that says "start this program"). `ds-env` points at the venv's Python. `ir` points at R.
- **Registering a kernel** just writes a kernelspec folder. It doesn't install anything, which is why VS Code can use the same kernels without the server running.
- **Language servers (LSP)** are separate helper programs (`python-lsp-server`, R `languageserver`) that provide hover docs, signatures, and diagnostics. `jupyterlab-lsp` connects them to the editor.
- **systemd user service** keeps the server running in the background under your account. It starts at login (or at boot with linger).
- **Auth:** Jupyter reads the `JUPYTER_TOKEN` environment variable. The service loads it from the env file, so the token never appears in the command line or process list.

---

## 3. Daily commands

### Service control

```fish
systemctl --user status jupyterlab.service
systemctl --user restart jupyterlab.service     # after adding packages
systemctl --user stop jupyterlab.service
systemctl --user start jupyterlab.service
systemctl --user disable --now jupyterlab.service   # stop and don't start at login
systemctl --user enable --now jupyterlab.service    # the reverse
journalctl --user -u jupyterlab.service -e          # recent logs
journalctl --user -u jupyterlab.service -f          # follow logs live
```

### Print the login URL (fish)

```fish
echo "http://127.0.0.1:8888/lab?token=$(cat ~/.config/jupyterlab/token)"
```

In fish, `(cmd)` inside double quotes is **not** expanded. Use `$(cmd)` inside quotes, or close the quotes first: `"...token="(cat file)`. This caused an empty output earlier.

In bash/zsh, `$(cmd)` works the same way.

### Just the token

```fish
cat ~/.config/jupyterlab/token
```

---

## 4. Managing packages

### Python (into the Jupyter environment)

```fish
cd ~/Workspace/ds-env/jupyter-workspace
uv add <package>                    # add
uv add --dev <package>              # dev-only tools
uv remove <package>                 # remove
uv sync                             # make .venv match uv.lock
uv lock --upgrade && uv sync        # upgrade everything within constraints
uv tree                             # dependency tree
systemctl --user restart jupyterlab.service
```

A running kernel doesn't see new packages until you **restart the kernel** (Kernel menu, Restart Kernel).

### R

Install from inside R, not with uv:

```fish
R
```
```r
install.packages("tidyverse")       # compiles from source, takes a while
q()
```

Packages land in `~/.local/share/R/library`. For per-project R package versions later, look at `renv`.

### Currently installed in this project

`jupyterlab`, `ipykernel`, `ipywidgets`, `pandas`, `numpy`, `matplotlib`, `seaborn`, `scikit-learn`, `tqdm`, `rich`, `requests`, `openpyxl`, `pyarrow`, `jupyterlab-lsp`, `python-lsp-server`.

Optional extras that were suggested: `python-lsp-ruff` (lint diagnostics) and `rpy2` (R from Python cells with `%load_ext rpy2.ipython`).

R side: `IRkernel`, `languageserver`.

---

## 5. Kernels

```fish
cd ~/Workspace/ds-env/jupyter-workspace
uv run jupyter kernelspec list                   # what's registered
uv run jupyter kernelspec remove python3         # hide the duplicate built-in tile
uv run jupyter kernelspec remove <name>          # remove any kernel
```

### Re-register the Python kernel

```fish
cd ~/Workspace/ds-env/jupyter-workspace
uv run python -m ipykernel install --user --name=ds-env --display-name="Python (ds-env)"
```

Safe to repeat. It overwrites the old entry.

### Re-register the R kernel

```fish
cd ~/Workspace/ds-env/jupyter-workspace
uv run R -e 'IRkernel::installspec(user = TRUE)'
```

### Kernel for a separate project

Each project keeps its own `.venv`. To use it in Jupyter or VS Code:

```fish
cd ~/Workspace/<project>
uv add --dev ipykernel
uv run python -m ipykernel install --user --name=<project> --display-name="Python (<project>)"
```

Rule of thumb: the shared `ds-env` kernel is for quick exploration. Real projects get their own kernel so their packages stay isolated.

---

## 6. Using the same kernels in VS Code

1. Install the **Python** and **Jupyter** extensions (and **R** by REditorSupport for R files).
2. Open an `.ipynb`, click **Select Kernel**, choose **Jupyter Kernel...**, then **Python (ds-env)** or **R**.
3. This doesn't need the server running, because VS Code reads the same kernelspec folder.

Optional, to share live kernels with the browser: **Select Kernel**, **Existing Jupyter Server...**, then enter `http://127.0.0.1:8888/?token=<token>`. This needs the service running.

If the kernels are missing: reload the window (Ctrl+Shift+P, "Developer: Reload Window") and confirm `uv run jupyter kernelspec list` shows them. The open-source `code` build uses Open VSX, where Microsoft's extensions can be missing. Microsoft's build avoids this.

R in VS Code: if it can't find R, set `r.rpath.linux` to `/usr/bin/R`.

---

## 7. Security notes

### What is set up

- The server binds to **127.0.0.1** only, and `allow_remote_access=False`. Nothing on your network can reach it.
- Login requires the **token**. Opening `?token=...` once sets a cookie for that browser.
- The token lives in a `chmod 600` file, not in the service file or command line.
- Service hardening: `NoNewPrivileges=true`, `PrivateTmp=true`, `port_retries=0` (fails instead of silently switching ports).

### How to test that auth is working

Open a private browser window and go to `http://127.0.0.1:8888/lab` with no token. It should show a login page. Or use curl:

```fish
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8888/api/status
curl -s -o /dev/null -w "%{http_code}\n" "http://127.0.0.1:8888/api/status?token=wrong"
curl -s -o /dev/null -w "%{http_code}\n" "http://127.0.0.1:8888/api/status?token="(cat ~/.config/jupyterlab/token)
```

Expected: `403`, `403`, `200`.

### Why the old PWA never asks for the token

A browser or PWA that logged in once keeps a valid cookie, and the PWA shares Brave's cookies for `127.0.0.1`. This is expected, not a hole. To force a fresh login, clear the `127.0.0.1` entry at `brave://settings/content/all`, or reinstall the PWA from `http://127.0.0.1:8888/lab` with no token in the URL. Deleting `jupyter_cookie_secret` also works, but only if you delete the file Jupyter actually uses. Find it with:

```fish
uv run jupyter --paths
find ~/.local/share/jupyter /run/user/(id -u)/jupyter -name jupyter_cookie_secret 2>/dev/null
```

### Rotate the token

Do this if the URL leaks (screenshots, chats, Git):

```fish
openssl rand -hex 32 > ~/.config/jupyterlab/token
chmod 600 ~/.config/jupyterlab/token
printf 'JUPYTER_TOKEN=%s\n' (cat ~/.config/jupyterlab/token) > ~/.config/jupyterlab/env
chmod 600 ~/.config/jupyterlab/env
systemctl --user restart jupyterlab.service
```

### Honest limits

The token protects against other users, browser tabs, and network access. It does **not** protect against malware already running as your user, since that process could read the token file anyway. Jupyter also gives full shell access as you, so never expose it beyond localhost.

If you ever need access from another device, use an SSH tunnel or Tailscale. Never change the bind to `0.0.0.0` without HTTPS and a firewall rule.

---

## 8. The service file (current version)

`~/.config/systemd/user/jupyterlab.service`:

```ini
[Unit]
Description=JupyterLab Workspace Server
After=network.target

[Service]
Type=simple
EnvironmentFile=%h/.config/jupyterlab/env
Environment=PATH=%h/Workspace/ds-env/jupyter-workspace/.venv/bin:/usr/local/bin:/usr/bin
WorkingDirectory=%h/Workspace/ds-env/jupyter-workspace
ExecStart=%h/Workspace/ds-env/jupyter-workspace/.venv/bin/jupyter-lab --no-browser --ServerApp.ip=127.0.0.1 --ServerApp.port=8888 --ServerApp.port_retries=0 --ServerApp.allow_remote_access=False
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=default.target
```

- `%h` means your home directory.
- The `PATH` line lets the server find the venv's tools and `R`.
- `WorkingDirectory` decides which folder the Jupyter file browser opens in. To browse all of `~/Workspace` instead, change only that line to `%h/Workspace`.
- After any edit: `systemctl --user daemon-reload` then `systemctl --user restart jupyterlab.service`.
- Start at boot without logging in: `loginctl enable-linger marvin`.

---

## 9. Troubleshooting

| Symptom | Check / fix |
|---|---|
| Service `failed` | `journalctl --user -u jupyterlab.service -e` |
| Port 8888 in use | `ss -ltnp \| grep 8888` finds the process. Stop the old server, then restart the service |
| Empty `echo` of the URL (fish) | Use `$(cat ...)` inside quotes |
| Kernel dies or "No such file" | The `.venv` is broken or was rebuilt. `cd` into the project, run `uv sync`, then re-register the kernel |
| Package imports fail in a notebook | Wrong kernel selected, or kernel not restarted after `uv add` |
| `python3.12` not found | `fish_add_path ~/.local/bin` (uv's links live there) |
| R completions missing | `R -e 'library(languageserver)'` should run without error. Reinstall the package if it errors |
| LSP indicator red | Settings, Language Servers, check the error. Confirm `python-lsp-server` is installed in the project |
| Duplicate "python3" tile | `uv run jupyter kernelspec remove python3` (may return after `uv sync`) |
| R package fails to compile | The error names a missing library. Install it with pacman and retry |
| Token rejected after rotating | Restart the service so the new env file is loaded |
| uv warns `~/.local/bin` not on PATH | `fish_add_path ~/.local/bin` |

Past typos worth remembering: `uv python pin 3.12v` (stray `v`) and `which python 3.12` (space). Both silently do the wrong thing.

---

## 10. Rebuild from scratch

If everything breaks, this recreates the setup (fish syntax):

```fish
# 0. Stop and clean
systemctl --user disable --now jupyterlab.service
rm -rf ~/Workspace/ds-env/jupyter-workspace

# 1. Project
mkdir -p ~/Workspace/ds-env/jupyter-workspace
cd ~/Workspace/ds-env/jupyter-workspace
uv init --bare --no-workspace
uv python pin 3.12
uv add jupyterlab ipykernel ipywidgets pandas numpy matplotlib seaborn scikit-learn tqdm rich requests openpyxl pyarrow jupyterlab-lsp python-lsp-server

# 2. Kernels
uv run python -m ipykernel install --user --name=ds-env --display-name="Python (ds-env)"
uv run R -e 'IRkernel::installspec(user = TRUE)'

# 3. Token (skip if the files still exist)
mkdir -p ~/.config/jupyterlab
openssl rand -hex 32 > ~/.config/jupyterlab/token
chmod 600 ~/.config/jupyterlab/token
printf 'JUPYTER_TOKEN=%s\n' (cat ~/.config/jupyterlab/token) > ~/.config/jupyterlab/env
chmod 600 ~/.config/jupyterlab/env

# 4. Service: recreate the file from section 8, then:
systemctl --user daemon-reload
systemctl --user enable --now jupyterlab.service
```

R prerequisites if R was removed:

```fish
sudo pacman -S --needed r gcc-fortran base-devel
mkdir -p ~/.local/share/R/library
echo 'R_LIBS_USER=${HOME}/.local/share/R/library' >> ~/.Renviron
R -e 'install.packages(c("IRkernel", "languageserver"), repos = "https://cloud.r-project.org")'
```

---

## 11. Complete removal

```fish
systemctl --user disable --now jupyterlab.service
rm -f ~/.config/systemd/user/jupyterlab.service
systemctl --user daemon-reload
rm -rf ~/.config/jupyterlab
rm -rf ~/Workspace/ds-env/jupyter-workspace
rm -rf ~/.local/share/jupyter/kernels/ds-env ~/.local/share/jupyter/kernels/ir
```

Optional: `~/.local/share/R/library` (R packages) and `~/.Renviron`. Uninstall the Brave PWA from `brave://apps`.

---

## 12. Educational extras

### Notebook shortcuts (JupyterLab and VS Code)

| Key | Action |
|---|---|
| Shift+Enter | Run cell and move to next |
| Ctrl+Enter | Run cell and stay |
| Esc / Enter | Command mode / edit mode |
| A / B | Insert cell above / below (command mode) |
| D, D | Delete cell |
| M / Y | Convert to Markdown / code |
| Tab | Completion |
| Shift+Tab | Signature and docs popup |
| 0, 0 | Restart kernel (command mode) |

### Useful magics (Python kernel)

```python
%load_ext autoreload      # reload edited modules automatically
%autoreload 2
%matplotlib inline        # plots appear below cells
%timeit sum(range(1000))  # time a single line
%%time                    # time a whole cell (first line of the cell)
%who                      # list variables
!ls -la                   # run a shell command
```

### Notebooks and Git

Notebooks store outputs and metadata, which makes noisy diffs. Two helpers:

```fish
uv add --dev nbstripout
nbstripout --install                 # inside a git repo: strips outputs on commit

uv add --dev jupytext
jupytext --set-formats ipynb,py:percent notebook.ipynb   # pairs the notebook with a clean .py
```

Commit the `.py` pair for readable diffs. Always add `.venv/` and `.ipynb_checkpoints/` to `.gitignore`.

### Exporting and running

```fish
uv run jupyter nbconvert --to html notebook.ipynb
uv run jupyter nbconvert --to script notebook.ipynb
uv run jupyter nbconvert --to notebook --execute notebook.ipynb   # run top to bottom
```

Running a notebook top to bottom (Kernel, Restart Kernel and Run All) before sharing is the best test that it's reproducible. Hidden state from out-of-order cells is the most common notebook bug.

### Quick experiments without touching the project

```fish
uv run --with seaborn --with statsmodels jupyter lab   # temporary extra packages
uvx jupyterlab                                         # throwaway, no project at all
```

### Good habits

- Keep one environment per project, and use `ds-env` only as a scratchpad.
- Put imports and constants in the first cell.
- Restart and run all before trusting results.
- Commit `pyproject.toml` and `uv.lock`, never `.venv/`.
- Treat the token like a password. Rotate it if it leaks.
