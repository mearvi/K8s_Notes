# Chapter 1 — Infrastructure Setup

This chapter builds the local virtualization layer used by the Kubernetes learning lab. It covers the Windows workstation prerequisites, VirtualBox networking, Vagrant installation and project layout, the complete three-node Vagrantfile, and the guest operating-system preparation required before kubeadm is installed.

The chapter documents the final three-node configuration:

- `master1` — `192.168.56.10`
- `worker1` — `192.168.56.20`
- `worker2` — `192.168.56.21`

The abandoned larger-node design is intentionally excluded.

---

## 1.1 Target architecture

| Node | Role | Private IP | vCPU | Memory |
|---|---|---:|---:|---:|
| `master1` | Kubernetes control plane | `192.168.56.10` | 2 | 4096 MB |
| `worker1` | Kubernetes worker | `192.168.56.20` | 2 | 6144 MB |
| `worker2` | Kubernetes worker | `192.168.56.21` | 2 | 6144 MB |

Total virtual allocation: **6 vCPU and 16 GB RAM**.

The VMs use Ubuntu 22.04 (`ubuntu/jammy64`). VirtualBox provides two network paths:

1. **NAT interface** — outbound connectivity from the guest, commonly assigned an address in `10.0.2.0/24`.
2. **Host-only/private interface** — stable communication between the Windows host and the VMs, using the `192.168.56.0/24` network.

Typical guest interface mapping is:

| Guest interface (typical) | VirtualBox attachment | Purpose |
|---|---|---|
| `enp0s3` | NAT | Outbound access |
| `enp0s8` | Host-only | Stable private lab connectivity |

Interface names are dependent on the guest and VirtualBox configuration. Confirm them on each VM rather than relying on the names alone.

## 1.2 Windows workstation prerequisites

Before creating the VMs, verify the following:

- Hardware virtualization (Intel VT-x or AMD-V) is enabled in the workstation firmware/BIOS.
- Oracle VirtualBox is installed.
- HashiCorp Vagrant is installed.
- The Windows host has sufficient free memory and disk space for the three VMs and their images.
- A terminal is available to run PowerShell commands and Vagrant.
- The host-only network is present in VirtualBox.

The source lab notes recorded a workstation with an 11th-generation Intel Core i7-1185G7, 64 GB RAM, and NVMe storage. The three-node allocation above is the final lab allocation documented here.

### Verify installations

In PowerShell:

```powershell
vagrant --version
VBoxManage --version
```

If `VBoxManage` is not in `PATH`, run it from the VirtualBox installation directory or add that directory to the user/system `PATH`.

## 1.3 Windows Hyper-V and VirtualBox coexistence

VirtualBox performance and networking can be affected by Windows hypervisor features, depending on Windows build, VirtualBox version, and the features enabled on the host. The lab notes treated disabling the Windows hypervisor as a **conditional troubleshooting measure**, not a mandatory setup step.

### Recommended order

1. Start with the existing Windows configuration.
2. Confirm hardware virtualization is enabled in firmware.
3. Create and test one Vagrant VM before starting the complete lab.
4. If VirtualBox exhibits persistent virtualization-backend conflicts or severe instability, review the Windows hypervisor configuration.
5. Before disabling Windows virtualization features, consider whether WSL2, Windows Sandbox, Docker Desktop, or other tools depend on them.

### Check the hypervisor launch setting

Run PowerShell as Administrator:

```powershell
bcdedit /enum {current}
```

Review the `hypervisorlaunchtype` entry if present.

### Conditional workaround: disable hypervisor launch

The original lab notes recorded this command as a possible workaround **only when VirtualBox is slow or Hyper-V conflicts occur**:

```powershell
bcdedit /set hypervisorlaunchtype off
```

Restart Windows for the boot configuration change to take effect.

This setting can prevent WSL2 and other Windows features that depend on the Microsoft hypervisor from working. Do not apply it casually on a workstation that depends on those features.

To restore the normal automatic hypervisor launch behavior:

```powershell
bcdedit /set hypervisorlaunchtype auto
```

Restart Windows again.

### Windows optional features

If a conflict remains, inspect **Turn Windows features on or off** and review the state of Hyper-V, Windows Hypervisor Platform, and Virtual Machine Platform. Change only the feature that is actually implicated by the host's configuration and the installed VirtualBox version. Disabling these features can affect WSL2, Windows Sandbox, and other virtualization-dependent software.

Windows Memory Integrity was also mentioned in the lab notes as a possible source of interaction with virtualization. Treat it as a conditional compatibility check, not a default setting to disable. Follow your organization's endpoint-security policy before changing it.

> **Important:** The recorded lab notes do not establish that every Windows virtualization feature was disabled. The command above is documented as an optional workaround, not as a required step or proof that it was executed on every build of the lab.

## 1.4 Create and map the VirtualBox host-only adapter

The static node addresses are on `192.168.56.0/24`. The host-only adapter provides the private Layer-2 network used by the Vagrant guests.

### Create the adapter in VirtualBox

1. Open **Oracle VM VirtualBox**.
2. Open **File → Tools → Network Manager** (menu wording can vary by VirtualBox release).
3. Select the **Host-only Networks** section.
4. Create a host-only network/adapter.
5. Configure an IPv4 address on the host adapter in the `192.168.56.0/24` subnet, commonly `192.168.56.1`, with mask `255.255.255.0`.
6. Ensure the network does not overlap with another active host, VPN, or corporate network.
7. Apply the changes.

The host-side address `192.168.56.1` is a conventional choice for this subnet; the fixed guest addresses are the values explicitly defined by the Vagrantfile.

### DHCP considerations

The Vagrantfile assigns each guest a static private IP. A DHCP server is therefore not required to assign the three node addresses. If VirtualBox's host-only DHCP server is enabled, ensure its dynamic pool does not overlap the static addresses. For a simple fixed-address lab, either disable host-only DHCP or configure a non-overlapping pool.

### Confirm the host adapter on Windows

In PowerShell:

```powershell
Get-NetAdapter
Get-NetIPConfiguration
ipconfig /all
```

Identify the VirtualBox Host-Only Ethernet Adapter and verify that its IPv4 address belongs to `192.168.56.0/24`.

After the VMs are started, test reachability from Windows:

```powershell
ping 192.168.56.10
ping 192.168.56.20
ping 192.168.56.21
```

A successful ping is a useful network check, subject to Windows Firewall and guest firewall policy.

## 1.5 Install and verify Vagrant

Install Vagrant using the official HashiCorp distribution for Windows. Install VirtualBox separately and verify that the installed Vagrant release supports the installed VirtualBox release.

Verify:

```powershell
vagrant --version
VBoxManage --version
```

Vagrant is the VM lifecycle and provisioning tool; VirtualBox is the hypervisor/provider. The Vagrantfile below uses the VirtualBox provider explicitly.

## 1.6 Create the project directory

Use the following project structure:

```text
C:\k8s-lab\
├── Vagrantfile
├── scripts\
│   └── common.sh
└── manifests\
```

Create the directories from PowerShell:

```powershell
New-Item -ItemType Directory -Force C:\k8s-lab\scripts
New-Item -ItemType Directory -Force C:\k8s-lab\manifests
Set-Location C:\k8s-lab
```

Create the `Vagrantfile` in `C:\k8s-lab` and the provisioning script at `C:\k8s-lab\scripts\common.sh`.

---

## 1.7 Complete Vagrantfile — final three-node configuration

The following is the complete Vagrantfile recovered from the final three-node lab configuration. The node definitions and loop are shown in full.

**File:** `C:\k8s-lab\Vagrantfile`

```ruby
Vagrant.configure("2") do |config|

    BOX = "ubuntu/jammy64"

    config.vm.boot_timeout = 900
    config.vm.box_check_update = false
    config.ssh.insert_key = false

    nodes = {
        "master1" => {
            ip: "192.168.56.10",
            memory: 4096,
            cpus: 2
        },

        "worker1" => {
            ip: "192.168.56.20",
            memory: 6144,
            cpus: 2
        },

        "worker2" => {
            ip: "192.168.56.21",
            memory: 6144,
            cpus: 2
        }
    }

    nodes.each do |name, cfg|

        config.vm.define name do |node|

            node.vm.box = BOX
            node.vm.hostname = name

            node.vm.network "private_network",
                ip: cfg[:ip]

            node.vm.provider "virtualbox" do |vb|
                vb.name = "k8s-#{name}"
                vb.memory = cfg[:memory]
                vb.cpus = cfg[:cpus]

                vb.customize ["modifyvm", :id, "--ioapic", "on"]
            end

            node.vm.provision "shell",
                path: "scripts/common.sh",
                privileged: true

        end
    end
end
```

### Vagrantfile configuration explained

| Setting | Purpose |
|---|---|
| `BOX = "ubuntu/jammy64"` | Uses the Ubuntu 22.04 Vagrant box. |
| `boot_timeout = 900` | Allows up to 900 seconds for a VM to boot. |
| `box_check_update = false` | Prevents automatic box update checks during normal Vagrant operations. |
| `ssh.insert_key = false` | Keeps the box's existing Vagrant SSH key behavior rather than inserting a newly generated key. |
| `nodes` hash | Defines each VM's private IP, memory, and CPU allocation. |
| `nodes.each` | Creates each VM from the same reusable definition. |
| `private_network` | Requests a private/host-only network address for each VM. |
| `vb.name` | Gives the VirtualBox VM a predictable name. |
| `vb.customize ... --ioapic on` | Enables the I/O APIC setting for the VM. |
| `privileged: true` | Runs the common provisioning script with elevated privileges. |

### Start the VMs

From `C:\k8s-lab`:

```powershell
vagrant up
```

For a more controlled first boot, start the machines one at a time:

```powershell
vagrant up master1
vagrant up worker1
vagrant up worker2
```

Check their state:

```powershell
vagrant status
```

Connect to a guest:

```powershell
vagrant ssh master1
```

The first boot downloads the Ubuntu box if it is not already cached, creates the VMs, configures their networks, and runs the provisioning script.

---

## 1.8 Common guest provisioning script

The common provisioning script prepares each Ubuntu guest for the later Kubernetes installation. It disables swap, loads the required kernel modules, configures kernel networking parameters, installs containerd and Kubernetes packages, and enables the relevant services.

**File:** `C:\k8s-lab\scripts\common.sh`

```bash
#!/bin/bash

swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

modprobe overlay
modprobe br_netfilter

cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sysctl --system

apt-get update

apt-get install -y \
    curl \
    apt-transport-https \
    ca-certificates \
    gnupg \
    lsb-release

mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
    gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
    tee /etc/apt/sources.list.d/kubernetes.list

apt-get update

apt-get install -y \
    containerd \
    kubelet \
    kubeadm \
    kubectl

apt-mark hold kubelet kubeadm kubectl

mkdir -p /etc/containerd

containerd config default | tee /etc/containerd/config.toml

sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
    /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd

systemctl enable kubelet
```

> The script above preserves the package repository, package names, and containerd configuration approach recorded in the lab notes. The Kubernetes APT repository is configured for the `v1.30` minor release series. Package availability and upstream repository instructions can change; check the Kubernetes installation documentation before rebuilding the lab at a later date.

## 1.9 OS-level preparation: swap

Kubernetes kubelet expects swap to be disabled unless the cluster is deliberately configured for supported swap behavior. This lab follows the conventional kubeadm setup with swap disabled.

### Disable swap immediately

```bash
sudo swapoff -a
```

This disables active swap until the next boot.

### Disable swap persistently

The provisioning script comments out swap entries in `/etc/fstab`:

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

This prevents the matching fstab swap entries from being activated during subsequent boots.

### Verify

```bash
swapon --show
free -h
```

`swapon --show` should return no active swap devices for this lab configuration.

The `sed` expression is intentionally the same expression used in the recovered provisioning script. Review `/etc/fstab` if a machine uses a nonstandard swap configuration.

## 1.10 OS-level preparation: kernel modules

The Linux bridge netfilter module allows bridged traffic to be visible to netfilter/iptables processing. The overlay module is commonly required by container runtimes.

Load the modules immediately:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure them to load at boot:

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

Verify that they are loaded:

```bash
lsmod | grep -E 'overlay|br_netfilter'
```

## 1.11 OS-level preparation: sysctl networking parameters

The following kernel parameters are written to `/etc/sysctl.d/k8s.conf`:

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
```

Apply all sysctl configuration:

```bash
sudo sysctl --system
```

Verify the effective values:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

Expected values:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

These settings are required groundwork for the cluster's Linux networking and packet forwarding behavior.

## 1.12 Containerd and Kubernetes package preparation

The provisioning script:

1. Installs prerequisites for HTTPS APT repositories and key management.
2. Adds the Kubernetes package-signing key to `/etc/apt/keyrings`.
3. Adds the Kubernetes v1.30 APT repository.
4. Installs `containerd`, `kubelet`, `kubeadm`, and `kubectl`.
5. Holds the Kubernetes packages to prevent unintended package upgrades.
6. Generates the default containerd configuration.
7. switches containerd to the systemd cgroup driver.
8. Restarts and enables containerd and enables kubelet.

### Verify package and service state

On each node:

```bash
containerd --version
kubelet --version
kubeadm version
kubectl version --client
```

Check containerd:

```bash
sudo systemctl status containerd --no-pager
sudo systemctl is-enabled containerd
```

Check kubelet:

```bash
sudo systemctl status kubelet --no-pager
sudo systemctl is-enabled kubelet
```

Before the cluster is initialized and the node has joined a cluster, kubelet may not yet report a fully operational Kubernetes node. The important infrastructure checkpoint here is that the package is installed and the service is enabled.

### Confirm containerd cgroup driver

```bash
sudo grep -n 'SystemdCgroup' /etc/containerd/config.toml
```

The intended setting is:

```toml
SystemdCgroup = true
```

---

## 1.13 Validate the three-node infrastructure

From Windows PowerShell:

```powershell
vagrant status
ping 192.168.56.10
ping 192.168.56.20
ping 192.168.56.21
```

On each guest:

```bash
hostname
ip -br address
free -h
swapon --show
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
sudo systemctl is-enabled containerd
sudo systemctl is-enabled kubelet
```

Expected host-only addresses:

| Hostname | Expected private address |
|---|---|
| `master1` | `192.168.56.10/24` |
| `worker1` | `192.168.56.20/24` |
| `worker2` | `192.168.56.21/24` |

The NAT address may also appear in `ip -br address`. Do not mistake the NAT address for the private address that will be used for stable node communication.

### Confirm the control-plane address before kubeadm initialization

The lab notes show that each VM has both NAT and host-only interfaces. When the cluster is initialized in the next chapter, explicitly select the control-plane host-only address so kubeadm does not advertise the NAT address.

For this topology, the control-plane address is:

```text
192.168.56.10
```

The kubeadm initialization procedure belongs in Chapter 2.

## 1.14 Operational recommendations

- Start the VMs sequentially on the first boot and when the host is under heavy load.
- Keep the Vagrantfile and provisioning script under version control.
- Take VirtualBox snapshots at stable milestones before making substantial changes.
- Do not change the host-only subnet after the Kubernetes cluster has been initialized unless you are prepared to update the cluster networking configuration.
- Avoid starting unrelated high-memory workloads while all three VMs and monitoring components are running.
- Do not commit SSH private keys, Kubernetes certificates, kubeconfig files, passwords, or API tokens.
- Treat the single-control-plane cluster as a learning environment, not a highly available production design.

## 1.15 Chapter completion checklist

- [ ] VirtualBox installed and working.
- [ ] Vagrant installed and working.
- [ ] Hardware virtualization enabled in firmware.
- [ ] Host-only adapter created for `192.168.56.0/24`.
- [ ] Windows host can reach the three guest private IPs.
- [ ] `master1`, `worker1`, and `worker2` are running.
- [ ] Static guest addresses match the topology table.
- [ ] Swap is disabled and persistent swap entries are commented in `/etc/fstab`.
- [ ] `overlay` and `br_netfilter` are configured to load at boot.
- [ ] Required sysctl parameters are set to `1`.
- [ ] containerd is installed, enabled, and configured for the systemd cgroup driver.
- [ ] kubelet, kubeadm, and kubectl are installed.

**Outcome:** the local three-node virtualization layer is ready for Kubernetes cluster bootstrap. Continue with Chapter 2 to initialize the control plane and join the worker nodes.
