# Ansible Lab – VirtualBox Home Lab
 
A hands-on Ansible lab built in VirtualBox using two Linux VMs. The goal was to establish a working control node / managed node relationship with SSH key-based authentication, ad-hoc commands, and a multi-task playbook.
 
---
 
## Environment
 
| Role | OS | IP |
|---|---|---|
| Control Node | Ubuntu Desktop | 192.168.56.102 |
| Managed Node | Linux Mint | 192.168.56.101 |
  
**Authentication:** SSH key-based (no password prompts during automation)
 
---
 
## Setup
 
### 1. Generate SSH Key Pair on Control Node
 
```bash
ssh-keygen -t rsa
```
 
This produces two files:
- `~/.ssh/id_rsa` — private key (stays on control node)
- `~/.ssh/id_rsa.pub` — public key (copied to managed node)
 
### 2. Copy Public Key to Managed Node
 
```bash
ssh-copy-id lo@192.168.56.101
```
 
This drops the public key into `~/.ssh/authorized_keys` on the managed node. When Ansible initiates an SSH session, the managed node checks that file and authenticates against the control node's private key so no password required.
 
### 3. Enable SSH on Managed Node #required if control node cannot reach or connect to managed node
 
```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
sudo ufw allow ssh
sudo ufw status
```
 
### 4. Create Project Directory and Inventory File
 
```bash
mkdir ~/ansible-lab
cd ~/ansible-lab
nano inventory.ini
```
 
**inventory.ini:**
```ini
[managed]
mintnode ansible_host=192.168.56.101 ansible_user=lo
```
 
---
 
## Ad-Hoc Commands
 
Ad-hoc commands let you run one line tasks against managed node which is useful for quick checks and verification without a playbook
 
```bash
# Check uptime on managed node
ansible -i inventory.ini managed -m command -a "uptime"
 
# Check who is logged in
ansible -i inventory.ini managed -m command -a "who"
```
 
---
 
## Playbook
 
**Filename:** `myplaybook.yml`
 
```yaml
---
- name: Ansible Lab Playbook
  hosts: managed
  become: yes
  tasks:
    - name: Install curl
      apt:
        name: curl
        state: present
 
    - name: Create a test file
      copy:
        content: "THE ONE PIECE IS REAL!\n"
        dest: /home/lo/ansible_test.txt
 
    - name: Filter and check system memory
      ansible.builtin.setup:
        filter:
          - 'ansible_*_mb'
```
 
**Run the playbook:**
```bash
ansible-playbook -i inventory.ini myplaybook.yml
```
 
---
 
## Observations
 
- The `apt` module, `copy`, and `ansible.builtin.setup` module are **idempotent** — running the playbook multiple times produces the same result without making unnecessary changes. Ansible checks current state before acting.
- The `free-h` command is **not truly idempotent** - it collects and reports current system memory usage and runs unconditionally to report changing data.
- SSH key-based auth is essential for Ansible automation password prompts break non-interactive execution.
- If the managed node is unreachable, SSH service status and firewall rules should be the first troubleshooting steps.
 
---
 
## Skills Demonstrated

- SSH key generation and passwordless authentication
- Ansible inventory file structure
- Ad-hoc command execution
- Multi-tasking playbook
- Idempotency analysis
