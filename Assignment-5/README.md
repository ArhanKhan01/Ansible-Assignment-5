# Ansible Assignment-5 — SonarQube Ansible Role

## Overview

This project implements a reusable **Ansible Role for installing and configuring SonarQube**.

The role is designed to satisfy the following requirements:

- Install a specific SonarQube version
- Support different operating systems
- Keep configuration values variableized
- Use Jinja2 templates
- Use separate Ansible handlers
- Support Ubuntu
- Support Red Hat Enterprise Linux (RHEL)
- Run the role on Ubuntu only
- Run the role on RHEL only
- Run the role on both operating systems

The deployment uses a direct SonarQube ZIP installation with Java 17. Docker is not used.

---

## Tool Used

**SonarQube**

Configured version:

```text
10.6.0.92116
```

The SonarQube version is controlled through an Ansible variable, making the role reusable for different versions.

---

## Operating Systems

The role supports:

- Ubuntu
- Red Hat Enterprise Linux (RHEL 10)

Ansible detects the operating-system family and automatically loads the appropriate prerequisite tasks.

---

## Project Structure

```text
ansible-assignment-5/
│
├── inventory
├── site.yml
│
└── roles/
    └── sonarqube/
        │
        ├── defaults/
        │   └── main.yml
        │
        ├── handlers/
        │   └── main.yml
        │
        ├── tasks/
        │   ├── main.yml
        │   ├── debian.yml
        │   └── redhat.yml
        │
        ├── templates/
        │   ├── sonar.properties.j2
        │   └── sonarqube.service.j2
        │
        └── vars/
            ├── Debian.yml
            └── RedHat.yml
```

<img width="610" height="549" alt="image" src="https://github.com/user-attachments/assets/a46cfe00-d114-452b-a5fc-890d016ce298" />


---

# File and Folder Description

## `inventory`

Defines the target servers and their groups.

Example:

```ini
[ubuntu]
ubuntu1 ansible_host=YOUR_UBUNTU_PUBLIC_IP ansible_user=ubuntu ansible_ssh_private_key_file=/home/YOUR_USER/.ssh/YOUR_KEY.pem

[redhat]
redhat1 ansible_host=YOUR_RHEL_PUBLIC_IP ansible_user=ec2-user ansible_ssh_private_key_file=/home/YOUR_USER/.ssh/YOUR_KEY.pem

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

<img width="1206" height="205" alt="image" src="https://github.com/user-attachments/assets/a00d5605-4280-41fd-b186-396c976204d6" />


The inventory makes it possible to run the role on Ubuntu, RHEL, or both.

---

## `site.yml`

This is the main Ansible playbook.

```yaml
---
- name: Install and configure SonarQube
  hosts: all
  become: true

  roles:
    - sonarqube
```

The playbook calls the `sonarqube` role.

---

# Role Components

## `defaults/main.yml`

Contains default and configurable variables.

Example:

```yaml
sonarqube_version: "10.6.0.92116"

sonarqube_install_dir: "/opt/sonarqube"

sonarqube_user: "sonarqube"
sonarqube_group: "sonarqube"

sonarqube_web_host: "0.0.0.0"
sonarqube_web_port: 9000

sonarqube_java_home: "/opt/java17"

sonarqube_vm_max_map_count: 524288
sonarqube_file_max: 131072
```

This satisfies the requirement:

> Configuration should be variableized.

For example, the SonarQube port is controlled by:

```yaml
sonarqube_web_port: 9000
```

---

## `tasks/main.yml`

This is the main task file and controls the complete SonarQube deployment.

It performs tasks such as:

- Load OS-specific variables
- Load OS-specific prerequisite tasks
- Create the SonarQube group
- Create the SonarQube user
- Create installation directories
- Configure Linux kernel parameters
- Download SonarQube
- Extract SonarQube
- Set ownership
- Set permissions
- Make `sonar.sh` executable
- Restore SELinux context when enabled
- Generate SonarQube configuration
- Generate the systemd service
- Enable and start SonarQube

---

## `tasks/debian.yml`

Contains prerequisite installation tasks for Ubuntu/Debian-family systems.

Ubuntu uses `apt`.

Required packages include:

```text
OpenJDK 17
unzip
wget
curl
ca-certificates
procps
```

---

## `tasks/redhat.yml`

Contains prerequisite installation tasks for Red Hat-family systems.

The target system used in this assignment is **RHEL 10**.

Because the required OpenJDK package was unavailable through the configured repositories in the target environment, Java 17 is installed using **Amazon Corretto**.

Java is installed under:

```text
/opt/java17
```

The role configures `JAVA_HOME` and the Java executable path accordingly.

---

# Templates

The role uses Jinja2 templates to generate dynamic configuration files.

This satisfies the assignment requirements:

> Try using jinja template.

and:

> It should include Templates for the configuration files with all the dynamic value that needs to be updated.

---

## `templates/sonar.properties.j2`

Generates:

```text
/opt/sonarqube/conf/sonar.properties
```

Example:

```jinja2
# SonarQube configuration
# Managed by Ansible

sonar.web.host={{ sonarqube_web_host }}
sonar.web.port={{ sonarqube_web_port }}
```

The values are taken from `defaults/main.yml`.

No external PostgreSQL database is configured in the final simplified lab implementation.

---

## `templates/sonarqube.service.j2`

Generates:

```text
/etc/systemd/system/sonarqube.service
```

The template dynamically configures:

- SonarQube user
- SonarQube group
- Java path
- SonarQube installation path
- Start command
- Stop command
- Resource limits

Example:

```ini
User={{ sonarqube_user }}
Group={{ sonarqube_group }}

Environment="JAVA_HOME={{ sonarqube_java_home }}"
Environment="SONAR_JAVA_PATH={{ sonarqube_java_path }}"

ExecStart={{ sonarqube_install_dir }}/bin/linux-x86-64/sonar.sh start
ExecStop={{ sonarqube_install_dir }}/bin/linux-x86-64/sonar.sh stop
```

---

# Handlers

Handlers are maintained separately in:

```text
roles/sonarqube/handlers/main.yml
```

Example:

```yaml
---
- name: Reload systemd
  ansible.builtin.systemd:
    daemon_reload: true

- name: Restart SonarQube
  ansible.builtin.systemd:
    name: "{{ sonarqube_service_name }}"
    state: restarted
```

Tasks notify handlers when required.

Example:

```yaml
notify: Restart SonarQube
```

This satisfies the requirement that handlers are maintained separately from tasks.

---

# OS-Specific Variables

## `vars/Debian.yml`

Contains variables specific to Debian-family systems.

Ubuntu is identified by:

```text
ansible_os_family = Debian
```

The Java executable used on Ubuntu is:

```text
/usr/bin/java
```

---

## `vars/RedHat.yml`

Contains variables specific to Red Hat-family systems.

RHEL is identified by:

```text
ansible_os_family = RedHat
```

The Java installation used on RHEL is:

```text
/opt/java17
```

---

# SonarQube User and Group

The role creates a dedicated service account:

```text
User: sonarqube
Group: sonarqube
```

SonarQube runs using this account instead of root.

Installation directory:

```text
/opt/sonarqube
```

---

# Java Configuration

SonarQube 10.6 requires Java 17.

## Ubuntu

Java 17 is installed using the Ubuntu package manager.

The system Java executable is used:

```text
/usr/bin/java
```

## RHEL 10

Java 17 is installed using Amazon Corretto and stored under:

```text
/opt/java17
```

The systemd service is configured to use the correct Java path for the operating system.

---

# Linux and SELinux Configuration

The role configures required Linux kernel parameters:

```text
vm.max_map_count=524288
fs.file-max=131072
```

The role also ensures that the SonarQube startup script is executable:

```text
/opt/sonarqube/bin/linux-x86-64/sonar.sh
```

On RHEL, the role restores the SELinux context for the SonarQube installation when SELinux is enabled.

---

# Version-Specific Installation

The SonarQube version is controlled using:

```yaml
sonarqube_version: "10.6.0.92116"
```

The download URL is generated dynamically:

```yaml
sonarqube_download_url: "https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-{{ sonarqube_version }}.zip"
```

Changing `sonarqube_version` changes the SonarQube version downloaded by the role.

---

# Execution Options

## Run on Ubuntu only

```bash
ansible-playbook -i inventory site.yml --limit ubuntu
```

<img width="1292" height="532" alt="Screenshot 2026-08-30 010347" src="https://github.com/user-attachments/assets/65eaa390-6e06-4fea-8e5b-32b38871fe14" />


## Run on RHEL only

```bash
ansible-playbook -i inventory site.yml --limit redhat
```

<img width="1295" height="530" alt="Screenshot 2026-08-30 011353" src="https://github.com/user-attachments/assets/550306c2-1b58-417f-b801-52e94196d8ec" />


## Run on both Ubuntu and RHEL

```bash
ansible-playbook -i inventory site.yml
```

<img width="1302" height="531" alt="Screenshot 2026-08-30 013725" src="https://github.com/user-attachments/assets/c4934a60-1d22-401c-8112-781fac75b7d2" />


---

# Ansible Connectivity Test

Before running the playbook:

```bash
ansible all -i inventory -m ping
```

Expected result:

```text
ubuntu1 | SUCCESS
redhat1 | SUCCESS
```

<img width="772" height="201" alt="image" src="https://github.com/user-attachments/assets/1bf00e6a-7792-483f-aee8-5fdecf9e8b57" />


---

# Syntax Validation

Validate the playbook:

```bash
ansible-playbook -i inventory site.yml --syntax-check
```

Expected result:

```text
playbook: site.yml
```

<img width="947" height="78" alt="image" src="https://github.com/user-attachments/assets/7854f802-aa0f-47dd-bd4e-04ae4d511d10" />


---

# Browser Access

SonarQube runs on port:

```text
9000
```

## Access SonarQube on Ubuntu

When SonarQube is running on the Ubuntu EC2 instance, open:

```text
http://UBUNTU_PUBLIC_IP:9000
```

<img width="1363" height="729" alt="Screenshot 2026-08-30 010822" src="https://github.com/user-attachments/assets/c5321830-d1aa-47b5-9a27-5318cad244a2" />


Make sure port `9000` is allowed in the Ubuntu EC2 Security Group.

---

## Access SonarQube on Red Hat

When SonarQube is running on the RHEL EC2 instance, open:

```text
http://RHEL_PUBLIC_IP:9000
```

<img width="1365" height="728" alt="Screenshot 2026-08-30 012821" src="https://github.com/user-attachments/assets/32754800-c12b-4392-94b0-ff53da181574" />


Make sure port `9000` is allowed in the RHEL EC2 Security Group.

---

## Important

Do not use an EC2 private IP such as:

```text
172.31.x.x
```

from your local browser.

Use the **Public IPv4 address** of the EC2 instance where SonarQube is running.

The Security Group should allow:

```text
Type: Custom TCP
Port: 9000
Source: My IP
```

For temporary assignment testing:

```text
Source: 0.0.0.0/0
```

---

# Assignment Requirement Mapping

| Assignment Requirement | Implementation |
|---|---|
| Version-specific installation | `sonarqube_version` |
| OS independent | `debian.yml` and `redhat.yml` |
| Configuration variableized | `defaults/main.yml` |
| Jinja2 template | `.j2` files in `templates/` |
| Dynamic configuration | Jinja2 variables |
| Separate handlers | `handlers/main.yml` |
| Ubuntu execution | `--limit ubuntu` |
| RHEL execution | `--limit redhat` |
| Both OS execution | Run without `--limit` |

---

# Final Result

The SonarQube Ansible role provides:

- Version-specific SonarQube installation
- Ubuntu support
- RHEL 10 support
- Java 17 configuration
- Variable-driven configuration
- Jinja2 templates
- Separate handlers
- Systemd service management
- Linux kernel parameter configuration
- Permission handling
- SELinux context handling
- Selective execution using inventory groups

---

# Conclusion

This project demonstrates a reusable and configurable **Ansible SonarQube Role** that can be executed on Ubuntu, RHEL, or both.

The role separates:

```text
Tasks
Variables
Templates
Handlers
OS-specific configuration
```

This makes the role easier to maintain, reuse, and modify for different environments.

**Assignment-5: Completed Successfully.**
