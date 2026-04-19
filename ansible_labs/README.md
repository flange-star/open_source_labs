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
 
### 3. Enable SSH on Managed Node 
 
```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```
### Required if control node cannot reach or connect to managed node
```
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
  vars_files:
    -vault.yml

  tasks:
    - name: Install curl
      apt:
        name: curl
        state: present

    - name: Create a test file
      copy:
        content: "THE ONE PIECE IS REAL!\n"
        dest: /home/lo/ansible_test.txt
    
    - name: Install tree 
      apt:
        name: tree
        state: present
      notify: Confirm tree installed

    - name: Filter and Check system memory
      setup:
        filter:
          - 'ansible_*_mb'
  handlers:
    - name: Confirm tree installed
      debug: 
        msg: "tree was installed & handler fired on state change!"
```
 
**Run the playbook:**
```bash
ansible-playbook -i inventory.ini myplaybook.yml
```
## Handlers
 
Handlers are tasks that only execute when explicitly notified by another task. Unlike regular tasks which run every time the playbook executes, a handler only fires if the task that notified it resulted in an actual state change on the managed node.
 
### How It Works
 
The `Install tree` task uses `notify: Confirm tree installed` to register interest in the handler. If Ansible installs tree because it was not already present, a state change occurs and the handler is flagged. Once all tasks finish, Ansible runs any flagged handlers at the end of the play.
 
If tree is already installed, Ansible skips the install, no state change occurs, and the handler never fires. This is idempotency in action — the playbook behaves differently based on actual system state rather than running blindly every time.
 
### First Run vs. Second Run
 
| Run | tree status | Handler |
|---|---|---|
| First | Not installed → installs | Fires |
| Second | Already installed → skipped | Silent |

---

## Credential Management – Ansible Vault Integration
 
The playbook uses `become: yes` for privilege escalation, which initially required manually entering the sudo password on every run with `--ask-become-pass`. Everytime a command like apt is used in will need sudo access to install the problem and I did not want to keep typing in the sudo password for the managed node everytime the playbook ran. Therefore, I wanted to integrate Ansible Vault to handle credentials securely without human input.
 
### 1. Create the Vault File
 
```bash
ansible-vault create vault.yml
```
 
Inside the vault file:
```yaml
ansible_become_pass: [your managed node password]
```
 
This file is AES-256 encrypted and cannot be read without the vault password.
 
### 2. Create a Vault Password File
 
```bash
echo 'yourvaultpassword' > ~/.vault_pass
chmod 600 ~/.vault_pass
```
 
- you create a password for Ansbile Vault and pass it to the vault_pass file
- Stores the vault password so Ansible can unlock the vault automatically
- `chmod 600` restricts access to your user only at the OS level
  
### 3. Configure ansible.cfg
 
```ini
[defaults]
inventory = inventory.ini
vault_password_file = ~/.vault_pass
remote_user = lo
```
 
Ansible reads this file automatically when executed from the project directory — no manual flags needed.
 
### Before vs. After
 
| | Command |
|---|---|
| Before | `ansible-playbook -i inventory.ini myplaybook.yml --ask-become-pass` |
| After | `ansible-playbook myplaybook.yml` |
 
Same result — fully automated, no passwords typed manually, credentials encrypted at rest.
 
### Security Note
Adding NOPASSWD: ALL to the /etc/sudoers file via visudo was a shortcut that would allow the managed node to run as sudo with no password required at all, but I did not feel this was an appropriate approach to mimic a real production environment as it can lead to privilege escalation exploitation.
 
---
 
## Observations
 
- The `apt` and `copy` modules are idempotent - check current system state before acting and only make changes when necessary and running the playbook multiple times produces    the same result.
- The `ansible.builtin.setup` module are **idempotent** — it is a read-only fact-gathering module that changes nothing on the managed node and running the playbook multiple times produces the same result.
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
- YAML syntax
