---
title: CAPEv2 Linux Malware Sandbox Installation Guide
date: 2026-03-02
categories: [Cybersecurity, Malware Analysis]
tags: [capev2, linux, sandbox, malware, ubuntu, automation]
description: A comprehensive guide to setting up CAPEv2 for automating Linux  malware triaging.
image:
  path: /assets/images/cape/cape.png
  alt: CAPE
---

[CAPE (Config and Payload Extraction)](https://github.com/kevoreilly/CAPEv2) is a malware sandbox designed to execute and analyze malicious files in an isolated environment. Derived from Cuckoo v1, CAPE extends traditional sandbox capabilities with features like automated dynamic malware unpacking, YARA-based malware classification, static and dynamic malware configuration extraction, and an advanced debugger for anti-sandbox countermeasures.

## Core Features

* **Behavioral Instrumentation:** Uses API hooking to monitor file operations and network traffic.
* **Malware Classification:** Utilizes behavioral and network signatures, YARA scans, and Suricata scans.
* **Automated Unpacking:** Supports techniques like process injection and DLL injection to capture unpacked payloads.
* **Debugger Integration:** Allows programmable debugging actions via YARA signatures to handle evasive malware.

## History

* Originated from Cuckoo Sandbox, a Google Summer of Code project in 2010.
* CAPE was developed in 2015 by Kevin O'Reilly at Context Information Security.
* Publicly released as CAPE version 1 in September 2016.
* In 2019, **CAPEv2** was launched with contributions from Andriy 'doomedraven' Brukhovetskyy, porting the project to Python 3.

---

## Prerequisites

* Recommended host machine: [Ubuntu 22.04 LTS](https://releases.ubuntu.com/jammy/).
* **Nested Virtualization:** Ensure this is enabled on your hypervisor if testing locally.
* **Hardware:** At least 100 GB of disk space and 16 GB of RAM.

> This installation of CAPE is specifically configured to analyze **Linux-based malware**.
{: .prompt-info }

---

## Procedure

The official documentation for CAPEv2 can be found [here](https://capev2.readthedocs.io/en/latest/installation/host/index.html).

### 1. Setting Up the Host Machine

Install Ubuntu 22.04 LTS and verify your hardware configuration in your VM settings.

![Hardware Configuration](/assets/images/cape/hardware-config.png){: w="700" h="400" .shadow }
_Example hardware allocation for the CAPE host_

After installing the host OS, run the following commands to prepare the environment:

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo chmod -R a+rwx /opt/
cd /opt
sudo apt install git -y
git clone [https://github.com/kevoreilly/CAPEv2.git](https://github.com/kevoreilly/CAPEv2.git)
cd CAPEv2/installer
````

### 2\. Installing Hypervisor (KVM)

While you can use any hypervisor, CAPE recommends **KVM**. Before running the script, replace `<WOOT>` placeholders with real hardware patterns to prevent sandbox detection. In a lab, you can use any random 4 characters (e.g., `MJDP`).

```bash
# Replace placeholders and set permissions
sed -i 's/<WOOT>/MJDP/g' kvm-qemu.sh
chmod a+x kvm-qemu.sh
chmod a+x cape2.sh

# Install KVM (Replace <username> with your actual user)
sudo ./kvm-qemu.sh all <username> | tee kvm-qemu.log
sudo reboot
```

After reboot, install the **Virtual Machine Manager**:

```bash
sudo ./kvm-qemu.sh virtmanager <username> | tee kvm-qemu-virt-manager.log
sudo reboot
```

### 3\. Installing CAPE

Run the main installation script:

```bash
sudo ./cape2.sh all cape | tee cape.log
sudo reboot
```

> The `cape2.sh` script creates a user named `cape` with no default login. To run commands as this user, use:
> `sudo su - cape -c /bin/bash`
> {: .prompt-tip }

#### Verify Installation

Check if the CAPE services are running:

```bash
sudo systemctl status cape*
```

![CAPE services](/assets/images/cape/cape-services.png){: w="700" h="400" .shadow }
_Verifying active CAPE services_


To access the web interface, identify your host IP:

```bash
ifconfig
```

![ifconfig](/assets/images/cape/ifconfig.png){: w="700" h="400" .shadow }
_Host ip_

Then, navigate to `http://<YOUR_IP>:8000` in your browser.

![Web ui](/assets/images/cape/initial-dash.png){: w="700" h="400" .shadow }
_The CAPE Web Interface_

-----

### 4\. Dependency Management (Poetry)

CAPE uses **Poetry** for virtual environments. Navigate to the CAPE directory and initialize:

```bash
cd /opt/CAPEv2/
poetry install
poetry env list # Confirm the environment
```

**Optional Dependencies:**

```bash
sudo -u cape poetry run pip install -r extra/optional_dependencies.txt
```

> **Important:** All future CAPE commands must be executed through Poetry:
> `sudo -u cape poetry run python3 cuckoo.py`
> {: .prompt-warning }

-----

### 5\. Setting Up the Linux Guest Machine

Download a Linux ISO (e.g., [Ubuntu 20.04 LTS](https://releases.ubuntu.com/focal/)) and launch the Virtual Machine Manager:

```bash
virt-manager
```

![QEMU](/assets/images/cape/QEMU.png){: w="700" h="400" .shadow }
_QEMU KVM_

1.  Go to **File \> New Virtual Machine**.
2.  Select **Local Install Media (ISO)**.
3.  Allocate at least **4GB RAM** and **25GB Disk**.
4.  Name the VM (e.g., `cuckoo1`).

{: w="500" h="350" }

#### Guest Configuration (32-bit Support)

To analyze 32-bit malware on a 64-bit guest:

```bash
sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386 -y
```

#### Networking & Traffic Tapping

We must route guest traffic to the host's virtual interface (usually `virbr0`).

Identify the host interface:

```bash
ifconfig # Look for virbr0 (e.g., 192.168.122.1)
```

![VIRBR0](/assets/images/cape/virbr0.png){: w="700" h="400" .shadow }
_virbr0_

Identify the guest interface (e.g., `enp1s0`):

![enp1s0](/assets/images/cape/enp1s0.png){: w="700" h="400" .shadow }
_enp1s0_

Run these commands **on the guest machine** to route traffic:

```bash
sudo iptables -t nat -A POSTROUTING -o enp1s0 -s 192.168.122.0/24 -j MASQUERADE
sudo iptables -P FORWARD DROP
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -s 192.168.122.0/24 -j ACCEPT
sudo iptables -A FORWARD -s 192.168.122.0/24 -d 192.168.122.0/24 -j ACCEPT
sudo iptables -A FORWARD -j LOG
```

Verify with `sudo iptables --list`.

![iptables](/assets/images/cape/iptables.png){: w="700" h="400" .shadow }
_iptables_

Test the connection from the **host** using `tcpdump`:

```bash
sudo tcpdump -i virbr0
```

![tcpdump](/assets/images/cape/tcpdump.png){: w="700" h="400" .shadow }
_tcpdump_

#### Installing the CAPE Agent

On the **guest VM**, download and run the automated agent script:

```bash
wget [https://raw.githubusercontent.com/r3shav/CAPEv2/refs/heads/master/extra/linux_agent.sh](https://raw.githubusercontent.com/r3shav/CAPEv2/refs/heads/master/extra/linux_agent.sh)
chmod +x linux_agent.sh
sudo ./linux_agent.sh
```

Reboot the guest. From the host, verify the agent is listening:

```bash
curl <GUEST_IP>:8000
```

![cronjob](/assets/images/cape/cronjob.png){: w="700" h="400" .shadow }
_cronjob_
*A successful curl indicates the agent cronjob is working*

Finally, take a **Snapshot** in Virt-Manager so CAPE can revert the VM after each analysis.

![snapshot](/assets/images/cape/snapshot.png){: w="700" h="400" .shadow }
_snapshot_

-----

### 6\. Altering Configuration Files

Configurations are located in `/opt/CAPEv2/conf`.

![configuration-files](/assets/images/cape/configuration-files.png){: w="700" h="400" .shadow }
_configuration-files_

Modify the following:

  * **cuckoo.conf**: Specify the network adapter IP for routing.
    ![cuckoo-conf](/assets/images/cape/cuckoo-conf.png){: w="700" h="400" .shadow }

  * **auxiliary.conf**: Verify the interface name.
    ![auxiliary](/assets/images/cape/auxiliary.png){: w="700" h="400" .shadow }

  * **kvm.conf**: Define your VM details (name, snapshot).
    ![kvm](/assets/images/cape/kvm.png){: w="700" h="400" .shadow }

  * **processing.conf**: Ensure `strace` is enabled.
    ![processing](/assets/images/cape/processing.png){: w="700" h="400" .shadow }

  * **routing.conf**: Set the internet route and adapter.
    ![routing](/assets/images/cape/routing.png){: w="700" h="400" .shadow }

-----

### 7\. Submitting Samples

You can now submit files through the web portal.

![submission-filedetails](/assets/images/cape/submission-filedetails.png){: w="700" h="400" .shadow }
*Submitting a malware sample*

View the queue and results in the portal:

![portal](/assets/images/cape/portal.png){: w="700" h="400" .shadow }

#### Analysis Results

CAPE provides detailed behavioral and network analysis:

![behavioural](/assets/images/cape/behavioural.png){: w="700" h="400" .shadow }
{: w="700" h="400" }

![network](/assets/images/cape/network.png){: w="700" h="400" .shadow }

```
```