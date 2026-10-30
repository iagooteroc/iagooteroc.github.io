## Overview

I wanted to create a reproducible configuration for all my current and future devices running Arch. With **Ansible**, you can write a set of automated tasks with idempotency in something called a **playbook**.

In a playbook you can define a desired state and, when run, it will make sure the system matches the desired state, even if it is run multiple times.

---
## **Step 1** -- Ansible and inventory

**Installation on Arch Linux:**
```bash
# Install the package
sudo pacman -S ansible
```

Normally, you would define an inventory that contains a list of hosts to manage over SSH. However, I will be running this playbook locally from each device, meaning I can skip this step by including `hosts: localhost` and `connection: local` in the playbook.

---
## **Step 2** -- Variables

I want to define a different set of packages to be installed in each device. For that, I started by creating a file for variables named `groups_var/all.yaml` to describe packages in the different repositories that I use:
```yaml
# Official repository packages (pacman)
pacman_packages:
  base:
    - git
    - dolphin
    - ark
    - unrar
    - firefox
    - vlc
    - okular

# AUR packages
aur_packages:
  base:
    - vscodium

# Flatpack packages
flatpak_packages:
  base:
    - com.rtosta.zapzap
```

These `base` packages will be installed on every device. Now I can create other groups of packages to be installed only on specific devices, for example:
```yaml
pacman_packages:
  base:
    [...]
  gamedev:
    - godot
    - blender
    - gimp
  productivity:
    - libreoffice-still
```

And to describe which groups will be installed on each device, I created a dictionary to map devices to a list of package groups:
```yaml
# Mapping from device type to package groups
device_groups:
  laptop:
    - base
    - productivity
  desktop:
    - base
    - productivity
    - gamedev
```

---
## **Step 3** -- Playbook

The playbook is the format Ansible uses to define a list of tasks. 

To start, I created `playbook.yaml` and included the `become: yes` instruction, which is used to obtain root privileges and is required for pacman:
```yaml
- name: Local Arch Package Installer
  hosts: localhost
  connection: local
  become: yes
```

Then I defined the variable that will be used to decide which packages will be installed:
```yaml
  vars:
    device: "laptop"  # override with -e device=...
```

And lastly, the list of tasks. I use `paru` as my AUR helper, so I need to compile and install it first:
```yaml
  tasks:
    # Ensure base-devel is installed (required for compiling paru)
    - name: Ensure base-devel is present
      community.general.pacman:
        name: base-devel
        state: present

    # Check if paru command exists
    - name: Check if paru is installed
      ansible.builtin.command: which paru
      register: paru_check
      failed_when: false
      changed_when: false

    # If paru is not installed, compile and install it from AUR
    - name: Install paru from AUR if missing
      when: paru_check.rc != 0
      block:
        - name: Create temporary directory for paru build
          ansible.builtin.tempfile:
            state: directory
            suffix: paru
          register: paru_tmp

        - name: Clone paru repository
          ansible.builtin.git:
            repo: https://aur.archlinux.org/paru.git
            dest: "{{ paru_tmp.path }}/paru"
            depth: 1

        - name: Build and install paru
          ansible.builtin.command:
            cmd: makepkg -si --noconfirm
            chdir: "{{ paru_tmp.path }}/paru"
          become: no  # AUR helper cannot be compiled as root
          become_user: "{{ ansible_user_id }}"

        - name: Clean up temporary directory
          ansible.builtin.file:
            path: "{{ paru_tmp.path }}"
            state: absent
```

After that, the `device` variable is used to obtain the list of packages to install with pacman:
```yaml
  tasks:
    [...]
    # Official repository package installation task
    - name: Install official repository packages for device [{{ device }}]
      community.general.pacman:
        name: "{{ device_groups[device] | map('extract', pacman_packages) | flatten }}"
        state: present
      when: device_groups[device] is defined
```

The same is done with AUR, but switching `become_user` to a normal user, because AUR helpers do not allow execution as root:
```yaml
  tasks:
    [...]
    # AUR package installation task
    - name: Install AUR packages for group [{{ install_group }}]
      kewlfft.aur.aur:
        name: "{{ device_groups[device] | map('extract', aur_packages) | flatten }}"
        use: paru
        state: present
      become: yes
      become_user: "{{ ansible_user_id }}"
      when: device_groups[device] is defined
```

For flatpak packages I first make sure flatpak is installed and add the Flathub repository, and then use the same trick to get which packages to install:
```yaml
  tasks:
    [...]
    # Ensure flatpak itself is installed
    - name: Ensure flatpak is installed
      community.general.pacman:
        name: flatpak
        state: present

    # Add Flathub remote (if not already present)
    - name: Add Flathub remote
      community.general.flatpak_remote:
        name: flathub
        state: present
        flatpakrepo_url: https://dl.flathub.org/repo/flathub.flatpakrepo

    # Install Flatpak packages according to group
    - name: Install Flatpak packages for group [{{ install_group }}]
      community.general.flatpak:
        name: "{{ flatpak_packages[install_group] }}"
        state: present
        remote: flathub
      when: flatpak_packages[install_group] is defined
```

---
## **Step 4** -- Requirements and usage

To be able to run the playbook I created a `requirements.yaml` file to describe which collections it depends on. These are the `community.general` for pacman and flatpak packages, and `kewlfft.aur` for the AUR:
```yaml
collections:
  - name: community.general
  - name: kewlfft.aur
```

You need to install them like this:
```bash
ansible-galaxy collection install -r requirements.yaml
```

With that done, the playbook can be run like this:
```bash
ansible-playbook playbook.yml -e "device=laptop" --ask-become-pass
```

Using `--ask-become-pass` will prompt you for the privilege escalation password.

---
## Lessons learned

**Ansible** is great for defining a set of tasks that build your desired state.

#### Future work

- Create users.

- Create directories.

- Configure the shell.

#### Sources

- https://docs.ansible.com/projects/ansible/latest/index.html
- https://github.com/almondheil/ansible-arch