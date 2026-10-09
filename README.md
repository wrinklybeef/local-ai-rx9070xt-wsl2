# Local LLMs on an RX 9070 XT with WSL2

I wanted a local chat and coding setup using the GPU already in my Windows PC.
This is how I got ROCm, Ollama and Open WebUI working together in WSL2, including
the missing runtime, service configuration and memory problems I ran into.

The setup worked in June 2026. The commands preserve that configuration, with
version caveats and suggested changes called out below.

## Hardware and software

| Component | My setup |
|---|---|
| CPU | Intel Core i7-14700K |
| GPU | AMD Radeon RX 9070 XT, 16 GB VRAM, `gfx1201` |
| Host | Windows 11 |
| Linux | Ubuntu 26.04 under WSL2, with systemd |
| ROCm | Recorded as 7.2.4 from Ubuntu's archive; package version needs confirmation |
| WSL GPU bridge | librocdxg, built from source; commit not recorded |
| Ollama | Linux systemd service, with the v0.30.10 ROCm archive |
| Web UI | Open WebUI's `main` image, using Docker Engine inside Ubuntu |

My notes did not capture the exact Windows/WSL builds, installed AMD driver,
Ollama binary version, Docker version or Open WebUI image digest. In particular,
[Ubuntu's current `rocm` metapackage](https://packages.ubuntu.com/resolute/rocm)
is listed as 7.1.0-0ubuntu6, not 7.2.4. The system packages and Ollama's bundled
ROCm runtime are separate; the original version claim still needs checking on
the machine with `apt-cache policy rocm` and `dpkg-query -W rocm rocminfo`.

## Architecture

```text
Windows browser: http://localhost:3000
    |
    +-- WSL2 / Ubuntu
          +-- Docker: Open WebUI (3000 -> 8080)
          |     -> host.docker.internal:11434
          +-- Ollama systemd service
                -> ROCm -> librocdxg -> /dev/dxg -> RX 9070 XT
```

Inference runs locally. Installing software and downloading models still require
network access. Docker Desktop and a Windows Ollama installation are not part of
this setup.

## Setup

### 1. Check WSL and Windows prerequisites

In Windows PowerShell:

```powershell
wsl --list --verbose
```

The Ubuntu distribution must show version `2`. If it shows `1`, convert it with
`wsl --set-version Ubuntu 2`, replacing `Ubuntu` with its name from the listing.
Inside Ubuntu, `ps -p 1 -o comm=` should report `systemd`. If needed, follow
[Microsoft's systemd instructions](https://learn.microsoft.com/en-us/windows/wsl/systemd)
(WSL 0.67.6 or newer).

Install a compatible AMD Windows driver and the
[Windows SDK](https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/).
My notes specify Adrenalin 26.2.2 or later. AMD documents the ROCDXG introduction
with [Adrenalin 26.2.2 and ROCm 7.2.1](https://rocm.docs.amd.com/projects/radeon-ryzen/en/latest/index.html);
that does not establish compatibility with every later driver/runtime combination.
The SDK path below uses `10.0.28000.0`; substitute the version actually installed.

### 2. Install ROCm and build librocdxg

Run the remaining commands in Ubuntu unless marked otherwise. The original
package installation was:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y rocm git cmake build-essential zstd
```

`upgrade` updates the whole distribution. The `rocm` package is in Ubuntu's
`universe` repository. You also need `curl` for the downloads below; install it
with `sudo apt install curl` if absent.

At this point, `rocminfo` failed on my machine:

```text
hsa api call failure at: ./rocminfo.cc:1324
Call returned HSA_STATUS_ERROR_OUT_OF_RESOURCES
```

I had no `/dev/kfd` or `/dev/dri/`. This setup uses WSL's `/dev/dxg` through
librocdxg rather than the native Linux AMD kernel driver. That error alone does
not identify the cause on every machine.

I built the bridge in my Linux home directory. A clone under `/mnt/c` had failed
with `core.filemode` errors; moving it avoided the Windows filesystem issue.

```bash
cd ~
git clone https://github.com/ROCm/librocdxg.git
cd librocdxg && mkdir -p build && cd build
export win_sdk="/mnt/c/Program Files (x86)/Windows Kits/10/Include/10.0.28000.0"
cmake .. -DWIN_SDK="${win_sdk}/shared"
make && sudo make install
HSA_ENABLE_DXG_DETECTION=1 rocminfo
```

`rocminfo` then listed both the CPU and RX 9070 XT as HSA agents.

**Version caveat:** the [standalone librocdxg repository](https://github.com/ROCm/librocdxg)
is now deprecated and points to `rocm-systems`. It documents CMake 3.15+ and
GCC 11.4+, and a version compatibility matrix. The unpinned clone above is what
I used, not a reproducible source revision. Check that matrix before combining
releases; newer runtimes also change whether `HSA_ENABLE_DXG_DETECTION` is needed.

### 3. Install Ollama and its ROCm runtime

My original command was `curl -fsSL https://ollama.com/install.sh | sh`.
For reproduction, download and inspect the privileged installer, then use
[Ollama's version option](https://docs.ollama.com/linux) to match the v0.30.10
runtime below. This is a documented adjustment, not a rerun of my installation:

```bash
curl -fsSL https://ollama.com/install.sh -o install.sh
less install.sh
OLLAMA_VERSION=0.30.10 sh install.sh
```

It creates an `ollama` user and systemd service. With the default configuration,
models are stored in `/usr/share/ollama/.ollama/models`.

The version option pins the binary, not the downloaded installer script. Do not
mix a current binary with this older runtime archive.

My service ran on the CPU despite `rocminfo` seeing the GPU, even as the `ollama`
user. The log contained:

```text
id=cpu library=cpu compute="" total_vram="0 B"
```

`ls /usr/local/lib/ollama/` showed only `cuda_v12`, `cuda_v13` and `vulkan`.
The [v0.30.10 installer source](https://github.com/ollama/ollama/blob/v0.30.10/scripts/install.sh)
exits its WSL2 branch before reaching AMD runtime installation. This explains
the missing runtime for that installer, without assuming librocdxg installation
order caused it.

I installed the separate ROCm archive. The command below adds `-f` to my original
`curl -L` so HTTP errors fail. Before extraction, compare `sha256sum
ollama-rocm.tar.zst` with the checksum on the
[v0.30.10 release](https://github.com/ollama/ollama/releases/tag/v0.30.10).

```bash
sudo systemctl stop ollama
curl -fL https://github.com/ollama/ollama/releases/download/v0.30.10/ollama-linux-amd64-rocm.tar.zst -o ollama-rocm.tar.zst
```

After checking the download:

```bash
sudo tar --zstd -C /usr -xf ollama-rocm.tar.zst
```

The runtime landed in `/usr/lib/ollama/rocm_v7_2`, but my service searched
`/usr/local/lib/ollama`. Adding `Environment="OLLAMA_DEBUG=1"` temporarily to the
service override exposed the search paths. This symlink fixed the mismatch:

```bash
sudo ln -s /usr/lib/ollama/rocm_v7_2 /usr/local/lib/ollama/rocm_v7_2
```

Check both paths first. This is a fix for the layout I observed, not a required
step for every Ollama release. An installer update can replace the runtime
directory and remove the link.

### 4. Configure the service

```bash
sudo systemctl edit ollama
```

Add this override:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="HSA_ENABLE_DXG_DETECTION=1"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

The wider binding lets the WebUI container reach Ollama through the WSL host.
Variables exported in `.bashrc` do not configure this system service.

**Network boundary:** `0.0.0.0` listens on every IPv4 interface, and
[Ollama's local API has no authentication](https://docs.ollama.com/api/authentication).
Keep port 11434 off untrusted networks; WebUI login does not protect direct API
access. Reachability from the LAN depends on WSL networking and firewall rules.
Windows-to-WSL localhost access alone does not require this wildcard binding;
see [Microsoft's networking documentation](https://learn.microsoft.com/en-us/windows/wsl/networking).

### 5. Install Docker and start Open WebUI

I installed Docker Engine inside Ubuntu using its convenience script:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"
```

As with Ollama, inspect the downloaded script before running it. Docker documents
its limitations in the [Ubuntu installation guide](https://docs.docker.com/engine/install/ubuntu/).
The [`docker` group grants root-level privileges](https://docs.docker.com/engine/install/linux-postinstall/).
Start a new login session after adding yourself; check `id -nG` for `docker`.

For local-only access, use `-p 127.0.0.1:3000:8080` in place of the original port
argument below. Docker documents this [loopback binding](https://docs.docker.com/engine/network/port-publishing/),
but I did not test it in the original setup. The recorded command publishes on
all host interfaces. Neither choice restricts Ollama's separate port 11434.

This is the container command that worked for me:

```bash
docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  --add-host=host.docker.internal:host-gateway \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

`main` is a moving image tag. Use a specific release or digest for repeatable
installs, following [Open WebUI's current instructions](https://docs.openwebui.com/getting-started/quick-start/).
Those instructions also recommend a persistent `WEBUI_SECRET_KEY` to preserve
sessions across container replacement. Keep the named volume containing WebUI
data. Open `http://localhost:3000` in Windows
and create the initial administrator account locally before allowing other access.

My first attempt used `--network=host` and was not reachable as expected. Port
mapping fixed my setup. This does not mean host networking cannot work under WSL:
host mode also removes the `3000:8080` mapping, leaving the application's port.

## Verify GPU inference

Download a model, then actually run a prompt; pulling alone does not load it:

```bash
ollama pull qwen2.5-coder:14b
ollama run qwen2.5-coder:14b
```

In another Ubuntu terminal, while the model is loaded:

```bash
ollama ps
journalctl -u ollama --since "5 minutes ago" | grep -iE "library=|offloaded"
```

My successful logs included the following lines (layer counts depend on model):

```text
library=ROCm name=ROCm0 description="AMD Radeon RX 9070 XT"
load_tensors: offloading output layer to GPU
load_tensors: offloading 47 repeating layers to GPU
load_tensors: offloaded 49/49 layers to GPU
```

Look for ROCm and full layer offload, plus `100% GPU` in `ollama ps`. That describes
model placement, not zero CPU usage. `rocm-smi` did not work in this WSL setup;
I used Ollama's logs and Windows Task Manager's GPU compute graphs instead.

## Troubleshooting

### Open WebUI connects but lists no models

I first checked the API from inside the container:

```bash
docker exec -it open-webui curl http://host.docker.internal:11434/api/tags
```

It returned connection refused while Ollama listened only on `127.0.0.1`.
The [service override](#4-configure-the-service) fixed that. If a newer WebUI
image lacks `curl`, this diagnostic needs a different HTTP client inside the container.

I also tried starting `OLLAMA_HOST=0.0.0.0:11434 ollama serve` manually. That
instance ran as my user and read a different model directory. The files had
remained under the service account all along:

```bash
sudo find / -iname "*qwen2.5-coder*" 2>/dev/null
# /usr/share/ollama/.ollama/models/...
```

Use the systemd service consistently rather than starting another server as your
own user. For CPU-only inference, return to the [runtime and path checks](#3-install-ollama-and-its-rocm-runtime).

### Larger models froze Windows

`qwen3.5:27b` downloaded about 17 GB on my 16 GB card. `ollama ps` showed
`16%/84% CPU/GPU`. Partial CPU offload is supported, but this setup became
unresponsive during generation.

A separate 13 GB, 3-bit 27B model loaded at about 22 GB, with a context of
131,072 tokens shown by `ollama ps`. Model weights are only part of memory use;
context cache and runtime buffers also need room. Setting
`OLLAMA_CONTEXT_LENGTH=8192` did not change the observed context, and I did not
establish why. A model-specific limit worked:

```bash
cat > Modelfile << 'EOF'
FROM batiai/qwen3.6-27b:q3
PARAMETER num_ctx 8192
EOF
ollama create qwen3.6-27b-8k -f Modelfile
ollama run qwen3.6-27b-8k
```

This uses the community model [batiai/qwen3.6-27b:q3](https://ollama.com/batiai/qwen3.6-27b:q3)
and Ollama's documented [`num_ctx` parameter](https://docs.ollama.com/modelfile).
`ollama ps` then showed 13 GB, `100% GPU`, context 8192, and the freezes stopped.
Select the new model name in WebUI and check again after a prompt: client context
settings can override the model default. These memory figures are observations,
not sizing guarantees for other versions or concurrent requests.

### Precautionary gfx1201 symlink — not a confirmed fix

I added this before encountering a corresponding error:

```bash
sudo mkdir -p /usr/lib/ollama/rocm_v7_2/hipblaslt/library
sudo ln -sf /usr/lib/ollama/rocm_v7_2/rocblas/library/TensileLibrary_lazy_gfx1201.dat \
            /usr/lib/ollama/rocm_v7_2/hipblaslt/library/TensileLibrary_lazy_gfx1201.dat
```

It was prompted by [ROCm issue #7192](https://github.com/ROCm/rocm-libraries/issues/7192)
and [Ollama issue #12908](https://github.com/ollama/ollama/issues/12908), which report
related architecture/library lookup problems in different configurations. They
do not establish that this exact link is necessary or correct for v0.30.10.
The runtime directory mismatch was my confirmed blocker.

This command is retained as part of the setup record, not included in the setup
sequence. Do not apply it blindly: `ln -sf` replaces the destination, and a rocBLAS
data file is not automatically a valid hipBLASLt replacement. Check the actual
error and the runtime's files before considering a workaround.

To stop the stack, run `wsl --shutdown` in Windows PowerShell. This stops **all**
running WSL distributions, not just this Ubuntu instance.
