# Encapsulation System Plan (Project name: "capsule")

## 1. Intent

This project builds a flexible, observable software sandbox. The environment isolates execution while allowing interactive control. It permits on-the-fly reconfiguration. Restarts are not required. The technology stack remains lightweight, proven, and strictly avoids unnecessary complexity.

## 2. Requirements and Architecture

Design constraints dictate the technology choices. Standard application containers fail the primary requirements.



* **Runtime Flexibility:** Docker and Podman seal namespaces at startup. They cannot hot-plug ports or mounts without stopping the environment. We require dynamic injection. We use system containers (LXC) managed by Incus. They share the host kernel but run a full init system. This allows attaching devices and proxying ports while the system remains live.
* **Host Integration:** Docker writes to host volumes as root. This breaks file permissions. We require the container user to map directly to the host user. We use unprivileged LXC containers and punch a specific hole in the ID map. Container user "Alice" maps to host UID 1000.
* **Provisioning:** We build on a Debian base image. We provision the environment using an Incus profile and `cloud-init`. This automates user creation, package installation (Git, Python via `uv`), and mount point preparation on first boot.
* **Observability:** We require visibility into file, process, and network access. Agents inside the container are too heavy. We run eBPF tools on the host. 



* **Noise Reduction:** A full operating system generates massive background noise. We must filter this. Filtering by UID fails because Alice has passwordless `sudo` rights. Escalated processes or detached cron jobs would escape the filter. We discard UID tracking. We filter eBPF events strictly by directory path and cgroup ID. 

## 3. Implementation Tasks

These are the concrete steps required to build the encapsulation system.

### Phase 1: Host Preparation
1.  Configure `/etc/subuid` and `/etc/subgid` to allow the root user to delegate host UID/GID 1000.
2.  Install Incus and initialize the daemon.
3.  Install eBPF dependencies: `bpftrace` and `bpfcc-tools`.

### Phase 2: Environment Definition
1.  Create an Incus profile (`datascience-profile.yaml`).
2.  Define the raw ID mapping for UID/GID 1000 within the profile.
3.  Write the `cloud-init` configuration within the profile to create user Alice, grant `sudo` access, install system dependencies, and install the required Python stack (`uv`, `ipython`, `playwright`, `numpy`, `pandas`, `matplotlib`).

### Phase 3: Runtime Execution
1.  Launch the Incus container using the Debian 12 image and the custom profile.
2.  Attach the read/write host directory using `incus config device add`.
3.  Attach the read-only data directory.
4.  Pass through the host's NVIDIA GPU.
5.  Proxy HTTP ports dynamically when required.

### Phase 4: Observability Setup
1.  Extract the container's root Process ID and resolve its cgroup ID.
2.  Write the `bpftrace` script (`monitor.bt`) to hook `sys_enter_execve` and `sys_enter_openat`.
3.  Implement the path-matching logic (`strncmp`) in the `bpftrace` script to filter file access to the specific mount points.
4.  Run `tcplife-bpfcc` in the background to capture network IPs and ports.

## 4. Sanity Checks

Execute these tests immediately after deployment to verify the encapsulation boundaries.

* **UID Mapping Check:** Create a file inside `/home/alice/host` from within the container. Verify from the host terminal that the file is owned by UID 1000, not root.
* **Hot-Plug Check:** Start a generic web server inside the container. Proxy a port from the host to the container using `incus config device add`. Verify HTTP access from the host browser. Remove the proxy device and verify access drops immediately without container interruption.
* **GPU Access Check:** Run a Python script importing PyTorch or a similar library inside the container. Verify it detects the NVIDIA hardware.
* **Observability Check:** Start the `bpftrace` monitor on the host. Trigger a background system update (`apt update`) inside the container. The log must remain silent. Read a file in `/home/alice/read-only/data`. The log must instantly output the JSON event.
* **Escalation Check:** Inside the container, run `sudo touch /home/alice/host/test.txt`. Verify the eBPF monitor on the host still captures the event despite the user switching to root.