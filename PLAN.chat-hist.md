---
title: "Encapsulating Software with Dynamic Permissions"
tags: []
author: gemini
count: 13
exporter: 3.0.0
date: 2026-03-21T20-40-46+0100
url: https://gemini.google.com/app/9b4765fcb1b4f0b1
---

# Encapsulating Software with Dynamic Permissions

## Table of Contents

- [1: I'm planning to build a custom encapsulation for software that should…](#chat-1)
- [2: lxc sounds interesting indeed. i'd like to know how it is different in…](#chat-2)
- [3: Thanks, i can see how lxc is useful now. Is there an equivalent of a d…](#chat-3)
- [4: Let's see how an lxc setup for an example use case of mine could look…](#chat-4)
- [5: yes both please](#chat-5)
- [6: Yes, show me how to monitor with ebpf](#chat-6)
- [7: \(for the record, let's scrap the "jupyter inside the container" idea f…](#chat-7)
- [8: yes please](#chat-8)
- [9: wait, but tcplife \_can\_ log the ips and ports? why not bpftrace? or is…](#chat-9)
- [10: thanks, i like the compromise suggestion. one concern i have still is…](#chat-10)
- [11: i think the filter by directory path is sensible, but the uid filter a…](#chat-11)
- [12: okay, we've got that kinda nailed down to a sensible requirement. i'd…](#chat-12)
- [13: Thanks, this is good. Let's now create a PLAN.md file. Please describe…](#chat-13)

## chat-1

> I'm planning to build a custom encapsulation for software that should be able to run interactively, e.g. command line access for a user should be possible, http access via specific ports should be possible, file exchange between the runtime environment and the host should be possible. All of these ideally without restarting the environment where the software is running. If I could wish for anything, I'd wish for on-the-fly enabling/disabling of permissions for this software (file access, ports). I'm trying to learn what possible options I have to do that on linux. I know the basics of docker, that podman is an alternative with tighter permissions per default, that VMs are too heavy for my usecase, that access control lists exist but neither how they work nor if they would be a fit for my usecase. I've heard of dev-containers but don't know enough how they work to understand how they differ from docker. Also, I've heard of nix as a deterministic environment manager and can see how this could be useful in my case, but also that dockerfiles kind of cater to a similar problem, so there is an overlap. Overall, i want to keep it: lightweight, meaing as little technologies as possible and also quickly reconfigurable because i'm not sure yet exactly what all requirements are; simple and boring and reliable over needing to use the newest thing, meaning rather trusting proven technologies that have been shown to be reliable; observable, meaning when the software in the environment does something (file access, http call) i'd like to be able to see, record and search that (this requirement is a more involved one, we could leave this out for now, unless there is a super easy way to do this that integrates well with all the other tools for creating such an encapsulated environment). Now, I'd first like to learn what the differences, overlaps and design considerations are of the tools (docker, podman, dev-containers, acl, nix) i've already mentioned, and also which tools for my purpose i've overlooked.

Your requirements conflict. Strict isolation tools lock down configuration at startup. On-the-fly flexibility requires looser boundaries. You must choose which matters more.

Here is how the tools you mentioned work, where they overlap, and why some fail your requirements.

### The Tools You Mentioned

These tools solve different parts of the encapsulation problem. Some manage the software installation. Others manage the runtime boundaries.

| Tool | Primary Purpose | Overlaps With | Limitations for Your Use Case |
|---|---|---|---|
| Docker | Application containerization. | Podman. | You cannot easily change port bindings or volume mounts on the fly. Requires container restart. |
| Podman | Rootless application containerization. | Docker. | Tighter default security than Docker. Still requires restarts for permission changes. |
| Dev-containers | IDE configuration standard. | Docker/Podman. | It is just a wrapper. It uses Docker or Podman underneath. It adds no new runtime capabilities. |
| Nix | Deterministic package management. | Docker (partially). | Nix isolates dependencies. It does not isolate processes, filesystems, or networks. It builds the environment; it does not secure it. |
| ACLs | Granular host filesystem permissions. | None. | ACLs only control file access. They do nothing for network ports, process isolation, or interactive CLI constraints. |Export to Sheets

### The "On-The-Fly" Problem

Linux containers use features called namespaces and cgroups. Namespaces hide system resources from processes. Cgroups limit resource usage.

When you start a container, Linux seals these namespaces. You cannot inject a new port map into a running Docker container easily. You must stop the process, redefine the network namespace, and restart it. If you want true on-the-fly changes without restarting the environment, standard application containers (Docker/Podman) are the wrong choice.

### Overlooked Tools

To get closer to your goals, you need tools that operate differently than standard application containers.

- **LXC / Incus (System Containers):** These run full Linux environments without the heavy overhead of a VM. They use the same kernel features as Docker. However, their architecture allows hot-plugging. You can attach new disk mounts or pass through new network ports while the container runs. This is proven, reliable technology.
- **Systemd-nspawn:** This tool comes built into systemd. It is a simple tool for spawning namespaces. It is lightweight, boring, and native to almost all modern Linux distributions. It is excellent for running a specific command with a virtualized file system and network.
- **Bubblewrap:** This is a low-level, unprivileged sandboxing tool. It allows you to construct a custom namespace around a single process. Flatpak uses this under the hood. It is strictly for lockdown. You define exact filesystem access at execution time.
- **eBPF (For Observability):** This is the modern standard for Linux tracing. Tools like `bpftrace` or Tetragon let you monitor the kernel safely. You can log exactly which files a process opens and which network calls it makes. It does this without modifying the target software.

### Summary of the Best Path

Docker and Podman box you in. Nix solves build determinism, not runtime isolation.

For your specific combination of lightweight encapsulation, interactive access, and dynamic reconfigurability, system containers provide the best fit.

Would you like me to outline the specific commands to configure and hot-plug a mount into an LXC container?

___
###### [top](#table-of-contents)
## chat-2

> lxc sounds interesting indeed. i'd like to know how it is different in practice to docker containers, a rough overview of how it works, which tooling support exists for it, are there "images" like there are for docker/podman, and as a concrete example: if i create a system container with a user "alice" and id 1001 and this user exists like that on the host too, can i map the user in the container to a user on the host, so that when a folder is mapped into the system container and the software creates a file in the container i can see it on the has as being created by "alice" (e.g. overcome the problem of docker writing everything as root on the host)?

Here are the practical differences between LXC and Docker, how the technology works, and how to solve your user mapping problem.

### Practical Differences: LXC vs. Docker

Docker runs applications. LXC runs operating systems.+1

When you start a Docker container, you run a single process (like a web server). When that process dies, the container dies.

When you start an LXC container, you boot a full Linux init system (like systemd). It behaves exactly like a virtual machine, but without the hardware overhead. You can SSH into it, install packages via `apt`, run cron jobs, and run multiple background services simultaneously. It stays running until you shut it down.+1

### How It Works

LXC uses the exact same Linux kernel features as Docker: namespaces (for isolation) and cgroups (for resource limits).

The difference is the execution model. Docker uses layered filesystems to build application states. LXC carves out a fenced-off area of the host kernel and boots a standard Linux distribution inside it. There is no hypervisor. The processes inside LXC talk directly to the host kernel, making it incredibly fast and lightweight.

### Tooling Support and Images

Yes, LXC uses images. However, they are full OS templates rather than application layers. You download a clean image of Debian, Ubuntu, or Alpine, and build your environment from there.

The modern tooling for LXC is **Incus** (a recent, open-source fork of Canonical's LXD).

Incus provides a simple command-line interface. You manage containers with commands like `incus launch`, `incus start`, and `incus config`. It also has a REST API, allowing you to script and automate everything.

### The "Alice" Example: User Mapping

You can map user "Alice" (UID 1001) in the container directly to "Alice" (UID 1001) on the host. This completely avoids Docker's root ownership problem.

LXC runs unprivileged containers by default. It maps container UIDs to high, unused host UIDs for security. Normally, container UID 0 (root) becomes host UID 100000. Container UID 1001 becomes host UID 101001.+2

To fix this for Alice, you punch a hole in the user map. You tell the container config to do this:

1. Map container UIDs 0 to 1000 to host UIDs 100000 to 101000.
2. **Map container UID 1001 directly to host UID 1001.**
3. Map container UIDs 1002+ to host UIDs 101002+.

You then add a single line to the host's `/etc/subuid` and `/etc/subgid` files allowing the container manager to use UID 1001.

When Alice creates a file inside the mapped folder in the container, the host sees it as owned by Alice (UID 1001). The container remains secure because the container's root user (UID 0) is still isolated as host UID 100000.

Would you like the exact Incus configuration commands to set up this specific UID mapping for Alice?

___
###### [top](#table-of-contents)
## chat-3

> Thanks, i can see how lxc is useful now. Is there an equivalent of a dockerfile for lxc/incus, or does environment setup work via any other tool of choice? (e.g. bash scripts, ansible, nix, ...)

There is no exact Dockerfile equivalent. Dockerfiles build layered application states. LXC and Incus containers are full operating systems. You provision them like virtual machines.+1

You have three main options for setting up an Incus environment.

### 1. Distrobuilder (The Image Builder)

This is the closest tool to a Dockerfile. It uses a YAML configuration file. You define repositories, packages, and scripts. It builds a flat root filesystem image. You import this resulting image into Incus.+3

It lacks Docker's step-by-step layer caching. It is strictly for creating a custom base OS image before the container ever runs.

### 2. Cloud-init (The Bootstrapper)

Incus integrates natively with `cloud-init`. You pass a YAML file to Incus when you launch the container. The container executes these instructions on its first boot.+1

It handles initial state setup. It creates users, injects SSH keys, writes configuration files, installs packages, and runs initial shell commands.

### 3. External Provisioners (The VM Approach)

Once the container boots, you manage it exactly like a standalone Linux server. You use your preferred configuration management tools.

- **Bash Scripts:** You can push a script into the container and execute it directly using `incus exec`.
- **Ansible:** You run playbooks against the container. Ansible has a dedicated connection plugin for Incus. It connects directly via the Incus API instead of requiring SSH.
- **Nix:** You can install the Nix package manager inside the container. This gives you deterministic package management inside the isolated environment.

You must separate the base image creation from the runtime configuration. Distrobuilder creates the blank slate. Cloud-init or external tools configure the running instance.

Would you like to see a minimal `cloud-init` YAML example that creates a user and installs software on first boot?

___
###### [top](#table-of-contents)
## chat-4

> Let's see how an lxc setup for an example use case of mine could look like: i want to build on debian, have a user "alice" that maps to a host user, in the lxc alice has full sudo rights, i want git and python's uv to be installed, also for python the ipython, jupyter, playwright for python should be installed with uv, as well as numpy, pandas and matplotlib. further, i want per default a folder from the lxc container in /home/alice/host to be mapped on the host and let the alice in the container have read and write access to this folder, also i want a folder /home/alice/read-only/data to be a read only mapping to a host folder (this will have some numpy arrays and json files),
> 
> and i want to give the software running in the container access to the host's nvidia gpu.

This requires a combination of host configuration, a declarative cloud-init profile, and runtime device attachments.

I will use Incus (the modern continuation of LXD) for this setup. I assume your host user "Alice" has UID 1000 and GID 1000.

### 1. Host Preparation

Incus needs permission to use your host's UID 1000. It runs as root, but maps unprivileged container users to high host UIDs by default. You must punch a hole for UID 1000.

Run these commands on your host:

```bash
echo "root:1000:1" | sudo tee -a /etc/subuid
echo "root:1000:1" | sudo tee -a /etc/subgid
```

### 2. The Configuration Profile

We will create an Incus profile. This profile contains the ID mapping and the `cloud-init` instructions. Cloud-init will run exactly once when the Debian image first boots.

Create a file named `datascience-profile.yaml`:

```yaml
config:
  raw.idmap: |
    uid 1000 1000
    gid 1000 1000
  user.user-data: |
    #cloud-config
    users:
      - name: alice
        uid: 1000
        groups: sudo
        shell: /bin/bash
        sudo: ALL=(ALL) NOPASSWD:ALL
    packages:
      - git
      - curl
    runcmd:
      - mkdir -p /home/alice/host /home/alice/read-only/data
      - chown -R alice:alice /home/alice
      - sudo -u alice curl -LsSf https://astral.sh/uv/install.sh | sudo -u alice sh
      - sudo -u alice /home/alice/.cargo/bin/uv venv /home/alice/venv
      - sudo -u alice /home/alice/.cargo/bin/uv pip install ipython jupyter playwright numpy pandas matplotlib
      - sudo -u alice /home/alice/venv/bin/playwright install --with-deps chromium
```

Apply this profile to Incus:

```bash
incus profile create datascience
cat datascience-profile.yaml | incus profile edit datascience
```

### 3. Launch and Attach Resources

Now you launch the container and attach the mounts and hardware.

1. **Launch the container.** This uses the default profile (for networking) and your new profile (for setup).

```bash
incus launch images:debian/12 myenv -p default -p datascience
```
2. **Attach the Read/Write host folder.**

```bash
incus config device add myenv shared_dir disk source=/path/on/host/share path=/home/alice/host
```
3. **Attach the Read-Only data folder.** ```bash
incus config device add myenv ro_data disk source=/path/on/host/data path=/home/alice/read-only/data readonly=true

```

```
4. **Pass through the NVIDIA GPU.** Incus handles the complex cgroup and device node mapping automatically.

```bash
incus config device add myenv mygpu gpu
```
> **Note on Playwright:** The `cloud-init` script installs Chromium and its system dependencies automatically. Cloud-init runs in the background. Wait a minute or two after launching the container before expecting the Python environment to be fully ready.

Would you like me to outline the commands to open a shell as Alice inside this environment or show how to expose the Jupyter server port to your host?

___
###### [top](#table-of-contents)
## chat-5

> yes both please

You need two specific commands to access the environment and map the port.

### 1. Open a Shell as Alice

Use the `incus exec` command. Pass the `--user` flag to sudo to specify the user context.

```bash
incus exec myenv -- sudo --login --user alice
```

This drops you directly into Alice's home directory. Her environment variables will be loaded.

### 2. Expose the Jupyter Port

Incus handles port forwarding dynamically. You do not need to restart the container. You add a `proxy` device to the running instance.

Run this command on your host:

```bash
incus config device add myenv jupyterport proxy listen=tcp:0.0.0.0:8888 connect=tcp:127.0.0.1:8888
```

This tells the host to listen on port 8888. It forwards all traffic to port 8888 inside the container.

To use this, open your shell as Alice. Start Jupyter. Bind it to the local interface inside the container.

```bash
/home/alice/venv/bin/jupyter lab --ip=127.0.0.1 --port=8888
```

Open a web browser on your host machine. Navigate to `http://localhost:8888`. You will see the Jupyter interface.

If you want to remove the port access later, run:

```bash
incus config device remove myenv jupyterport
```

The port closes immediately. The container keeps running.

Would you like to know how to monitor network and file access within this running container using eBPF, as you mentioned in your initial requirements?

___
###### [top](#table-of-contents)
## chat-6

> Yes, show me how to monitor with ebpf

You must run eBPF tools on the host. Containers lack the security privileges to load kernel programs. The host kernel handles all container actions. You will monitor the host and filter for the container.

Install the eBPF compiler collection (BCC) tools on your host. These are pre-written eBPF scripts.

```bash
sudo apt install bpfcc-tools
```

### Monitor File Access

Use the `opensnoop` tool. It traces the `open()` system call globally.

Run this on the host:

```bash
sudo opensnoop-bpfcc
```

This prints every file opened by any process. You will see the PID, the command name, and the file path.

### Monitor Network Connections

Use the `tcplife` tool. It logs TCP sessions. It shows the local IP, remote IP, ports, and the amount of data transferred.

Run this on the host:

```bash
sudo tcplife-bpfcc
```

### Filter for Your Container

These tools monitor the entire host. You need to isolate the container's activity. First, find the root Process ID (PID) of your container on the host.

```bash
incus info myenv | grep Pid
```

Assume the PID is `54321`. Every process inside the container is a child of this PID. You can filter the eBPF output using standard Linux tools.

```bash
sudo opensnoop-bpfcc | grep -E "python|jupyter"
```

If you need structured, searchable logs instead of a live terminal feed, install **Tetragon**. It is a dedicated eBPF security and observability agent. It outputs JSON logs. It can filter events natively by the container's cgroup.

Would you like me to provide the configuration file needed to deploy Tetragon for this specific container?

___
###### [top](#table-of-contents)
## chat-7

> (for the record, let's scrap the "jupyter inside the container" idea for now, because it might be too cumbersome to set this up so that execution of notebooks can be observed "in realtime" from the host by opening a notebook, tricky yjs interaction and unstable implementation in jupyter and all that)
> 
> 
> 
> 
> i've taken a look at tetragon, it seems a bit "heavy" for my use case. i care about file access, process starts/stops, and network access in the container. is there another option to deal with the output of opensnoop and tcplife?

Dropping Jupyter avoids unnecessary complexity. Tetragon is an enterprise security tool. It is too heavy for your needs.

You need to track file access, process lifecycles, and network activity. You have three simple ways to handle this data without heavy agents.

### 1. Write to Flat Files

The BCC tools (`opensnoop`, `execsnoop`, `tcplife`) output plain text. You can redirect this text directly to files on the host. Run the processes in the background.

```bash
sudo opensnoop-bpfcc > /var/log/incus-files.log &
sudo execsnoop-bpfcc > /var/log/incus-processes.log &
sudo tcplife-bpfcc > /var/log/incus-network.log &
```

Search these files later using `grep`. Filter the text using the container's base process ID.

### 2. Send to the System Journal

You can route the text output to the host's built-in logging system. Pipe the output to the `logger` command.

```bash
sudo opensnoop-bpfcc | logger -t incus-files &
```

Read the recorded events using `journalctl`.

```bash
journalctl -t incus-files
```

### 3. Replace BCC with bpftrace

Parsing plain text logs is fragile. If you need structured logs like JSON, do not use BCC tools. Use `bpftrace`.

It is a lightweight tracing language. It compiles directly to eBPF. It handles JSON natively.

Install it on your host.

```bash
sudo apt install bpftrace
```

You run a single script on the host. This script attaches to kernel tracepoints for files, processes, and network sockets. You instruct the script to format the output as JSON. You filter the events by the container's cgroup ID. This guarantees you only see activity from your specific container.

Would you like the exact `bpftrace` script that combines file, process, and network tracing into one JSON stream filtered by your container?

___
###### [top](#table-of-contents)
## chat-8

> yes please

You need the container's cgroup ID. eBPF uses this ID to filter events directly in the kernel.

### 1. Get the Cgroup ID

Find the root process ID of your container. Extract its cgroup path. Read the inode number of that path. Run this on your host:

```bash
PID=$(incus info myenv | awk '/Pid:/ {print $2}')
CGROUP_PATH=$(cat /proc/$PID/cgroup | cut -d: -f3)
CGROUP_ID=$(stat -c %i /sys/fs/cgroup$CGROUP_PATH)
echo $CGROUP_ID
```

### 2. The bpftrace Script

Create a file named `monitor.bt`. This script hooks into the kernel system calls for process execution, file opening, and network connections. It prints manually formatted JSON.

```c
BEGIN {
    printf("{\"event\": \"start\", \"cgroup\": %d}\n", $1);
}

tracepoint:syscalls:sys_enter_execve
/cgroup == $1/
{
    printf("{\"event\": \"process\", \"pid\": %d, \"cmd\": \"%s\"}\n", pid, str(args->filename));
}

tracepoint:syscalls:sys_enter_openat
/cgroup == $1/
{
    printf("{\"event\": \"file\", \"pid\": %d, \"file\": \"%s\"}\n", pid, str(args->filename));
}

tracepoint:syscalls:sys_enter_connect
/cgroup == $1/
{
    printf("{\"event\": \"network\", \"pid\": %d, \"process\": \"%s\"}\n", pid, comm);
}
```

### 3. Run the Tracer

Execute the script. Pass the `CGROUP_ID` variable as the first argument.

```bash
sudo bpftrace monitor.bt $CGROUP_ID
```

### The Network Tracing Limitation

This script logs the fact that a network connection attempt occurred. It identifies the process. It does not log the target IP address or port.

Extracting IP addresses in raw `bpftrace` requires including specific kernel headers and casting deep C structures (`struct sockaddr`). This breaks constantly across different kernel versions. It violates your requirement for simple, reliable technology.

If you strictly need exact IP addresses and ports in JSON format, you must write a custom eBPF program in C or Go using `libbpf`, or accept the overhead of a tool like Tetragon. Raw `bpftrace` is the wrong tool for reliable packet inspection.

Would you like to move on to restricting the container's resource usage, like capping the RAM or CPU cores available to Alice?

___
###### [top](#table-of-contents)
## chat-9

> wait, but tcplife _can_ log the ips and ports? why not bpftrace? or is tcplife exactly such a custom program that uses libbpf?

You hit the nail on the head. That is exactly what is happening.

`tcplife` is not a simple script; it is a dedicated, custom-built program from the BCC (BPF Compiler Collection) suite. It does all the heavy lifting in C and Python that `bpftrace` struggles to do elegantly.

Here is the breakdown of why `tcplife` can do it easily and why writing it in raw `bpftrace` is a headache.

### How `tcplife` Works (The BCC Approach)

`tcplife` is actually two programs wrapped into one:

1. **The Kernel Code (C):** It contains raw C code that gets compiled on the fly and injected into the kernel. Because it is written in C, it can easily include Linux kernel headers. It hooks into TCP state changes (like `tcp_set_state`), grabs the raw `struct sock` (the kernel's internal socket data structure), handles the complicated logic of checking if it is IPv4 or IPv6, and deals with network-byte-order vs. host-byte-order conversions.
2. **The User-Space Code (Python):** The C code sends these raw bytes over a high-speed ring buffer to a Python script. The Python script takes those bytes, formats them into human-readable IP addresses using standard libraries, and prints them to your screen.

### Why `bpftrace` Struggles with IPs

`bpftrace` is an `awk`-like scripting language designed for quick, one-off tracing. It is not a full programming language like Python or C.

To get an IP address out of a network connection in `bpftrace`, you have to:

1. Cast raw kernel memory pointers to kernel structures like `struct sockaddr` or `struct sock`.
2. Navigate deep into nested unions and structs (e.g., `sk->__sk_common.skc_rcv_saddr`).
3. Manually pass those raw integers to `bpftrace`'s `ntop()` function to format them as strings.

**The catch:** Linux kernel developers change these internal structures frequently. A `bpftrace` script that successfully extracts an IP address on Linux kernel 5.15 might completely crash or return garbage data on kernel 6.1 because the byte offset of the IP address inside the `struct sock` moved by 8 bytes.

`tcplife` handles these kernel version differences gracefully under the hood. A simple `bpftrace` script cannot.

### The Compromise

If you want the absolute easiest path for your use case without deploying heavy agents like Tetragon:

1. **For Files and Processes:** Use `bpftrace` (like the script I provided). It handles `execve` and `openat` system calls beautifully because file paths and command names are easy strings to extract.
2. **For Network IPs:** Keep running `tcplife` in the background and pipe its output to a file or the system journal.

Mixing the two gives you the structured JSON you want for the easy stuff (files/processes) and relies on the robust, pre-built BCC tooling for the hard stuff (network IPs and ports).

Would you like to move on to restricting the container's resource usage, like capping the RAM or CPU cores available to Alice?

___
###### [top](#table-of-contents)
## chat-10

> thanks, i like the compromise suggestion. one concern i have still is that the log will be quite noisy, since the lxc runs a full operating system there will be a _lot_ of file access happening.

Systemd generates massive background noise. It constantly reads `/proc` and writes to `/var/log`.

You must filter events directly in the kernel. Capturing everything and using `grep` later wastes CPU.

You have two strict ways to eliminate this OS noise.

### 1. Filter by User ID (UID)

We mapped the container user Alice to host UID 1000. The container's background OS services run as root. Root maps to host UID 100000.

You can instruct your eBPF tools to ignore everything except UID 1000.

In `bpftrace`, you add `uid == 1000` to your condition filter.

```c
tracepoint:syscalls:sys_enter_openat
/cgroup == $1 && uid == 1000/
{
    printf("{\"event\": \"file\", \"file\": \"%s\"}\n", str(args->filename));
}
```

This immediately drops 99% of the noise. You will only see activity initiated by your software running under Alice.

### 2. Filter by Directory Path

If you only care about specific data folders, filter the path string.

`bpftrace` uses `strncmp` to match the beginning of a file path.

```c
tracepoint:syscalls:sys_enter_openat
/cgroup == $1 && strncmp(str(args->filename), "/home/alice/host", 16) == 0/
{
    printf("{\"event\": \"file_access\", \"file\": \"%s\"}\n", str(args->filename));
}
```

Combine both filters in your script. The log will remain completely silent until your specific user touches your specific mapped folders.

Would you like the complete, updated `bpftrace` script containing these exact UID and path filters, or should we move on to restricting the container's RAM and CPU limits?

___
###### [top](#table-of-contents)
## chat-11

> i think the filter by directory path is sensible, but the uid filter approach has a problem: imagine alice starts a cron job as root, then the job won't be the same uid as her but it will be still an action that ultimately comes from this user but escapes the filter rules.

You are correct. The UID filter is fundamentally flawed for your setup because you gave Alice full passwordless `sudo` rights.

When Alice runs `sudo`, her Effective UID changes to the container's root (mapped to host UID 100000). If she creates a system cron job, that job runs entirely independently of her user session. The `uid == 1000` check will blind you to these actions.

If you give a user root access inside a system container, they effectively _are_ the operating system. Perfectly separating their root actions from the background OS root actions using simple filters is impossible.

Here is how you deal with this reality.

### 1. Rely Exclusively on the Path Filter

If your goal is to know what happens to the files in `/home/alice/host` and `/home/alice/read-only/data`, watch the data, not the user.

The path filter (`strncmp`) does not care if Alice, root, or a detached cron job opens the file. If a process touches that directory, eBPF logs it. This drops the OS background noise while guaranteeing you do not miss escalated file access.

### 2. The Limitations of Tracking Actors

To track a user through `sudo` in Linux, you need to monitor the Audit UID (`loginuid`). The kernel uses this to remember the original user who logged in, even after they switch to root.

However, standard `bpftrace` does not have a simple built-in variable for this. Extracting it requires writing C-style struct dereferences to pull it from kernel memory (`((struct task_struct *)curtask)->loginuid.val`). Just like extracting IP addresses, this makes your script fragile and prone to breaking across kernel updates.

Furthermore, if Alice schedules a system cron job, that job spawns from the cron daemon, not her login session. The `loginuid` link is permanently broken.

### The Verdict

Drop the UID filter. It will give you a false sense of visibility.

Use the cgroup filter to isolate the container, and use the directory path filter to isolate the relevant file activity. Let the container's OS do its noisy background work in system directories, and only capture events that strike the specific mount points you care about.

Shall we move on to configuring the CPU and RAM limits for this container to restrict what Alice's environment can consume?

___
###### [top](#table-of-contents)
## chat-12

> okay, we've got that kinda nailed down to a sensible requirement. i'd like to consolidate what we've discussed and decided so far. it is very important that all the considerations and questions i've brought up are included in this consolidation. they describe the requirements i have to this encapsulation system, the tools i've considered useful, the questions about the tools i've had reflect also what i've been thinking about. please put this together into a kind of design plan for this encapsulation system. this should not be code, but a structured documentation of the thought process that played out in this conversation.

Design constraints dictate technology choices. You brought specific requirements for an encapsulation system. We evaluated multiple tools and discarded those that failed your criteria. This document records the resulting design plan and the thought process behind it.

**1. Core Requirements**
You need an interactive software environment. It requires command-line access. It needs HTTP port mapping and host-to-container file exchange. The critical constraint is on-the-fly reconfigurability. You must change permissions and ports without restarting the environment. The stack must remain lightweight, simple, and proven. You also demand deep observability to track file, process, and network access.

**2. Tool Evaluation**
You initially considered Docker, Podman, Dev-containers, Nix, and ACLs. We rejected standard application containers. Docker and Podman seal namespaces at startup. They cannot hot-plug ports or mounts without a restart. Dev-containers just wrap these underlying engines. Nix manages packages deterministically but provides no runtime sandboxing. ACLs manage files but ignore networks and processes.

We selected LXC managed by Incus. System containers run a full OS init system. They share the host kernel but behave like lightweight virtual machines. This architecture supports hot-plugging devices and proxying ports while the system runs.

**3. Environment Provisioning**
The system uses a Debian base image. We rejected Dockerfiles. System containers require OS provisioning. We use an Incus profile with `cloud-init` for initial setup.

The environment requires specific user mapping. Docker writes to host volumes as root. You needed the container user "Alice" to map directly to the host user "Alice" (UID 1000). We achieve this by punching a hole in the unprivileged container's ID map. Files created in the container retain correct host ownership.

The `cloud-init` script installs Git, Python (`uv`), and data science libraries. It creates specific mount points. We map a read/write host folder and a read-only data folder. We pass the host's NVIDIA GPU directly into the container. Alice receives passwordless `sudo` rights.

**4. Observability Strategy**
You require visibility into the container's actions. The monitoring tools must run on the host.

We evaluated Tetragon. We rejected it as too heavy. We evaluated writing a pure `bpftrace` script. We rejected it for network monitoring. Extracting IP addresses from kernel structures with `bpftrace` breaks across kernel updates.

We chose a hybrid eBPF approach. We use a custom `bpftrace` script to log process executions and file access as JSON. We use the BCC tool `tcplife` in the background to log network connections. Both tools isolate target activity using the container's cgroup ID.

**5. Noise Reduction and Filtering**
LXC runs systemd. Systemd generates massive background file activity. We needed to filter this noise.

I suggested filtering eBPF events by Alice's UID. You identified a critical flaw. Alice has `sudo` rights. If she schedules a system cron job, it runs as root. The UID filter would ignore it. Tracking the original login user through `sudo` and detached processes is fragile.

We discarded the UID filter. We filter file access strictly by directory path. The `bpftrace` script only logs events that touch the mapped host directories. This ignores background OS noise while capturing all relevant file operations, regardless of the executing user.

___
###### [top](#table-of-contents)
## chat-13

> Thanks, this is good. Let's now create a PLAN.md file. Please describe the encapsulation system and use in a pyramid like structure to keep the very first sentences short and light on implementation detail but high on intent and requirement. then expand on the requirements a bit more and include the thought process, showing what decisions where reached (this is basically the listing you just provided) and finally describe clearly what tools, tasks and sanity checks are necessary to turn these requirements, thought and decisions into the software that i want to build.

# Encapsulation System Plan

## 1. Intent

This project builds a flexible, observable software sandbox. The environment isolates execution while allowing interactive control. It permits on-the-fly reconfiguration. Restarts are not required. The technology stack remains lightweight, proven, and strictly avoids unnecessary complexity.

## 2. Requirements and Architecture

Design constraints dictate the technology choices. Standard application containers fail the primary requirements.

- **Runtime Flexibility:** Docker and Podman seal namespaces at startup. They cannot hot-plug ports or mounts without stopping the environment. We require dynamic injection. We use system containers (LXC) managed by Incus. They share the host kernel but run a full init system. This allows attaching devices and proxying ports while the system remains live.
- **Host Integration:** Docker writes to host volumes as root. This breaks file permissions. We require the container user to map directly to the host user. We use unprivileged LXC containers and punch a specific hole in the ID map. Container user "Alice" maps to host UID 1000.
- **Provisioning:** We build on a Debian base image. We provision the environment using an Incus profile and `cloud-init`. This automates user creation, package installation (Git, Python via `uv`), and mount point preparation on first boot.
- **Observability:** We require visibility into file, process, and network access. Agents inside the container are too heavy. We run eBPF tools on the host.
- **Noise Reduction:** A full operating system generates massive background noise. We must filter this. Filtering by UID fails because Alice has passwordless `sudo` rights. Escalated processes or detached cron jobs would escape the filter. We discard UID tracking. We filter eBPF events strictly by directory path and cgroup ID.

## 3. Implementation Tasks

These are the concrete steps required to build the encapsulation system.

### Phase 1: Host Preparation

1. Configure `/etc/subuid` and `/etc/subgid` to allow the root user to delegate host UID/GID 1000.
2. Install Incus and initialize the daemon.
3. Install eBPF dependencies: `bpftrace` and `bpfcc-tools`.

### Phase 2: Environment Definition

1. Create an Incus profile (`datascience-profile.yaml`).
2. Define the raw ID mapping for UID/GID 1000 within the profile.
3. Write the `cloud-init` configuration within the profile to create user Alice, grant `sudo` access, install system dependencies, and install the required Python stack (`uv`, `ipython`, `playwright`, `numpy`, `pandas`, `matplotlib`).

### Phase 3: Runtime Execution

1. Launch the Incus container using the Debian 12 image and the custom profile.
2. Attach the read/write host directory using `incus config device add`.
3. Attach the read-only data directory.
4. Pass through the host's NVIDIA GPU.
5. Proxy HTTP ports dynamically when required.

### Phase 4: Observability Setup

1. Extract the container's root Process ID and resolve its cgroup ID.
2. Write the `bpftrace` script (`monitor.bt`) to hook `sys_enter_execve` and `sys_enter_openat`.
3. Implement the path-matching logic (`strncmp`) in the `bpftrace` script to filter file access to the specific mount points.
4. Run `tcplife-bpfcc` in the background to capture network IPs and ports.

## 4. Sanity Checks

Execute these tests immediately after deployment to verify the encapsulation boundaries.

- **UID Mapping Check:** Create a file inside `/home/alice/host` from within the container. Verify from the host terminal that the file is owned by UID 1000, not root.
- **Hot-Plug Check:** Start a generic web server inside the container. Proxy a port from the host to the container using `incus config device add`. Verify HTTP access from the host browser. Remove the proxy device and verify access drops immediately without container interruption.
- **GPU Access Check:** Run a Python script importing PyTorch or a similar library inside the container. Verify it detects the NVIDIA hardware.
- **Observability Check:** Start the `bpftrace` monitor on the host. Trigger a background system update (`apt update`) inside the container. The log must remain silent. Read a file in `/home/alice/read-only/data`. The log must instantly output the JSON event.
- **Escalation Check:** Inside the container, run `sudo touch /home/alice/host/test.txt`. Verify the eBPF monitor on the host still captures the event despite the user switching to root.

Would you like to draft the exact `datascience-profile.yaml` file next, or start with the host system preparation commands?

___
###### [top](#table-of-contents)

