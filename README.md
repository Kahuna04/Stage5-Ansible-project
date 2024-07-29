# Automated Deployment and Configuration with Ansible for Boilerplates

This project automates the deployment and configuration of a boilerplate application using Ansible. It covers cloning the DevOps branch, installing dependencies, configuring PostgreSQL and messaging queues, setting up the application with Nginx reverse proxy and logging.

## Prerequisites

* A Linux server with Ubuntu 22.04 and Python 3.12 installed.
* SSH access to the server.

## Project Structure

The project repository should contain the following:

* `main.yaml`: The Ansible playbook for deployment and configuration.
* `inventory.cfg`: An inventory file defining the target server.
* `README.md`: This documentation file.

## Deployment Steps

1. **Clone the Repository:**
   - Clone the specified branch of your repository to the Linux server.

2. **Configure `inventory.cfg`:**
   - Create an `inventory.cfg` file in the same directory as your playbook.

3. **Run the Ansible Playbook:**
   - Execute the following command on the server:
     ```bash
     ansible-playbook main.yaml -i inventory.cfg
     ```

## Ansible Playbook (`main.yaml`)

```yaml
---
- hosts: hng
  become: yes
  vars:
    pg_password_file: /var/secrets/pg_pw.txt
    pg_user: postgres
    pg_password: securepassword
    pg_database: postgres
    pg_host: localhost
    app_port: 3000
    app_dir: /opt/stage_5b
    log_dir: /var/log/stage_5b

  tasks:
    - name: Ensure hng user exists
      user:
        name: hng
        state: present
        groups: sudo

    - name: Create necessary directories
      file:
        path: "{{ item }}"
        state: directory
        owner: hng
        mode: '0755'
      loop:
        - /var/secrets
        - "{{ log_dir }}"

    - name: Save PostgreSQL credentials
      copy:
        dest: "{{ pg_password_file }}"
        content: "PG_USER={{ pg_user }}\nPG_PASSWORD={{ pg_password }}\nPG_DATABASE={{ pg_database }}\n"
        owner: root
        group: root
        mode: '0600'

    - name: Read PostgreSQL credentials
      slurp:
        src: "{{ pg_password_file }}"
      register: pg_creds

    - name: Set PostgreSQL credentials as variables
      set_fact:
        pg_user: "{{ (pg_creds.content | b64decode).split('\n')[0].split('=')[1] }}"
        pg_password: "{{ (pg_creds.content | b64decode).split('\n')[1].split('=')[1] }}"
        pg_database: "{{ (pg_creds.content | b64decode).split('\n')[2].split('=')[1] }}"

    - name: Update and upgrade apt packages
      apt:
        update_cache: yes
        upgrade: dist

    - name: Install necessary packages
      apt:
        name:
          - git
          - postgresql
          - nginx
          - rabbitmq-server
          - golang
        state: present

    - name: Clone the boilerplate repository
      git:
        repo: https://github.com/hngprojects/hng_boilerplate_golang_web.git
        dest: "{{ app_dir }}"
        version: devops

    - name: Create app.env file
      copy:
        dest: "{{ app_dir }}/app.env"
        content: |
          SERVER_PORT={{ app_port }}
          SERVER_SECRET="mySecretKey"
          SERVER_ACCESSTOKENEXPIREDURATION=2
          REQUEST_PER_SECOND=6
          TRUSTED_PROXIES=["192.168.0.1", "192.168.0.2"]
          EXEMPT_FROM_THROTTLE=["127.0.0.1", "192.168.0.2", "::1"]

          DB_HOST=localhost
          DB_PORT=5432
          DB_CONNECTION=pgsql
          TIMEZONE=Africa/Lagos
          SSLMODE=disable
          USERNAME=postgres
          PASSWORD=securepassword
          DB_NAME=postgres
          MIGRATE=false
        owner: hng
        group: hng
        mode: '0600'

    - name: Install application dependencies
      shell: |
        cd {{ app_dir }}
        go mod tidy
      args:
        chdir: "{{ app_dir }}"
      become: yes
      become_user: hng

    - name: Ensure application runs on port 3000
      systemd:
        name: app
        state: started
        enabled: yes
        restart: always
        exec: |
          cd {{ app_dir }}
          nohup go run main.go > {{ log_dir }}/out.log 2> {{ log_dir }}/error.log &

    - name: Configure Nginx
      copy:
        dest: /etc/nginx/sites-available/default
        content: |
          server {
              listen 80;
              server_name _;
              location / {
                  proxy_pass http://127.0.0.1:{{ app_port }};
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Proto $scheme;
              }
          }
      notify:
        - Reload Nginx

    - name: Ensure log files are owned by hng user
      file:
        path: "{{ item }}"
        owner: hng
        group: hng
        state: touch
        mode: '0644'
      loop:
        - "{{ log_dir }}/error.log"
        - "{{ log_dir }}/out.log"

    - name: Ensure services are running
      service:
        name: "{{ item }}"
        state: started
      loop:
        - postgresql
        - nginx
        - rabbitmq-server

  handlers:
    - name: Reload Nginx
      service:
        name: nginx
        state: reloaded

```

## Example `inventory.cfg`:

```ini
[hng]
hng ansible_host=your_server_ip
```

## Customize `app.env`:

```
SERVER_PORT=3000
# ... other settings ...
```

## Notes:

* Replace placeholders like `https://github.com/hngprojects/hng_boilerplate_golang_web.git` with your actual repository URL.
* Customize the `app.env` file with your application-specific settings.
