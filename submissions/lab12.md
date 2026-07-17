# Lab 12 — BONUS — Submission

## Task 1: Install + Hello World

### Host Environment

- **Kernel (host):**

```text
Linux 6.8.0-1052-azure #58~22.04.1-Ubuntu SMP Thu Mar 26 05:02:21 UTC 2026 x86_64 Linux
```

- **KVM availability:**

```text
crw-rw---- 1 root 109 10, 232 Jul 17 16:31 /dev/kvm
```

- **containerd version:**

```text
2.2.1-1
```

---

## Kata Installation

- **Kata version:**

```text
3.3.0-alpha0-1896-g5f11c0f14
```

- **containerd runtime configuration:**

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
```

---

## Kernel Inside Containers

### runc (Docker)

```text
Linux f0acdfb2d60d 6.8.0-1052-azure #58~22.04.1-Ubuntu SMP Thu Mar 26 05:02:21 UTC 2026 x86_64 Linux
```

### Kata

```text
Error: failed to create shim task: Unix syslog delivery error
```

Expected behavior: Kata should show a different kernel because containers run inside isolated virtual machines.

---

## Why the Kernel Differs (Reading 12)

Kata Containers runs each container inside a lightweight virtual machine with its own kernel, while runc containers share the host kernel. This creates a stronger security boundary because workloads are isolated at the hardware virtualization layer.

In contrast, runc relies on Linux namespaces and cgroups while still depending on the host kernel. As a result, kernel vulnerabilities such as CVE-2024-21626 ("Leaky Vessels") may potentially allow container escape through kernel-level weaknesses. Kata reduces this attack surface by separating containers from the host kernel.

---

# Task 2: Isolation + Performance

## Isolation: `/dev` Differences

Not performed. Docker was used instead of nerdctl.

## Isolation: Capability Sets

Not performed. Docker was used instead of nerdctl.

---

## Startup Time (5-run Average)

| Runtime | Average Startup Time |
|---|---:|
| runc (Docker) | ~0.5 s |
| Kata | Failed to start |

---

## I/O Throughput (100MB `dd` Test)

Not performed because Kata failed to start.

---

## Trade-off Analysis (Reading 12)

Kata Containers provides stronger security isolation through VM-based container execution, which is useful for multi-tenant environments running untrusted workloads such as CI/CD runners or serverless applications.

The additional security comes with performance overhead because creating a micro-VM requires more startup time and memory than a standard runc container.

For trusted workloads where low latency is the main requirement, runc remains the better choice because it is lightweight and fast. Kata is more suitable when security isolation is more important than maximum performance.

---

# Bonus: Container Escape PoC

## Vector Chosen

**Option B: Privileged container with host filesystem write access**

### Reason

This option was selected because it is simple to demonstrate and clearly shows the difference between runc and Kata isolation.

---

## runc: Escape Succeeds

### Command

```bash
sudo docker run --rm --privileged \
  -v /tmp:/host_tmp \
  alpine:3.20 \
  sh -c 'echo "OVERWRITTEN BY RUNC" > /host_tmp/lab12-target && cat /host_tmp/lab12-target'
```

### Container Output

```text
OVERWRITTEN BY RUNC
```

### Host Verification

```bash
sudo cat /tmp/lab12-target
```

Output:

```text
OVERWRITTEN BY RUNC
```

The container successfully modified a file on the host through the bind mount.

---

## Kata: Escape Blocked

### Command

```bash
sudo docker run --rm \
  --runtime=io.containerd.kata.v2 \
  --privileged \
  -v /tmp:/host_tmp \
  alpine:3.20 \
  sh -c 'echo "ATTEMPTED FROM KATA" > /host_tmp/lab12-target'
```

### Container Output

```text
docker: Error response from daemon:
failed to create task for container:
failed to create shim task:
Unix syslog delivery error
```

### Host Verification

```bash
sudo cat /tmp/lab12-target
```

Output:

```text
OVERWRITTEN BY RUNC
```

The file was not modified by Kata because the Kata container failed to start.

---

# Threat Model Implication (Reading 12)

Kata reduces the risk of container escape attacks because every container runs inside an isolated micro-VM with its own kernel and virtualized environment.

Privileged operations inside the guest VM do not provide direct access to the host filesystem or host kernel. This is important for multi-tenant environments where users may run potentially malicious workloads.

However, Kata does not eliminate hardware-level threats such as side-channel attacks (for example Spectre/Meltdown) or cross-tenant timing attacks. Additional protection may require Confidential Containers (CoCo) with hardware-based memory encryption technologies such as Intel TDX or AMD SEV-SNP.
