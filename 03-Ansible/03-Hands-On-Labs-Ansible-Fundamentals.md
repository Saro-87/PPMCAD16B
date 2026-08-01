# Ansible: Hands-On Labs
## Ansible Fundamentals & Playbooks

**Prerequisites**: create 2 ubuntu ec2 servers of type t3.small on AWS.
Name one as "Control Node" and another as "Managed Node"

Control Node is where we install the Ansible server and Managed Node/s are the ones which are managed by Ansible.

---

## Lab 0: Setup

### 0.1 Install Ansible on "Control Node"

```bash
# Update package manager
sudo apt update

# Install Ansible (latest)
sudo apt install -y ansible

# Verify installation
ansible --version
```

### 0.2 Generate SSH Key (if not already done)

```bash
# Create SSH key
ssh-keygen -t rsa

# Display public key (copy for next step)
cat ~/.ssh/id_rsa.pub
```

### 0.3 Configure "Managed Nodes" (1 Ubuntu VMs)

On each managed node:
```bash
# Add your control node's SSH public key
echo "YOUR_PUBLIC_KEY" >> ~/.ssh/authorized_keys

# Set permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## Lab 1: Create Inventory

### 1.1 Create inventory file

Create `inventory.yml`:
```yaml
---
all:
  vars:
    ansible_user: ubuntu
    ansible_key_file: ~/.ssh/id_rsa

  children:
    webservers:
      hosts:
        web1:
          ansible_host: 10.0.1.10 # provide the actual IP of the managed nodes
# an example when managing multiple nodes, add when you provision 2 managed nodes
    dbservers:
      hosts:
        db1:
          ansible_host: 10.0.1.20 # provide the actual IP of the managed nodes
```

### 1.2 Test connectivity

```bash
# Test all hosts
ansible all -i inventory.yml -m ping

# Expected output:
# web1 | SUCCESS => {
#     "ping": "pong"
# }
# ... (for all hosts)
```

---

## Lab 2: Ad-Hoc Commands

### Run shell commands

```bash
# Get uptime on webservers
ansible webservers -i inventory.yml -m shell -a 'uptime'

# Get free disk space
ansible all -i inventory.yml -m shell -a 'df -h /'

# Check OS version
ansible all -i inventory.yml -m shell -a 'cat /etc/os-release'
```

---

## Lab 3: First Playbook

### 3.1 Create basic playbook

Create `install-nginx.yml`:
```yaml
---
- name: Install and start Nginx
  hosts: webservers
  become: yes
  vars:
    nginx_port: 80

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Start Nginx service
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Check if Nginx is running
      shell: systemctl is-active nginx
      register: nginx_status

    - name: Print Nginx status
      debug:
        msg: "Nginx status: {{ nginx_status.stdout }}"
```

### 3.2 Run the playbook

```bash
# Dry-run (--check): shows what would change without making changes
ansible-playbook -i inventory.yml install-nginx.yml --check

# Run for real
ansible-playbook -i inventory.yml install-nginx.yml

# Run with verbose output
ansible-playbook -i inventory.yml install-nginx.yml -v

# Even more verbose (shows SSH details)
ansible-playbook -i inventory.yml install-nginx.yml -vv
```

### 3.3 Verify idempotency

Run the playbook again:
```bash
ansible-playbook -i inventory.yml install-nginx.yml
```

**Expected**: All tasks show "ok" (no changes), demonstrating idempotency.

---

## Lab 4: Handlers

### 4.1 Create playbook with handlers

Create `nginx-config.yml`:
```yaml
---
- name: Configure Nginx
  hosts: webservers
  become: yes

  tasks:
    - name: Copy Nginx configuration
      copy:
        src: ./nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: restart nginx

    - name: Create web directory
      file:
        path: /var/www/myapp
        state: directory
        owner: www-data
        group: www-data

    - name: Create index page
      copy:
        content: "<h1>Hello from {{ inventory_hostname }}</h1>"
        dest: /var/www/myapp/index.html
        owner: www-data
        group: www-data

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

### 4.2 Create nginx.conf file

Create `nginx.conf`:
```
user www-data;
worker_processes auto;

events {
    worker_connections 512;
}

http {
    server {
        listen 80;
        server_name _;

        location / {
            root /var/www/myapp;
            index index.html;
        }
    }
}
```

### 4.3 Run and test

```bash
# Run the playbook
ansible-playbook -i inventory.yml nginx-config.yml

# Open the port 80 on the security group of the "Managed Node" and open the public ip of the server on the browser
curl http://<public_ip_of_managed_node>

# Expected output: <h1>Hello from web1</h1>

# Verify handler runs once (run playbook again, change config)
# Handler only runs if notify is triggered
```

---

## Lab 5: Variables & Jinja2 Templates

### 5.1 Create template file

Create `index.html.j2`:
```html
<!DOCTYPE html>
<html>
<head>
    <title>{{ app_name }}</title>
</head>
<body>
    <h1>{{ app_name }} on {{ inventory_hostname }}</h1>
    <p>Server IP: {{ ansible_default_ipv4.address }}</p>
    <p>OS: {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
    <ul>
        {% for item in features %}
        <li>{{ item }}</li>
        {% endfor %}
    </ul>
    <p>Environment: {{ environment }}</p>
</body>
</html>
```

### 5.2 Create playbook with variables

Create `deploy-app.yml`:
```yaml
---
- name: Deploy application with templates
  hosts: webservers
  become: yes

  vars:
    app_name: "My Production App"
    environment: "production"
    features:
      - "FastAPI Backend"
      - "PostgreSQL Database"
      - "Redis Cache"

  tasks:
    - name: Deploy index page from template
      template:
        src: index.html.j2
        dest: /var/www/myapp/index.html
        owner: www-data
        group: www-data
      notify: reload nginx

    - name: Display deployment info
      debug:
        msg: |
          Deployed {{ app_name }} to {{ inventory_hostname }}
          Environment: {{ environment }}
          Features: {{ features | join(', ') }}

  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
```

### 5.3 Run and verify

```bash
# Run playbook
ansible-playbook -i inventory.yml deploy-app.yml

# Check rendered template
ansible web1 -i inventory.yml -m shell -a 'cat /var/www/myapp/index.html'

# Test in browser
```

---