# 🧾 Ansible + Docker + Nginx Multi-OS Deployment — Complete Notes

## 📌 Project Overview
This project demonstrates a **multi-node, multi-OS infrastructure automation** using:
- Ansible (configuration management)
- Docker (containerization)
- Nginx (reverse proxy)
- Ansible Vault (secure credentials)

### 🎯 Goal
Deploy a containerized application across:
- Ubuntu
- CentOS
- Amazon Linux

And expose it via:
- Nginx reverse proxy (port 80)
- Backend container (port 8080)

---

## 🏗️ Final Architecture

~~~
Client → Nginx (port 80) → Docker container (port 8080)
~~~

---

## 📁 Project Structure

~~~
ansible-docker-project/
├── inventory/
├── group_vars/
│   └── all/vault.yml
├── roles/
│   ├── common/
│   ├── docker/
│   └── nginx/
├── site.yml
~~~

---

## ⚙️ What We Implemented

### ✅ 1. Common Role
- Installed basic packages
- Set hostname
- Set timezone
- Created deploy user

---

### ✅ 2. Docker Role
- Installed Docker (OS-specific)
- Started and enabled service
- Added user to docker group
- Logged into Docker Hub using Vault credentials
- Pulled image
- Ran container
- Performed health check

---

### ✅ 3. Nginx Role
- Installed Nginx (OS-specific)
- Removed default configs
- Deployed reverse proxy config
- Configured `/health` endpoint
- Restarted and validated service

---

### ✅ 4. Vault Integration
- Stored credentials securely

~~~
ansible-vault create group_vars/all/vault.yml
~~~

Example:

~~~
docker_username: your_username
docker_password: your_password
~~~

---

## 🧠 Major Issues Encountered & Fixes

---

### ❌ Issue 1: SSH / Connectivity Issues
**Symptoms:**
- Permission denied
- Unable to connect to nodes

**Fix:**
- Verified key permissions
- Used correct SSH command:

~~~
ssh -i key.pem user@host
~~~

---

### ❌ Issue 2: Nginx 502 Bad Gateway (CentOS)

**Cause:**
SELinux blocking Nginx → Docker communication

**Fix:**

~~~
setsebool -P httpd_can_network_connect 1
~~~

---

### ❌ Issue 3: `/health` returning 404

**Cause:**
Nginx location precedence issue

Original:

~~~
location / {
    proxy_pass ...
}

location /health {
    return 200 'OK';
}
~~~

**Fix:**
- Exact match
- Place before `/`

~~~
location = /health {
    return 200 'OK';
}
~~~

---

### ❌ Issue 4: Config Not Loaded (Ubuntu)

**Cause:**
Ubuntu does NOT use `conf.d` by default

**Fix:**
Added include in nginx.conf:

~~~
include /etc/nginx/conf.d/*.conf;
~~~

---

### ❌ Issue 5: Config Conflict (Amazon Linux)

**Cause:**
- default.conf + default.d interfering
- multiple server blocks

**Fix:**
- Removed default configs:

~~~
rm -f /etc/nginx/conf.d/default.conf
rm -rf /etc/nginx/default.d
~~~

- Used:

~~~
listen 80 default_server;
server_name _;
~~~

---

### ❌ Issue 6: Ansible `command` module not supporting pipe

**Problem:**

~~~
nginx -T | grep include
~~~

**Fix:**
Use `shell` module:

~~~
ansible all -m shell -a "nginx -T | grep include"
~~~

---

### ❌ Issue 7: docker_login module failed

**Error:**
~~~
No module named docker
~~~

**Cause:**
Docker SDK missing on remote nodes

---

### ❌ Issue 8: pip install failure on Amazon

**Error:**

~~~
Cannot uninstall requests ... installed by rpm
~~~

**Cause:**
System Python package conflict

---

### ✅ FINAL FIX (Best Solution)

👉 Avoid docker SDK completely

Use Docker CLI:

~~~
docker login -u username -p password
~~~

Implemented in Ansible:

~~~
command:
  argv:
    - docker
    - login
    - -u
    - "{{ docker_username }}"
    - -p
    - "{{ docker_password }}"
~~~

✔ No pip  
✔ No SDK  
✔ Works on all OS  

---

## 🔧 Key Learnings

### 📌 1. OS Differences Matter

| OS | Package Manager |
|----|---------------|
| Ubuntu | apt |
| CentOS | yum |
| Amazon Linux | dnf |

---

### 📌 2. Nginx Config Behavior
- Multiple config directories
- Server block conflicts
- Location precedence matters

---

### 📌 3. Ansible Module vs Shell
- command → no pipes
- shell → supports pipes

---

### 📌 4. Dependency Awareness
- Ansible modules require dependencies on remote nodes
- Avoid unnecessary dependencies when possible

---

### 📌 5. Vault Usage
- Never hardcode credentials
- Use encrypted variables

---

## 🧪 Final Validation

~~~
ansible all -m command -a "curl -s http://localhost/health"
~~~

Expected:

~~~
OK
OK
OK
~~~

---

## 🎯 Final Outcome

✔ Multi-OS deployment  
✔ Docker automation  
✔ Nginx reverse proxy  
✔ Secure credential handling  
✔ Fully working health endpoint  

---

## 🚀 Project Status

~~~
Task Completed Successfully
Production Ready Setup Achieved
~~~

---

## 💡 Interview Talking Points

- Multi-OS Ansible role design
- Debugging Nginx issues
- SELinux troubleshooting
- Docker deployment automation
- Vault usage for security
- Module vs CLI trade-offs

---

## 🔚 Conclusion

This project provided **hands-on experience with real-world DevOps problems**, including:
- Cross-platform issues
- Dependency conflicts
- Service debugging
- Infrastructure automation

It demonstrates **practical problem-solving skills**, not just tool usage.

---
