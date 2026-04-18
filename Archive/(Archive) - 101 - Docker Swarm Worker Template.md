#HomeLabRebuild/Docker #HomeLabRebuild/Proxmox 

# Proxmox VM Template Prep for Docker Swarm Workers 

- [x] **1. Create Base VM** ✅ 2025-08-16 #todoist
    - [x] Minimal Linux distro (Debian/Ubuntu LTS) ✅ 2025-08-16 #todoist
    - [x] CPU: ≥2 cores ✅ 2025-08-16 #todoist
    - [x] RAM: 4–8GB (adjust per workload) ✅ 2025-08-16 #todoist
    - [x] Disk: 20–40GB (template only) ✅ 2025-08-16 #todoist
    - [x] Install OS and updates ✅ 2025-08-16 #todoist

- [x] **2. Install Cloud-init** ✅ 2025-08-16 #todoist
    - [x] `sudo apt update && sudo apt upgrade -y` ✅ 2025-08-16 #todoist
    - [x] `sudo apt install cloud-init -y` ✅ 2025-08-16 #todoist
    - [x] Enable service: `sudo systemctl enable cloud-init` ✅ 2025-08-16 #todoist
    - [x] Clean previous cloud-init data: `sudo cloud-init clean --logs` ✅ 2025-08-16 #todoist

- [x] **3. Configure Networking** ✅ 2025-08-16 #todoist
    - [x] Verify primary network interface: ✅ 2025-08-16 #todoist
        ```bash
        ip link show
        ```
    - [x] Set DHCP on the interface: ✅ 2025-08-16 #todoist
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
    - [x] Verify connectivity: ✅ 2025-08-16 #todoist
        ```bash
        ip a show <interface_name>
        ping -c 3 8.8.8.8
        ```
    - [x] Leave VLAN blank (provisioning script will apply VLAN) ✅ 2025-08-16 #todoist
    - [x] Optional: install VLAN support: ✅ 2025-08-16 #todoist
        ```bash
        sudo apt install vlan -y
        sudo modprobe 8021q
        ```

- [x] **4. Optional: Pre-install Docker** ✅ 2025-08-16 #todoist
	- [x] `sudo apt install apt-transport-https ca-certificates curl software-properties-common ` ✅ 2025-08-16 #todoist
	- [x] `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg` ✅ 2025-08-16 #todoist
    - [x] `sudo apt install -y docker.io` ✅ 2025-08-16 #todoist
    - [x] `sudo apt install docker ✅ 2025-08-16 #todoist
    - [x] `sudo apt install docker-compose` ✅ 2025-08-16 #todoist
    - [x] Enable Docker on boot: `sudo systemctl enable docker` ✅ 2025-08-16 #todoist

- [x] **5. Enable SSH Access** ✅ 2025-08-16 #todoist
    - [x] Ensure SSH server is installed and running: ✅ 2025-08-16 #todoist
        ```bash
        sudo apt update
        sudo apt install -y openssh-server
        sudo systemctl enable ssh
        sudo systemctl start ssh
        ```
    - [x] Verify SSH service status: ✅ 2025-08-16 #todoist
        ```bash
        sudo systemctl status ssh
        ```
    - [x] Ensure your SSH key exists for cloud-init injection: ✅ 2025-08-16 #todoist
        ```bash
        ls $HOME/.ssh/id_rsa.pub
        ```
        - [x] If it does not exist, create it: ✅ 2025-08-16 #todoist
        ```bash
        ssh-keygen -t rsa -b 4096 -f $HOME/.ssh/id_rsa -N ""
        ```
    - [x] Test SSH login from another machine: ✅ 2025-08-16 #todoist
        ```bash
        ssh <user>@<vm_ip>
        ```
    - [x] Optional: disable password authentication for extra security: ✅ 2025-08-16 #todoist
        ```bash
        sudo nano /etc/ssh/sshd_config
        # Set:
        PasswordAuthentication no
        ```
        Then reload SSH:
        ```bash
        sudo systemctl reload ssh
        ```

- [x] **6. Cleanup & Prepare Template** ✅ 2025-08-16 
    - [x] Remove SSH host keys: `sudo rm -rf /etc/ssh/ssh_host_*` ✅ 2025-08-16 
    - [x] Clean cloud-init logs: `sudo cloud-init clean --logs` ✅ 2025-08-16 
    - [x] Remove logs/temp: `sudo rm -rf /var/log/* /tmp/*` ✅ 2025-08-16 
- [x] **7. Convert VM to Template** #todoist ✅ 2025-08-22
    - [x] Shutdown VM: `qm shutdown <VMID>` ✅ 2025-08-16 
    - [x] Convert to template: `qm template <VMID>` ✅ 2025-08-16 #todoist
    - [x] Note the template ID (e.g., 103) #todoist ✅ 2025-08-22

- [x] **8. Test Template** #todoist ✅ 2025-08-22
    - [ ] Clone a test VM manually #todoist
    - [ ] Check SSH key injection #todoist
    - [ ] Verify VLAN tagging and DHCP #todoist
    - [ ] Verify Docker starts #todoist
    - [ ] Optional: test NFS mount manually #todoist

