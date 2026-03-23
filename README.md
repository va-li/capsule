# Capsule

... is my shot at putting boundaries around a coding agent so that I can run it in full YOLO mode.

## Why?

I want to use a coding agent on my local machine, but don't trust it to not do something stupid.

So my goal is to build a sandbox environment that can:

- run the [pi coding agent](https://pi.dev/) inside it (or use any other coding agent with a CLI), with tools like Python, Git, uv, and nodejs installed
- isolate the agent's execution from the host system (e.g. file system, network, processes, GPU)
- run a terminal in the sandbox to inspect stuff and interact with the agent (e.g. change agent config files)
- share directories, _while the environment_ is running, between the host and the sandbox to exchange files (e.g. sharing a local code repository folder so the agent can read/write code, or sharing a read-only data folder with large datasets)
- map ports, _while the environment_ is running, between the host and the sandbox to exchange network traffic (e.g. running a web server in the sandbox and accessing it from the host browser)
- pass through the GPU to the sandbox so the agent can run machine learning models on the GPU
- limit the resources of the sandbox (CPU, RAM, storage), because I want to be able to use my machine even if the agent goes crazy
- observe file access, process execution, and network traffic from the host (e.g. check which files the agent is reading/writing in the sandbox and in the shared folders, check which domains the agent is connecting to)
- use custom models and API endpoints (this depends mostly on the coding agent, no the sandbox itself)

## How?

Primarily with Linux system containers (LXC)

- **LXC system containers managed with Incus**, because they are a virtualization technology creating a full linux system that shares the host kernel and provides isolation of resources. It supports hot-plugging of devices (files, ports, GPU) while the container is running, unlike Docker or Podman.
- **Mapping the container's root user to a non-root host user** (e.g. `dave`) to allow seamless file sharing without permission issues (docker writes to host volumes as root per default, which is annoying).
- **Installing the coding agent and tools with Incus when starting the container** (using `cloud-init`), so the environment is defined as code and can be easily reproduced or modified.

## Installation

### Requirements

- A host machine running Linux (I used Debian 13) with root access to install packages and configure user namespaces.

## Setup Instructions

### 1. Host Preparation

First, prepare the host machine. This will install Incus and configure user ID mapping so the container can natively share files with the host without breaking permissions.

**Step 1: Install Incus**

Install Incus (see [official documentation](https://linuxcontainers.org/incus/docs/main/tutorial/first_steps/#install-and-initialize-incus)):

```bash
sudo apt install incus
```

Allow your current host user to run incus commands without sudo

```bash
sudo usermod -aG incus-admin $USER
newgrp incus-admin
```

Initialize Incus (can now be run without sudo)

```
incus admin init --auto
```

**Step 3: Configure ID mapping**

An unprivileged container requires a base map of at least 65,536 IDs for the system's background users (e.g., `nobody`, `daemon`, etc.), *plus* our specific mapping so root can map the host's UID/GID directly into the container namespace (allowing file sharing without permissions issues).

Ensure the large block exists, then append your specific UID (`1000`):

```bash
# Provide the large default block if it doesn't already exist
echo "root:1000000:1000000000" | sudo tee -a /etc/subuid
echo "root:1000000:1000000000" | sudo tee -a /etc/subgid
```

Find the UID and GID of your host user:

```bash
id -u # e.g., 1000
id -g # e.g., 1000
```

Replace `1000` in the following lines with your actual host UID/GID from the previous step:

```bash
# Provide the 1-to-1 mapping hole-punch
echo "root:1000:1" | sudo tee -a /etc/subuid
echo "root:1000:1" | sudo tee -a /etc/subgid
```

Restart the Incus daemon so it recognizes the newly added mappings

```bash
sudo systemctl restart incus
```

### 2. Sandbox Environment Definition

The `capsule-profile.yaml` defines the environment configuration for the sandbox container, including network options, identity mappings, and `cloud-init` instructions for installing our python tooling.

**Step 1: Customize the container profile definition**

Open `capsule-profile.yaml` and verify the `uid` / `gid` logic. Find the following section:

```yaml
config:
  raw.idmap: |
    both 1000 0
```

Replace `1000` with your actual host UID/GID that you gathered during the host setup step. This will map the Host user (UID 1000) to the container's `root` user (UID 0). Note: Inside the container, you will operate entirely as `root`.

Under `user.user-data`, you can customize the commands that will run on the first boot of the container. For example, in the following we tell it to install a few packages and then `uv`. But you can run any commands you want to set up the environment, including installing the coding agent.

```yaml
config:
# ...
  user.user-data: |
    #cloud-config

    # Update repositories and install necessary base packages required for our Python setup
    package_update: true
    packages:
      - curl
      - git
      - python3
      - python3-venv
      - nodejs
      - npm

    # Run custom provisioning scripts 
    runcmd:
      - |
        export PYTHON_VERSION="3.13"
        
        # Step A: Install 'uv' (fast python package installer/manager) system-wide
        curl -LsSf https://astral.sh/uv/install.sh | sh -s -- --to /usr/local/bin

        # Set globally so uv defaults to the newly created standard system-level virtual environment
        echo 'export UV_PROJECT_ENVIRONMENT="/root/.venv"' >> /root/.bashrc
        echo "export UV_PYTHON=\"$PYTHON_VERSION\"" >> /root/.bashrc
```

**Step 2: Load the profile into Incus**

Create a new Incus profile named `capsule` and load the configuration from `capsule-profile.yaml`:

```bash
sudo incus profile create capsule
cat capsule-profile.yaml | sudo incus profile edit capsule
```

You can verify the profile was applied correctly by viewing it:

```bash
sudo incus profile show capsule
```

### 3. Runtime Execution

Launch a container with our profile and attach the required resources (directories, ports, GPU) while it's running.

**Launching the container**

Create and start the container instances (we'll name it `capsule-inst`) using the Debian 13 (Trixie) image and your custom profile.

Note the image name has a `/cloud` suffix. This is a special variant of the Debian image that includes `cloud-init`, which uses the configuration we defined in the profile to set up the environment on first boot.

```bash
sudo incus launch images:debian/13/cloud capsule-inst --profile default --profile capsule
```

**Accessing the Container**

Drop into the container to verify it's working. (You will enter as `root`, which is directly mapped to the host user `dave`):
```bash
incus exec capsule-inst -- bash
```

Or for the coding agent

```bash
sudo incus exec capsule-inst -- pi
```

**Allowing Internet Access**

Allow connections to/from the container and to the outside world by adding iptables rules on the host to accept forwarded traffic from the container's network bridge (`incusbr0`, see `capsule-profile.yaml`):

```bash
sudo iptables -I FORWARD -i incusbr0 -j ACCEPT
sudo iptables -I FORWARD -o incusbr0 -j ACCEPT
```

This should probably be made more restrictive. Currently, I'm not sure how.

**Attach Directories (hot-pluggable)**

Create your local host directory and read-only data directory, then attach them to the running container securely. Note that Incus requires absolute paths, so we use `$(pwd)` for the local repository folder:

```bash
# Attach read/write host directory (mapped to the local folder ./capsule-output)
# "host_rw" is just a name for the device, it can be anything
# source=... is the path on the host
# path=... is the path inside the container
mkdir -p ./capsule-output
incus config device add capsule-inst host_rw disk source=$(pwd)/capsule-output path=/root/capsule-output

# Attach read-only data directory
# readonly=true ensures the container cannot modify the host data
mkdir -p ~/capsule-data
incus config device add capsule-inst host_ro disk source=$(pwd)/capsule-data path=/root/capsule-data readonly=true
```

**Making ports inside the container available outside (hot-pluggable)**

To expose a web server running on port 8080 inside the container to the host:
  
```bash
incus config device add capsule-inst port8080 proxy listen=tcp:0.0.0.0:8080 connect=tcp:127.0.0.1:8080
```

Example usage:

Inside the container, start a simple Python HTTP server listening on port 8080:

```bash
python3 -m http.server 8080
```

Open your browser on the host and navigate to `http://localhost:8080`. You should see the directory listing served by the container's HTTP server.

**Pass through NVIDIA GPU**: 

tbd

### 5. Pi agent and TU Wien hosted model integration (Aqueduct)

Pi supports any LLM that is compatible with the OpenAI API specification. To integrate the TU Wien hosted models, add their API configuration to the container under `~/.pi/agent/models.json`.

```json
// Inside the LXC container, create or edit  ~/.pi/agent/models.json and add the following:                                                
{
  "providers": {
    "aqueduct-ai": {
      "baseUrl": "https://aqueduct.ai.datalab.tuwien.ac.at/v1/",
      "api": "openai-completions",
      "apiKey": "YOUR_API_KEY_GOES_HERE", // create api key at https://aqueduct.ai.datalab.tuwien.ac.at/aqueduct/management/tokens/
      "authHeader": true,
      "models": [
        {
          "id": "glm-4.7-355b", // here we use GLM-4.7-355B, but it could be any other model available from aqueduct
          "name": "Aqueduct GLM-4.7 355B",
          "reasoning": true,
          "contextWindow": 200000,
          "compat": {
                "supportsReasoningEffort": false
          }
        },
        {
          "id": "glm-4.5v-106b",
          "name": "Aqueduct GLM-4.5 106B (Vision)",
          "reasoning": true,
          "input": ["text", "image"],
          "contextWindow": 32768,
          "compat": {
                "supportsReasoningEffort": false
          }
        }
      ]
    }
  }
}
```

The `compat` field in both model definitions was necessary, otherwise I would get a 400 error from aqueduct saying that `reasoning_effort` is not a supported parameter.

> References:
> - Aqueduct user guide on tokens (API Key): https://tu-wien-datalab.github.io/aqueduct/user-guide/tokens/
> - Pi custom models documentation: https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/models.md
> - Aqueduct API documentation for completions: https://tu-wien-datalab.github.io/aqueduct/api/completions/
> - Available Aqueduct models with context window sizes: https://datalab.tuwien.ac.at/aiml/aqueduct/#available-models