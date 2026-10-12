# Data Science Setup - msi-cachy

## uv, venv setup

```
mkdir -p ~/Workspace/ds-env
cd ~/Workspace/ds-env

# Initialize project pinned to Python 3.12
uv init --python 3.12

# Create the dedicated local virtual environment
uv venv
```

## installing required packages and configuring
### Data manipulation, mathematics, and plotting
```
uv add numpy pandas polars scipy scikit-learn matplotlib seaborn statsmodels
```
### Jupyter kernel interfaces, parquet/excel formats, and utilities
```
uv add jupyterlab ipykernel ipywidgets tqdm rich requests openpyxl pyarrow
```
### Install GPU-Accelerated PyTorch (CUDA 12)
```
uv add torch torchvision \
  --index torch=[https://download.pytorch.org/whl/cu124](https://download.pytorch.org/whl/cu124) \
  --index-strategy unsafe-best-match
```
### Register Jupyter Kernel
~/Workspace/ds-env/.venv/bin/python -m ipykernel install --user --name=ds-env --display-name="Python (ds-env)"

### Headless JupyterLab Brave PWA Setup
```
mkdir -p ~/.config/systemd/user

string join \n \
'[Unit]' \
'Description=JupyterLab Workspace Server' \
'After=network.target' \
'' \
'[Service]' \
'Type=simple' \
'ExecStart=%h/Workspace/ds-env/.venv/bin/jupyter-lab --no-browser --ServerApp.token=\'\' --ServerApp.password=\'\' --ServerApp.ip=\'127.0.0.1\' --ServerApp.port=8888' \
'WorkingDirectory=%h/Workspace' \
'Restart=always' \
'RestartSec=5' \
'' \
'[Install]' \
'WantedBy=default.target' > ~/.config/systemd/user/jupyterlab.service
```
```
systemctl --user daemon-reload
systemctl --user enable --now jupyterlab.service
```
