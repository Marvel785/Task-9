# Docker Install Playbook Explanation

This document explains every step that happens when you run the `docker-install-playbook.yml`.

## Playbook Header

```yaml
- name: Setup ansible user and install Docker on Ubuntu VMs
  hosts: prod
  become: yes
  vars:
    docker_compose_version: "2.21.0"
```

- **hosts: prod** - Targets all machines in the `[prod]` group from host.ini
- **become: yes** - Runs tasks with sudo privileges
- **vars** - Sets Docker Compose version to download

## Phase 1: System Preparation

### 1. Update Package Cache
```yaml
- name: Update apt package cache
  apt:
    update_cache: yes
    cache_valid_time: 3600
```
**What happens:** Refreshes Ubuntu's package list (like `sudo apt update`). The `cache_valid_time: 3600` means it won't update again if cache is less than 1 hour old.

## Phase 2: Ansible User Setup

### 2. Create Ansible User
```yaml
- name: Create ansible user
  user:
    name: ansible
    shell: /bin/bash
    home: /home/ansible
    createhome: yes
    state: present
```
**What happens:** Creates a new user named `ansible` with bash shell and home directory at `/home/ansible`.

### 3. Add to Sudo Group
```yaml
- name: Add ansible user to sudo group
  user:
    name: ansible
    groups: sudo
    append: yes
```
**What happens:** Adds the ansible user to the `sudo` group, giving it administrative privileges.

### 4. Create SSH Directory
```yaml
- name: Create .ssh directory for ansible user
  file:
    path: /home/ansible/.ssh
    state: directory
    owner: ansible
    group: ansible
    mode: '0700'
```
**What happens:** Creates `.ssh` directory with proper permissions (read/write/execute for owner only).

### 5. Copy SSH Keys
```yaml
- name: Copy authorized_keys from current user to ansible user
  copy:
    src: "/home/{{ ansible_user }}/.ssh/authorized_keys"
    dest: /home/ansible/.ssh/authorized_keys
    owner: ansible
    group: ansible
    mode: '0600'
    remote_src: yes
```
**What happens:** Copies SSH keys from current user (user1/user2) to ansible user so you can SSH as ansible later. `{{ ansible_user }}` gets replaced with `user1` or `user2` from host.ini.

### 6. Setup Passwordless Sudo
```yaml
- name: Add ansible user to sudoers with NOPASSWD
  lineinfile:
    path: /etc/sudoers.d/ansible
    create: yes
    line: 'ansible ALL=(ALL) NOPASSWD:ALL'
    mode: '0440'
    validate: 'visudo -cf %s'
```
**What happens:** Allows ansible user to run sudo commands without entering a password. The `validate` ensures the sudoers file syntax is correct before saving.

## Phase 3: Docker Prerequisites

### 7. Install Required Packages
```yaml
- name: Install required packages for Ubuntu
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
      - software-properties-common
    state: present
```
**What happens:** Installs packages needed for Docker installation (HTTPS transport, certificates, curl for downloads, etc.).

### 8. Create Keyrings Directory
```yaml
- name: Create keyrings directory
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'
```
**What happens:** Creates directory for storing GPG keys (modern Ubuntu security practice).

### 9. Download Docker GPG Key
```yaml
- name: Add Docker's official GPG key
  get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: '0644'
```
**What happens:** Downloads Docker's official GPG key to verify package authenticity.

### 10. Add Docker Repository
```yaml
- name: Add Docker repository
  apt_repository:
    repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present
```
**What happens:** Adds Docker's official Ubuntu repository to your system. `{{ ansible_distribution_release }}` gets replaced with your Ubuntu version (jammy, focal, etc.).

## Phase 4: Docker Installation

### 11. Install Docker Packages
```yaml
- name: Install Docker CE on Ubuntu
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present
    update_cache: yes
```
**What happens:** Installs Docker Community Edition, CLI tools, container runtime, and plugins.

### 12. Install Standalone Docker Compose
```yaml
- name: Install Docker Compose (standalone)
  get_url:
    url: "https://github.com/docker/compose/releases/download/v{{ docker_compose_version }}/docker-compose-linux-x86_64"
    dest: /usr/local/bin/docker-compose
    mode: '0755'
```
**What happens:** Downloads Docker Compose binary (version 2.21.0) and makes it executable.

### 13. Create Docker Compose Symlink
```yaml
- name: Create docker-compose symlink
  file:
    src: /usr/local/bin/docker-compose
    dest: /usr/bin/docker-compose
    state: link
```
**What happens:** Creates a symlink so `docker-compose` command works from anywhere.

### 14. Start Docker Service
```yaml
- name: Start and enable Docker service
  systemd:
    name: docker
    state: started
    enabled: yes
```
**What happens:** Starts Docker service immediately and enables it to start automatically on boot.

### 15. Add Ansible User to Docker Group
```yaml
- name: Add ansible user to docker group
  user:
    name: ansible
    groups: docker
    append: yes
```
**What happens:** Allows ansible user to run Docker commands without sudo.

## Phase 5: File Management

### 16. Create Docker Directory
```yaml
- name: Create docker directory
  file:
    path: /home/ansible/docker
    state: directory
    owner: ansible
    group: ansible
    mode: '0755'
```
**What happens:** Creates `/home/ansible/docker/` directory for storing Docker-related files.

### 17. Copy Dockerfile
```yaml
- name: Copy Dockerfile
  copy:
    src: ./Dockerfile
    dest: /home/ansible/docker/Dockerfile
    owner: ansible
    group: ansible
    mode: '0644'
```
**What happens:** Copies your Dockerfile from the control machine to each VM's `/home/ansible/docker/` directory.

## Phase 6: Verification

### 18. Test Docker Installation
```yaml
- name: Verify Docker installation
  command: docker --version
  register: docker_version
  become_user: ansible
```
**What happens:** Runs `docker --version` as the ansible user and stores the output in `docker_version` variable.

### 19. Display Results
```yaml
- name: Display Docker version
  debug:
    msg: "Docker installed successfully: {{ docker_version.stdout }}"
```
**What happens:** Shows the Docker version output to confirm successful installation.

## Summary

After this playbook runs successfully on both VMs:

1. ✅ **ansible user created** with sudo access and SSH keys
2. ✅ **Docker installed** and running
3. ✅ **Docker Compose installed** (both plugin and standalone)
4. ✅ **Your Dockerfile copied** to both VMs
5. ✅ **Everything verified** and ready to use

You can now SSH to either VM as the ansible user and run Docker commands without sudo:
```bash
ssh ansible@192.168.112.128
docker --version
docker-compose --version
cd docker
cat Dockerfile
```