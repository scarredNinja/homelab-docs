---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
# Proxmox VM Template Prep for Docker Swarm Workers 

- **1. Create Base VM**
    - Minimal Linux distro (Debian/Ubuntu LTS) 
    - CPU: ≥2 cores 
    - RAM: 4–8GB (adjust per workload) 
    -  Disk: 20–40GB (template only)
    - Install OS and updates

- 2. Install Cloud-init
    - `sudo apt update && sudo apt upgrade -y` 
    - `sudo apt install cloud-init -y`
    - Enable service: `sudo systemctl enable cloud-init`
    - Clean previous cloud-init data: `sudo cloud-init clean --logs`

 **3. Configure Networking**
    - Verify primary network interface:
        ```bash
        ip link show
        ```
    - Set DHCP on the interface:
        - **Debian/Ubuntu pre-20.04** (`/etc/network/interfaces`):
            ```text
            auto <interface_name>
            iface <interface_name> inet dhcp
            ```
        - **Ubuntu 20.04+** (`netplan`):
            ```yaml
            network:
              version: 2
              ethernets:
                <interface_name>:
                  dhcp4: true
            ```
            Apply with:
            ```bash
            sudo netplan apply
            ```
    - Verify connectivity: 
        ```bash
        ip a show <interface_name>
        ping -c 3 8.8.8.8
        ```
    - Leave VLAN blank (provisioning script will apply VLAN) 
    - Optional: install VLAN support: 
        ```bash
        sudo apt install vlan -y
        sudo modprobe 8021q
        ```

-x **4. Optional: Pre-install Docker**✅ 2025-08-27
    -  Update package index and install prerequisites:
        ```bash
        sudo apt update
        sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
        ```
    - Install Docker Engine: 
        ```bash
        sudo apt install -y docker.io
        ```
    -  Enable Docker to start on boot: 
        ```bash
        sudo systemctl enable docker
        sudo systemctl start docker
        ```
    - Verify Docker installation:
        ```bash
        docker --version
        sudo systemctl status docker
        ```
    - Optional: install Docker Compose:
        ```bash
        sudo apt install -y docker-compose
        ```
    - Optional: add your cloud-init user to the Docker group: 
        ```bash
        sudo usermod -aG docker <CI_USER>
        ```


- **5. Enable SSH Access** 
    - Ensure SSH server is installed and running: 
        ```bash
        sudo apt update
        sudo apt install -y openssh-server
        sudo systemctl enable ssh
        sudo systemctl start ssh
        ```
    - Verify SSH service status:
        ```bash
        sudo systemctl status ssh
        ```
    - Test SSH login from another machine: 
        ```bash
        ssh <user>@<vm_ip>
        ```
    -  Ensure your public key exists for cloud-init injection:
        ```bash
        ls $HOME/.ssh/id_rsa.pub
        ```
    - Optional: disable password authentication for extra security: 
        ```bash
        sudo nano /etc/ssh/sshd_config
        # Set:
        PasswordAuthentication no
        ```
        Then reload SSH:
        ```bash
        sudo systemctl reload ssh
        ```


- **6. Cleanup & Prepare Template** 
    - Remove SSH host keys: `sudo rm -rf /etc/ssh/ssh_host_*` 
    - Clean cloud-init logs: `sudo cloud-init clean --logs` 
    - Remove logs/temp: `sudo rm -rf /var/log/* /tmp/*`

- **7. Convert VM to Template**
    - Shutdown VM: `qm shutdown <VMID>`
    - Convert to template: `qm template <VMID>` 
    - Note the template ID (e.g., 103)

- **8. Test Template** 
    - Clone a test VM manually 
    - Check SSH key injection 
    - Verify VLAN tagging and DHCP 
    - Verify Docker starts 
    - Optional: test NFS mount manually
