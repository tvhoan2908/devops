# 🐍 Tài liệu Ansible: Từ Zero đến Chuyên gia

> **Mục tiêu**: Sau khi hoàn thành tài liệu này, bạn có thể tự tin thiết kế, triển khai và vận hành hạ tầng tự động với Ansible ở quy mô production.
>
> **Cách học**: Mỗi bước có lệnh thực thi + file mẫu. Chạy thử trên VM Linux/macOS/WSL2.

---

## 📑 Mục lục

| Phần | Nội dung | Thời gian |
|------|----------|-----------|
| [P0. Chuẩn bị môi trường](#p0) | Cài Ansible, SSH, lab | 30 phút |
| [P1. Nền tảng](#p1) | Inventory, Ad-hoc, Playbook, Module | 3 giờ |
| [P2. Biến & Facts](#p2) | Variable, Fact, Magic Variable | 2 giờ |
| [P3. Cấu trúc nâng cao](#p3) | Handler, When, Loop, Register, Block | 3 giờ |
| [P4. Template & File](#p4) | Jinja2, Template, Lookup | 2 giờ |
| [P5. Role](#p5) | Cấu trúc role chuẩn | 3 giờ |
| [P6. Vault & Secret](#p6) | Mã hoá, file riêng biệt | 2 giờ |
| [P7. Dynamic Inventory](#p7) | AWS, GCP, Azure, K8s | 2 giờ |
| [P8. Galaxy & Collection](#p8) | Dùng lại, đóng gói | 2 giờ |
| [P9. Testing](#p9) | Molecule, CI/CD | 3 giờ |
| [P10. Production](#p10) | Ansible Tower/AAP, Scale | 4 giờ |
| [P11. Troubleshooting](#p11) | Debug thực chiến | 3 giờ |

---

<a id="p0"></a>
## P0. Chuẩn bị môi trường

### Bước 1: Cài đặt Ansible

#### 🅰️ Linux (Ubuntu/Debian)
```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible ansible-lint
```

#### 🅱️ macOS
```bash
brew install ansible ansible-lint
```

#### 🅲️ pip (khuyến nghị - cùng version mọi nơi)
```bash
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
pip install ansible ansible-core ansible-lint molecule molecule-docker docker
```

#### 🅳️ Windows (WSL2)
```bash
# Trong WSL2 Ubuntu
sudo apt update && sudo apt install -y ansible ansible-lint
```

### Bước 2: Tạo lab với Vagrant

```bash
mkdir -p ~/ansible-lab && cd ~/ansible-lab
```
```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.define "control" do |c|
    c.vm.hostname = "control"
    c.vm.network "private_network", ip: "192.168.56.10"
    c.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y ansible sshpass
    SHELL
  end

  %w[web db].each_with_index do |name, i|
    config.vm.define name do |node|
      node.vm.hostname = name
      node.vm.network "private_network", ip: "192.168.56.#{20+i}"
      node.vm.provision "shell", inline: <<-SHELL
        apt-get update
        apt-get install -y python3
      SHELL
    end
  end
end
```
```bash
vagrant up
vagrant ssh control
```

### Bước 3: Cấu hình SSH key (khuyến nghị)

```bash
# Trên control node
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# Copy sang các node (cần sshpass nếu dùng password)
ssh-copy-id vagrant@192.168.56.20
ssh-copy-id vagrant@192.168.56.21

# Hoặc dùng password qua inventory
```

### Bước 4: Ansible.cfg - cấu hình toàn cục

```ini
# ~/ansible-lab/ansible.cfg
[defaults]
inventory = ./inventory
roles_path = ./roles
collections_path = ./collections
remote_tmp = ~/.ansible/tmp
forks = 10
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
deprecation_warnings = False
interpreter_python = auto_silent

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ServerAliveInterval=60
```

### Bước 5: Kiểm tra cài đặt

```bash
ansible --version
ansible-config dump --only-changed

# Tạo inventory đầu tiên
cat > inventory <<EOF
[web]
192.168.56.20

[db]
192.168.56.21

[all:vars]
ansible_user=vagrant
ansible_ssh_private_key_file=~/.ssh/id_ed25519
EOF

# Ping test
ansible all -m ping
```
**Output kỳ vọng:**
```
192.168.56.20 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
192.168.56.21 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

**📚 Kiến thức nền tảng:**
- **Control node**: Máy cài Ansible (Linux/macOS)
- **Managed node**: Máy mục tiêu được quản lý (cần Python 2.7+ / 3.5+)
- **Inventory**: Danh sách các managed node
- **Playbook**: File YAML mô tả tác vụ
- **Module**: Đơn vị xử lý (file, copy, apt, yum, service...)

---

<a id="p1"></a>
## P1. Nền tảng

### Bước 1: Inventory - "Danh bạ" các server

```bash
# inventory (INI format)
[web]
web1.example.com
web2.example.com
web3.example.com ansible_host=192.168.56.20

[db]
db1.example.com ansible_port=2222
db2.example.com

[production:children]
web
db

[web:vars]
http_port=80
ansible_user=deploy
```

```yaml
# inventory.yaml (YAML format - khuyến nghị)
all:
  children:
    web:
      hosts:
        web1.example.com:
        web2.example.com:
        web3.example.com:
          ansible_host: 192.168.56.20
    db:
      hosts:
        db1.example.com:
          ansible_port: 2222
        db2.example.com:
    production:
      children:
        web:
        db:
    web:
      vars:
        http_port: 80
        ansible_user: deploy
```

```bash
# Lệnh quản lý inventory
ansible-inventory --list                # Xem cấu trúc JSON
ansible-inventory --graph               # Xem dạng cây
ansible-inventory --host web1.example.com  # Xem vars của 1 host
ansible all --list-hosts                # List tất cả hosts
ansible web --list-hosts                # List hosts trong group web
```

### Bước 2: Ad-hoc command - Lệnh nhanh

```bash
# Cú pháp: ansible <pattern> -m <module> -a "<args>"

# Ping
ansible all -m ping

# Chạy lệnh shell
ansible web -m shell -a "uptime"
ansible web -m command -a "df -h"        # command không có pipe/redirect
ansible web -m shell -a "cat /etc/os-release | grep PRETTY"

# Cài gói
ansible web -m apt -a "name=nginx state=present" --become
ansible db -m yum -a "name=postgresql state=present" --become

# Quản lý service
ansible web -m service -a "name=nginx state=started enabled=yes" --become

# Copy file
ansible all -m copy -a "src=./motd dest=/etc/motd mode=0644"

# Gather facts
ansible web -m setup | less             # Xem tất cả facts
ansible web -m setup -a "filter=ansible_eth*"

# Limit
ansible web -m shell -a "uptime" --limit web1.example.com

# Check mode (dry-run)
ansible web -m apt -a "name=nginx state=latest" --check --diff --become
```

### Bước 3: Playbook đầu tiên

```bash
mkdir -p ~/ansible-lab/p1 && cd ~/ansible-lab/p1
```
```yaml
# playbook.yaml
---
- name: Cài đặt Nginx cho web servers
  hosts: web
  become: yes
  gather_facts: yes

  tasks:
    - name: Cập nhật apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Cài Nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Đảm bảo Nginx đang chạy
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy file index.html
      ansible.builtin.copy:
        src: files/index.html
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'

    - name: Mở port 80
      ansible.builtin.ufw:
        rule: allow
        port: '80'
        proto: tcp
```
```html
<!-- files/index.html -->
<!DOCTYPE html>
<html>
<head><title>Welcome</title></head>
<body>
  <h1>Hello from {{ ansible_hostname }}</h1>
  <p>Managed by Ansible</p>
</body>
</html>
```

```bash
# Chạy playbook
ansible-playbook playbook.yaml

# Chạy với check mode
ansible-playbook playbook.yaml --check --diff

# Chạy với syntax check
ansible-playbook playbook.yaml --syntax-check

# Chạy với step-by-step (confirm mỗi task)
ansible-playbook playbook.yaml --step

# Chạy từ task cụ thể
ansible-playbook playbook.yaml --start-at-task="Deploy file index.html"

# Giới hạn hosts
ansible-playbook playbook.yaml --limit web1

# Tags
ansible-playbook playbook.yaml --tags "config"
ansible-playbook playbook.yaml --skip-tags "package"
```

### Bước 4: Thêm Tags cho task

```yaml
# playbook.yaml
- name: Cài đặt Nginx
  hosts: web
  become: yes
  tasks:
    - name: Cài Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
      tags:
        - package
        - install

    - name: Copy config
      ansible.builtin.copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      tags: config

    - name: Khởi động Nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
      tags: service
```

### Bước 5: Modules hay dùng nhất

```yaml
# Package management
- ansible.builtin.apt: name=nginx state=present
- ansible.builtin.yum: name=httpd state=present
- ansible.builtin.dnf: name=podman state=latest
- ansible.builtin.pip: name=django state=present
- ansible.builtin.npm: name=express global=yes

# File operations
- ansible.builtin.copy: src=./file dest=/tmp/file
- ansible.builtin.file: path=/tmp/x state=directory mode=0755
- ansible.builtin.file: path=/tmp/x state=absent
- ansible.builtin.lineinfile: path=/etc/hosts line="1.2.3.4 host1"
- ansible.builtin.replace: path=/etc/x regexp='^#' replace=''
- ansible.builtin.fetch: src=/etc/passwd dest=./backup/

# Service
- ansible.builtin.service: name=nginx state=started enabled=yes
- ansible.builtin.systemd: name=nginx state=restarted

# User management
- ansible.builtin.user: name=deploy groups=sudo shell=/bin/bash
- ansible.builtin.group: name=deploy state=present

# Command
- ansible.builtin.command: uptime         # Không có shell features
- ansible.builtin.shell: cat /etc/*release | head -1  # Có shell features
- ansible.builtin.script: ./script.sh    # Chạy script local trên remote
```

### Bước 6: Cú pháp YAML cơ bản

```yaml
---
# Comment
string: "Hello"
string_no_quotes: Hello
integer: 42
float: 3.14
boolean: true       # hoặc yes
null: ~
list:
  - item1
  - item2
  - item3
dictionary:
  key1: value1
  key2: value2
multiline: |
  Dòng 1
  Dòng 2
folded: >
  Dòng 1
  Dòng 2  # Nối với dòng 1 bằng space
```

### 🎯 Bài tập P1
1. Tạo inventory 3 group (web, db, lb) với 6 hosts
2. Viết playbook cài nginx trên web, postgresql trên db
3. Copy 1 file config từ control sang tất cả host
4. Chạy với `--check --diff` xem thay đổi gì trước khi apply

---

<a id="p2"></a>
## P2. Biến & Facts

### Bước 1: Khai báo biến

```yaml
# vars/playbook-vars.yaml
- name: Demo biến
  hosts: web
  vars:
    http_port: 80
    https_port: 443
    app_name: "MyApp"
    packages:
      - nginx
      - htop
      - vim
  tasks:
    - name: In biến
      ansible.builtin.debug:
        msg: "App {{ app_name }} listens on port {{ http_port }}"
```

### Bước 2: Các cấp ưu tiên biến (thấp → cao)

```
1. command line values      (--extra-vars)
2. role defaults            (defaults/main.yml)
3. inventory file or script group vars
4. inventory group_vars/all
5. playbook group_vars/all
6. inventory group_vars/*
7. playbook group_vars/*
8. inventory file or script host vars
9. inventory host_vars/*
10. playbook host_vars/*
11. host facts / cached set_facts
12. play vars
13. play vars_prompt
14. play vars_files
15. role vars (vars/main.yml)
16. block vars (block vars only)
17. task vars (only for the task)
18. include_vars
19. set_facts / registered vars
20. role (and include_role) params
21. include params
22. extra vars (always win precedence)
```

### Bước 3: group_vars & host_vars

```bash
# Cấu trúc thư mục
inventory/
├── production/
│   ├── hosts.yml
│   ├── group_vars/
│   │   ├── all.yml
│   │   ├── web.yml
│   │   └── db.yml
│   └── host_vars/
│       ├── web1.example.com.yml
│       └── db1.example.com.yml
└── staging/
    ├── hosts.yml
    └── group_vars/
```
```yaml
# inventory/production/group_vars/all.yml
---
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org
timezone: Asia/Ho_Chi_Minh
admin_email: [email protected]
```
```yaml
# inventory/production/group_vars/web.yml
---
http_port: 80
https_port: 443
nginx_user: www-data
```
```yaml
# inventory/production/host_vars/web1.example.com.yml
---
server_id: 1
is_primary: true
```

### Bước 4: Facts tự động

```bash
# Facts là thông tin Ansible tự thu thập từ host
ansible web -m setup | head -50
ansible web -m setup -a "filter=ansible_distribution*"
ansible web -m setup -a "filter=ansible_memtotal_mb"
ansible web -m setup -a "filter=ansible_processor*"
```
```yaml
# Dùng facts trong playbook
- name: Dùng facts
  hosts: web
  tasks:
    - name: Cấu hình sysctl dựa trên memory
      ansible.posix.sysctl:
        name: vm.swappiness
        value: "{{ 10 if ansible_memtotal_mb > 8192 else 30 }}"
        state: present

    - name: In hostname
      ansible.builtin.debug:
        msg: "Host {{ ansible_hostname }} chạy {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

### Bước 5: Magic Variables

```yaml
# Các biến đặc biệt Ansible cung cấp sẵn
- name: Demo magic variables
  ansible.builtin.debug:
    msg: |
      Hostname: {{ inventory_hostname }}
      Short name: {{ inventory_hostname_short }}
      Group: {{ group_names }}
      All groups: {{ groups.keys() | list }}
      Playbook dir: {{ playbook_dir }}
      Role path: {{ role_path }}
      Home dir: {{ ansible_env.HOME }}
      User: {{ ansible_user_id }}
```

### Bước 6: Set biến runtime

```yaml
# set_fact - tạo biến mới trong runtime
- name: Set fact
  ansible.builtin.set_fact:
    is_production: "{{ env == 'production' }}"
    app_version: "1.2.3"
    server_full_name: "{{ ansible_hostname }}.{{ domain }}"
```

### Bước 7: Truyền biến qua command line

```bash
# --extra-vars / -e
ansible-playbook playbook.yaml -e "env=production version=2.0"
ansible-playbook playbook.yaml -e @vars.json
ansible-playbook playbook.yaml -e @vars.yaml
```
```json
// vars.json
{
  "env": "production",
  "version": "2.0",
  "replicas": 5
}
```

### Bước 8: vars_files & vars_prompt

```yaml
# vars_files
- name: Demo
  hosts: web
  vars_files:
    - vars/common.yaml
    - vars/{{ env }}.yaml   # Tùy biến theo env
```
```yaml
# vars_prompt - hỏi khi chạy
- name: Demo
  hosts: web
  vars_prompt:
    - name: db_password
      prompt: "Enter DB password"
      private: yes
      encrypt: sha512_crypt
      confirm: yes
```

### 🎯 Bài tập P2
1. Tạo group_vars/all với 5 biến chung
2. Tạo host_vars riêng cho từng server
3. Dùng set_fact tính toán biến dựa trên facts
4. Chạy playbook với `-e` override biến

---

<a id="p3"></a>
## P3. Cấu trúc nâng cao

### Bước 1: Handlers - Task chạy khi có thay đổi

```yaml
# playbook.yaml
- name: Nginx với handler
  hosts: web
  become: yes
  tasks:
    - name: Copy nginx config
      ansible.builtin.copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: Restart Nginx   # Gọi handler khi file thay đổi

    - name: Copy site config
      ansible.builtin.copy:
        src: site.conf
        dest: /etc/nginx/sites-available/default
      notify: Reload Nginx

    - name: Đảm bảo Nginx đang chạy
      ansible.builtin.service:
        name: nginx
        state: started

  handlers:
    - name: Restart Nginx
      ansible.builtin.service:
        name: nginx
        state: restarted

    - name: Reload Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded

    - name: Multiple actions
      ansible.builtin.service:
        name: nginx
        state: restarted
      listen: "Restart Nginx"     # Cho phép gọi từ nhiều task
```

### Bước 2: When - Điều kiện

```yaml
- name: Điều kiện
  hosts: all
  become: yes
  tasks:
    # Theo OS family
    - name: Cài Apache trên Debian
      ansible.builtin.apt:
        name: apache2
        state: present
      when: ansible_os_family == "Debian"

    - name: Cài Apache trên RedHat
      ansible.builtin.dnf:
        name: httpd
        state: present
      when: ansible_os_family == "RedHat"

    # Theo group
    - name: Cài Docker chỉ trên web
      ansible.builtin.apt:
        name: docker.io
        state: present
      when: "'web' in group_names"

    # Multiple conditions
    - name: Cấu hình production
      ansible.builtin.copy:
        src: prod.conf
        dest: /etc/myapp/config.conf
      when:
        - env == "production"
        - ansible_memtotal_mb >= 4096
        - not ansible_check_mode

    # Theo biến có giá trị
    - name: Enable SSL
      ansible.builtin.copy:
        src: ssl.conf
        dest: /etc/nginx/ssl.conf
      when: enable_ssl | default(false) | bool

    # Theo file tồn tại
    - name: Chạy nếu file chưa tồn tại
      ansible.builtin.command: /usr/bin/init.sh
      when: not my_initialized.stat.exists
      vars:
        my_initialized: "{{ ansible_check_mode | default(false) }}"
```

### Bước 3: Loops - Lặp

```yaml
- name: Loop cơ bản
  hosts: web
  become: yes
  tasks:
    # Cài nhiều package
    - name: Cài packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
      loop:
        - nginx
        - htop
        - vim
        - git

    # Loop với dict
    - name: Tạo users
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        shell: "{{ item.shell }}"
      loop:
        - { name: 'alice', groups: 'sudo', shell: '/bin/bash' }
        - { name: 'bob',   groups: 'users', shell: '/bin/zsh' }

    # Loop với named keys
    - name: Copy files
      ansible.builtin.copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
      loop:
        - src: file1.txt
          dest: /tmp/file1.txt
        - src: file2.txt
          dest: /tmp/file2.txt
      loop_control:
        label: "{{ item.dest }}"      # Label hiển thị trong output

    # loop: dict -> key/value
    - name: Show dict
      ansible.builtin.debug:
        msg: "{{ item.key }} = {{ item.value }}"
      loop: "{{ http_settings | dict2items }}"

    # until - retry
    - name: Đợi service ready
      ansible.builtin.uri:
        url: http://localhost:8080/health
        status_code: 200
      register: result
      until: result.status == 200
      retries: 5
      delay: 10
```

### Bước 4: Register - Lưu kết quả

```yaml
- name: Register
  hosts: web
  tasks:
    - name: Lấy uptime
      ansible.builtin.command: uptime
      register: uptime_result

    - name: In kết quả
      ansible.builtin.debug:
        msg: "Uptime: {{ uptime_result.stdout }}"

    - name: Kiểm tra file
      ansible.builtin.stat:
        path: /etc/nginx/nginx.conf
      register: nginx_conf

    - name: In thông tin
      ansible.builtin.debug:
        msg: "File exists: {{ nginx_conf.stat.exists }}, size: {{ nginx_conf.stat.size }}"

    - name: Tạo user nếu chưa có
      ansible.builtin.user:
        name: deploy
        state: present
      register: user_result

    - name: Hiển thị nếu thay đổi
      ansible.builtin.debug:
        msg: "User đã được tạo"
      when: user_result.changed
```

### Bước 5: Block & Rescue - Xử lý lỗi

```yaml
- name: Block / Rescue / Always
  hosts: web
  become: yes
  tasks:
    - name: Khối xử lý
      block:
        - name: Cài package
          ansible.builtin.apt:
            name: nginx
            state: present

        - name: Copy config
          ansible.builtin.copy:
            src: nginx.conf
            dest: /etc/nginx/nginx.conf
          notify: Restart Nginx

        - name: Khởi động service
          ansible.builtin.service:
            name: nginx
            state: started

      rescue:
        - name: Ghi log nếu fail
          ansible.builtin.debug:
            msg: "Có lỗi xảy ra, đang rollback..."

        - name: Rollback
          ansible.builtin.copy:
            src: nginx.conf.bak
            dest: /etc/nginx/nginx.conf
          when: nginx_conf_backup.stat.exists

      always:
        - name: Luôn chạy
          ansible.builtin.debug:
            msg: "Hoàn thành khối xử lý"

    - name: Backup config trước
      ansible.builtin.stat:
        path: /etc/nginx/nginx.conf.bak
      register: nginx_conf_backup
```

### Bước 6: failed_when & changed_when

```yaml
# Kiểm soát khi nào task fail / thay đổi
- name: Custom status
  hosts: web
  tasks:
    - name: Chạy lệnh
      ansible.builtin.command: /opt/myscript.sh
      register: script_result
      failed_when: "'ERROR' in script_result.stderr"
      changed_when: "'CHANGED' in script_result.stdout"

    # Hoặc không bao giờ fail
    - name: Best-effort command
      ansible.builtin.command: /opt/optional.sh
      failed_when: false
```

### Bước 7: ignore_errors & any_errors_fatal

```yaml
- name: Error handling
  hosts: web
  tasks:
    - name: Có thể fail nhưng tiếp tục
      ansible.builtin.command: /opt/risky.sh
      ignore_errors: yes

    - name: Nếu task này fail thì dừng cả play
      ansible.builtin.command: /opt/critical.sh
      any_errors_fatal: true

    # Force fail
    - name: Luôn fail (dùng cho test)
      ansible.builtin.fail:
        msg: "Luôn luôn fail"
```

### Bước 8: Delegate & Local Action

```yaml
- name: Delegate
  hosts: web
  tasks:
    # Chạy task trên host khác
    - name: Cập nhật load balancer
      ansible.builtin.command: /opt/reload-lb.sh
      delegate_to: loadbalancer.example.com

    # Chạy trên control node
    - name: Lấy thông tin local
      ansible.builtin.setup:
      delegate_to: localhost
      run_once: true

    # run_once - chỉ chạy 1 lần
    - name: Tạo database (1 lần duy nhất)
      ansible.builtin.command: createdb myapp
      delegate_to: db1.example.com
      run_once: true
```

### 🎯 Bài tập P3
1. Tạo playbook có handler restart nginx khi config đổi
2. Loop cài 5 packages, tạo 3 users
3. Register output, dùng để debug
4. Block/Rescue: thử xoá file quan trọng, tự động khôi phục

---

<a id="p4"></a>
## P4. Template & File

### Bước 1: Jinja2 cơ bản

```jinja2
{# templates/nginx.conf.j2 #}
worker_processes {{ ansible_processor_vcpus }};

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
}

http {
    upstream backend {
    {% for host in groups['web'] %}
        server {{ hostvars[host]['ansible_host'] | default(host) }}:8080;
    {% endfor %}
    }

    server {
        listen {{ http_port | default(80) }};
        server_name {{ server_name | default('_') }};

        {% if enable_ssl | default(false) %}
        listen 443 ssl;
        ssl_certificate {{ ssl_cert_path }};
        ssl_certificate_key {{ ssl_key_path }};
        {% endif %}

        location / {
            proxy_pass http://backend;
        }
    }
}
```

### Bước 2: Jinja2 Filters

```jinja2
{# Biến đổi chuỗi #}
{{ "hello" | upper }}                        -> HELLO
{{ "HELLO" | lower }}                        -> hello
{{ "hello world" | title }}                  -> Hello World
{{ "  hello  " | trim }}                     -> hello
{{ "hello" | replace("h", "H") }}            -> Hello
{{ "/etc/nginx" | basename }}                -> nginx

{# List #}
{{ [3, 1, 2] | sort }}                       -> [1, 2, 3]
{{ [3, 1, 2] | max }}                        -> 3
{{ [1, 2, 3] | sum }}                        -> 6
{{ [1, 2, 3] | length }}                     -> 3
{{ [1, 2, 3] | join(',') }}                  -> 1,2,3
{{ [1, 2, 3] | first }}                      -> 1

{# Dict #}
{{ mydict | dict2items }}                    -> list of items
{{ items | items2dict }}                     -> dict from items

{# Default & required #}
{{ var | default('default_value') }}
{{ var | mandatory }}                        -> fail nếu không có

{# Conditional #}
{{ value | bool }}
{{ value | string }}
{{ value | int }}
{{ value | float }}

{# JSON/YAML #}
{{ var | to_json }}
{{ var | to_yaml }}
{{ var | to_nice_json }}
{{ var | to_nice_yaml }}

{# Network #}
{{ "192.168.1.10/24" | ipaddr('address') }}  -> 192.168.1.10
{{ "192.168.1.10" | ipaddr('host') }}        -> true
{{ ips | ipaddr('host') | list }}             -> filter list IPs

{# Path #}
{{ '/etc/nginx/nginx.conf' | basename }}     -> nginx.conf
{{ '/etc/nginx' | dirname }}                 -> /etc

{# Custom filter #}
{{ now() | strftime('%Y-%m-%d') }}
```

### Bước 3: Template module

```yaml
- name: Deploy nginx config
  hosts: web
  become: yes
  vars:
    http_port: 80
    server_name: example.com
    enable_ssl: true
  tasks:
    - name: Deploy template
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
        validate: 'nginx -t -c %s'    # Test trước khi ghi
      notify: Reload Nginx
```

### Bước 4: Template vòng lặp & điều kiện

```jinja2
{# templates/sshd_config.j2 #}
# {{ ansible_managed }}

Port {{ ssh_port | default(22) }}
ListenAddress {{ ansible_default_ipv4.address }}

# Authentication
{% if permit_root_login | default(false) %}
PermitRootLogin yes
{% else %}
PermitRootLogin no
{% endif %}

PasswordAuthentication {{ 'yes' if password_auth else 'no' }}

# Allowed users
{% for user in allowed_users %}
AllowUsers {{ user }}
{% endfor %}

# Allowed groups
AllowGroups {% for group in allowed_groups %}{{ group }} {% endfor %}

# Banner
Banner /etc/issue.net
```

### Bước 5: Lookup plugins

```yaml
# file lookup
- name: Read file
  ansible.builtin.debug:
    msg: "{{ lookup('file', 'secret.txt') }}"

# env lookup
- name: Env var
  ansible.builtin.debug:
    msg: "{{ lookup('env', 'HOME') }}"

# pipe - chạy command
- name: Random string
  ansible.builtin.debug:
    msg: "{{ lookup('pipe', 'date +%Y%m%d') }}"

# url
- name: Fetch URL
  ansible.builtin.debug:
    msg: "{{ lookup('url', 'https://api.ipify.org') }}"

# password - generate random
- name: Random password
  ansible.builtin.set_fact:
    db_password: "{{ lookup('password', '/tmp/password.txt chars=ascii_letters,digits length=16') }}"

# with_fileglob - loop over files
- name: Copy all configs
  ansible.builtin.copy:
    src: "{{ item }}"
    dest: "/etc/myapp/{{ item | basename }}"
  with_fileglob:
    - configs/*.conf

# with_file - line by line
- name: Process hosts
  ansible.builtin.debug:
    msg: "Host: {{ item }}"
  with_lines: cat hosts.txt

# first_found - tìm file đầu tiên tồn tại
- name: Use first existing file
  ansible.builtin.include_vars: "{{ item }}"
  with_first_found:
    - "vars/{{ env }}.yml"
    - "vars/default.yml"
```

### Bước 6: lineinfile & replace

```yaml
- name: Edit file
  hosts: web
  become: yes
  tasks:
    # Thay dòng có pattern
    - name: Set max clients in nginx.conf
      ansible.builtin.lineinfile:
        path: /etc/nginx/nginx.conf
        regexp: '^#?\s*worker_connections\s+\d+'
        line: 'worker_connections 2048;'
        state: present

    # Thêm dòng vào sau một pattern
    - name: Add custom header
      ansible.builtin.lineinfile:
        path: /etc/nginx/conf.d/custom.conf
        line: "add_header X-Frame-Options DENY;"
        insertafter: "^server {"
        create: yes

    # Xoá dòng
    - name: Remove dangerous directive
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        state: absent

    # Replace toàn bộ file
    - name: Replace timezone
      ansible.builtin.replace:
        path: /etc/timezone
        regexp: '^.*$'
        replace: 'Asia/Ho_Chi_Minh'

    # Thêm block nếu chưa có
    - name: Add block
      ansible.builtin.blockinfile:
        path: /etc/hosts
        block: |
          {{ item.ip }} {{ item.name }}
        marker: "# {mark} ANSIBLE MANAGED BLOCK {{ item.name }}"
      loop:
        - { ip: "10.0.0.1", name: "gateway" }
        - { ip: "10.0.0.2", name: "monitoring" }
```

### 🎯 Bài tập P4
1. Tạo template nginx.conf dùng facts của host
2. Template sshd_config với điều kiện bật/tắt root login
3. Dùng lineinfile thêm 3 entry vào /etc/hosts
4. Dùng lookup lấy IP public của host hiện tại

---

<a id="p5"></a>
## P5. Role - Cấu trúc chuẩn

### Bước 1: Tạo Role

```bash
# Cách 1: Dùng ansible-galaxy init
ansible-galaxy init roles/nginx
```

```bash
# Cách 2: Tạo thủ công (linh hoạt hơn)
mkdir -p roles/nginx/{defaults,handlers,meta,files,templates,tasks,vars}
touch roles/nginx/{defaults,handlers,meta,tasks,vars}/main.yml
```

```bash
# Cấu trúc hoàn chỉnh
roles/
└── nginx/
    ├── defaults/         # Biến mặc định (priority thấp nhất)
    │   └── main.yml
    ├── files/            # File tĩnh
    ├── handlers/         # Handlers
    │   └── main.yml
    ├── meta/             # Metadata (dependencies, platforms)
    │   └── main.yml
    ├── tasks/            # Tasks chính
    │   └── main.yml
    ├── templates/        # Jinja2 templates
    ├── tests/            # Test
    │   ├── inventory
    │   └── test.yml
    └── vars/             # Biến ưu tiên cao
        └── main.yml
```

### Bước 2: Role nginx hoàn chỉnh

```yaml
# roles/nginx/defaults/main.yml
---
nginx_user: www-data
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_listen_port: 80
nginx_ssl_enabled: false
nginx_server_name: _
nginx_document_root: /var/www/html
```
```yaml
# roles/nginx/vars/main.yml
---
nginx_packages_debian:
  - nginx
  - nginx-common
nginx_packages_redhat:
  - nginx
```
```yaml
# roles/nginx/handlers/main.yml
---
- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded

- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```
```yaml
# roles/nginx/tasks/main.yml
---
- name: Include OS-specific install
  ansible.builtin.include_tasks: install-{{ ansible_os_family }}.yml

- name: Copy nginx config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: 'nginx -t -c %s'
  notify: Reload nginx

- name: Copy site config
  ansible.builtin.template:
    src: site.conf.j2
    dest: /etc/nginx/sites-available/default
  notify: Reload nginx

- name: Deploy index.html
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ nginx_document_root }}/index.html"
    mode: '0644'

- name: Ensure nginx is started
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: yes
```
```yaml
# roles/nginx/tasks/install-Debian.yml
---
- name: Install nginx on Debian
  ansible.builtin.apt:
    name: "{{ nginx_packages_debian }}"
    state: present
    update_cache: yes
```
```yaml
# roles/nginx/tasks/install-RedHat.yml
---
- name: Install nginx on RedHat
  ansible.builtin.dnf:
    name: "{{ nginx_packages_redhat }}"
    state: present
```
```yaml
# roles/nginx/meta/main.yml
---
galaxy_info:
  author: Your Name
  description: Nginx role
  company: Your Company
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - focal
    - name: EL
      versions:
        - "9"
dependencies: []
```
```jinja2
# roles/nginx/templates/nginx.conf.j2
worker_processes {{ nginx_worker_processes }};

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    sendfile on;

    server {
        listen {{ nginx_listen_port }};
        server_name {{ nginx_server_name }};
        root {{ nginx_document_root }};

        {% if nginx_ssl_enabled %}
        listen 443 ssl;
        ssl_certificate /etc/nginx/ssl/server.crt;
        ssl_certificate_key /etc/nginx/ssl/server.key;
        {% endif %}
    }
}
```
```jinja2
# roles/nginx/templates/index.html.j2
<h1>Hello from {{ ansible_hostname }}</h1>
<p>Server: {{ ansible_fqdn }}</p>
```

### Bước 3: Dùng role trong playbook

```yaml
# site.yaml
---
- name: Setup web servers
  hosts: web
  become: yes
  roles:
    - role: nginx
      vars:
        nginx_listen_port: 8080
        nginx_server_name: "{{ ansible_fqdn }}"
    - role: geerlingguy.firewall
      tags: firewall

- name: Setup database
  hosts: db
  become: yes
  roles:
    - postgresql
    - backup
```

### Bước 4: Role dependencies

```yaml
# roles/myapp/meta/main.yml
dependencies:
  - role: common
    vars:
      some_var: foo
  - role: nginx
    vars:
      nginx_listen_port: 8080
  - role: postgres
    when: db_type == 'postgres'
```

### Bước 5: include_role & import_role

```yaml
# include_role - chạy động (dynamic)
- name: Conditional role
  ansible.builtin.include_role:
    name: nginx
  when: install_nginx | default(false)

# import_role - tải tĩnh (static)
- name: Static role
  ansible.builtin.import_role:
    name: nginx
  tasks:
    - name: After role
      ansible.builtin.debug:
        msg: "after nginx role"
```

### Bước 6: include_tasks & import_tasks

```yaml
# import_tasks - tĩnh (điều kiện áp dụng cho toàn file)
- name: Common tasks
  ansible.builtin.import_tasks: common.yml
  when: is_production

# include_tasks - động (điều kiện check từng task)
- name: Dynamic tasks
  ansible.builtin.include_tasks: setup.yml
  when: not is_configured
```

### 🎯 Bài tập P5
1. Tạo role "common" cài các package cơ bản
2. Tạo role "nginx" có dependency từ common
3. Tạo role cho ứng dụng của bạn (vd: deploy 1 web app)
4. Kết hợp 3 role trong 1 playbook

---

<a id="p6"></a>
## P6. Vault & Secret

### Bước 1: Tạo file mã hoá

```bash
# Tạo vault file
ansible-vault create secrets.yml
# Mở editor, nhập nội dung YAML, lưu lại
# Sẽ hỏi password
```
```yaml
# secrets.yml (đã mã hoá)
---
db_password: "S3cr3tP@ssw0rd"
api_key: "sk-1234567890abcdef"
ssh_private_key: |
  -----BEGIN RSA PRIVATE KEY-----
  ...
  -----END RSA PRIVATE KEY-----
```

```bash
# Xem file đã mã hoá
ansible-vault view secrets.yml

# Sửa file
ansible-vault edit secrets.yml

# Đổi password
ansible-vault rekey secrets.yml

# Mã hoá file có sẵn
ansible-vault encrypt existing_file.yml

# Giải mã
ansible-vault decrypt secrets.yml
```

### Bước 2: Chạy playbook với vault

```bash
# Cách 1: Hỏi password
ansible-playbook playbook.yaml --ask-vault-pass

# Cách 2: File password
echo "myVaultPass123" > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt
ansible-playbook playbook.yaml --vault-password-file ~/.vault_pass.txt

# Cách 3: Biến môi trường (ít an toàn)
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass.txt
ansible-playbook playbook.yaml

# Cách 4: Nhiều vault với nhiều password
ansible-playbook playbook.yaml \
  --vault-id dev@~/.vault_dev.txt \
  --vault-id prod@prompt

# Cách 5: Script custom
cat > ~/.vault_pass.py <<EOF
#!/usr/bin/env python3
import os, sys
# Lấy password từ keyring, vault, secret manager, etc.
print(os.environ.get('VAULT_PASSWORD', 'default'))
EOF
ansible-playbook playbook.yaml --vault-password-file ~/.vault_pass.py
```

### Bước 3: Dùng vault trong playbook

```yaml
# vault.yml (file riêng, đã mã hoá)
---
vault_db_user: admin
vault_db_password: "S3cr3tP@ssw0rd"
vault_api_key: "sk-abc123"
```
```yaml
# playbook.yaml
- name: Use vault
  hosts: web
  become: yes
  vars_files:
    - vars/common.yml
    - vault.yml    # File đã mã hoá

  tasks:
    - name: Create database
      community.postgresql.postgresql_user:
        name: "{{ vault_db_user }}"
        password: "{{ vault_db_password }}"
      delegate_to: db1.example.com

    - name: Deploy app with API key
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/myapp/app.conf
        mode: '0600'
```

### Bước 4: Vault tích hợp HashiCorp Vault

```bash
# Cài collection
ansible-galaxy collection install community.hashi_vault

# Cấu hình plugin
cat > ~/.ansible.cfg <<EOF
[defaults]
vault_identity_list = file:///path/to/client.jwt

[hashi_vault]
url = https://vault.example.com:8200
auth_method = approle
role_id = abc123
secret_id = def456
EOF
```
```yaml
# Dùng trong playbook
- name: Get secret from Vault
  ansible.builtin.set_fact:
    db_password: "{{ lookup('community.hashi_vault.vault_read', 'secret/data/db') }}"
```

### Bước 5: Single Encrypted Variable

```bash
# Mã hoá 1 string
ansible-vault encrypt_string 'MySecretPassword' --name 'db_password'
```
```yaml
# Output
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  66386439653...
```
```yaml
# Hoặc inline trong playbook
- name: Use encrypted var
  ansible.builtin.debug:
    msg: "{{ db_password }}"
  vars:
    db_password: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
```

### Bước 6: Best Practice bảo mật

```bash
# .gitignore cho repo ansible
cat > .gitignore <<EOF
*.vault
*_vault.yml
.vault_pass*
group_vars/all/vault.yml
host_vars/*/vault.yml
EOF

# KHÔNG BAO GIỜ commit vault password
# KHÔNG BAO GIỜ commit secret chưa mã hoá
# Dùng pre-commit hook check secret
```

### 🎯 Bài tập P6
1. Tạo vault.yml với 3 secret, mã hoá
2. Chạy playbook dùng --ask-vault-pass
3. Tạo file password, dùng --vault-password-file
4. Đặt permission 600 cho file chứa password

---

<a id="p7"></a>
## P7. Dynamic Inventory

### Bước 1: Tự viết dynamic inventory (script)

```bash
cat > inventory/dynamic.py <<'EOF'
#!/usr/bin/env python3
import json
import sys

def main():
    inventory = {
        "_meta": {
            "hostvars": {
                "web1.example.com": {"ansible_host": "10.0.0.1"},
                "web2.example.com": {"ansible_host": "10.0.0.2"},
                "db1.example.com":  {"ansible_host": "10.0.1.1"},
            }
        },
        "web": {
            "hosts": ["web1.example.com", "web2.example.com"],
            "vars": {"http_port": 80}
        },
        "db": {
            "hosts": ["db1.example.com"],
            "vars": {"db_port": 5432}
        },
        "production": {
            "children": ["web", "db"]
        }
    }
    print(json.dumps(inventory))

if __name__ == "__main__":
    if len(sys.argv) == 2 and sys.argv[1] == "--list":
        main()
    elif len(sys.argv) == 3 and sys.argv[1] == "--host":
        print(json.dumps({}))
    else:
        sys.exit(1)
EOF

chmod +x inventory/dynamic.py

# Test
ansible-inventory -i inventory/dynamic.py --list
ansible all -i inventory/dynamic.py -m ping
```

### Bước 2: Dynamic inventory AWS

```bash
# Cài
pip install boto3

# Cấu hình AWS
export AWS_ACCESS_KEY_ID=xxx
export AWS_SECRET_ACCESS_KEY=xxx
export AWS_REGION=us-east-1

# Plugin có sẵn
cp /usr/lib/python3/dist-packages/ansible/plugins/inventory/aws_ec2.yml .
# hoặc tải
curl -O https://raw.githubusercontent.com/ansible/ansible/stable-2.15/contrib/inventory/aws_ec2.yml

# File cấu hình
cat > aws_ec2.yml <<'EOF'
---
plugin: aws_ec2
regions:
  - us-east-1
  - us-west-2
keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: tags.Role
    prefix: role
  - key: instance_state
    prefix: state
hostnames:
  - tag:Name
  - dns-name
compose:
  ansible_host: public_ip_address
groups:
  running: instance_state == "running"
  stopped: instance_state == "stopped"
filters:
  tag:Project: myproject
EOF

# Dùng
ansible-inventory -i aws_ec2.yml --graph
ansible env_production -i aws_ec2.yml -m ping
```

### Bước 3: Dynamic inventory GCP

```bash
pip install google-cloud-compute

cat > gcp.yml <<'EOF'
---
plugin: gcp_compute
projects:
  - my-project-id
keyed_groups:
  - key: labels.environment
    prefix: env
  - key: labels.role
    prefix: role
hostnames:
  - name
compose:
  ansible_host: networkInterfaces[0].accessConfigs[0].natIP
groups:
  redhat: "'rhel' in name or 'centos' in name"
EOF

ansible-inventory -i gcp.yml --list
```

### Bước 4: Plugin constructed (kết hợp static + facts)

```yaml
# inventory/01-static.yml
---
plugin: yaml
hosts:
  web1.example.com:
  web2.example.com:
  web3.example.com:
  db1.example.com:
  db2.example.com:

# inventory/02-constructed.yml
---
plugin: constructed
strict: false
groups:
  webservers: inventory_hostname in groups['web']
  postgres_hosts: "'postgres' in (hostvars[inventory_hostname]['ansible_facts']['services'] | default({}) | dict2items | map(attribute='key') | list)"
  high_memory: hostvars[inventory_hostname]['ansible_memtotal_mb'] >= 8192
```

### Bước 5: Plugin cho Kubernetes

```bash
pip install kubernetes

cat > k8s.yml <<EOF
---
plugin: kubernetes
connections:
  - kubeconfig: ~/.kube/config
      context: production
EOF

# Dùng để quản lý K8s nodes bằng Ansible
ansible all -i k8s.yml -m k8s_info
```

### Bước 6: Multiple inventory sources

```bash
# Ansible tự động merge nhiều inventory
ansible-playbook site.yaml -i inventory/

# Cấu trúc thư mục
inventory/
├── 01-aws.yml          # Thứ tự quan trọng: 01-, 02-
├── 02-static.yml
└── 03-constructed.yml
```

### 🎯 Bài tập P7
1. Viết dynamic inventory trả về 5 host chia 2 group
2. Cấu hình AWS EC2 plugin, list hosts
3. Dùng constructed plugin tạo group theo OS

---

<a id="p8"></a>
## P8. Galaxy & Collection

### Bước 1: Cài từ Galaxy

```bash
# Role
ansible-galaxy role install geerlingguy.nginx
ansible-galaxy role install geerlingguy.docker
ansible-galaxy role install -r requirements.yml

# Collection
ansible-galaxy collection install community.general
ansible-galaxy collection install community.docker
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install kubernetes.core

# Cài từ file requirements
cat > requirements.yml <<EOF
roles:
  - name: geerlingguy.nginx
    version: 3.1.0
  - src: https://github.com/user/custom-role.git
    scm: git

collections:
  - name: community.general
    version: 7.0.0
  - name: community.docker
    source: https://galaxy.ansible.com
EOF

ansible-galaxy install -r requirements.yml
```

### Bước 2: Dùng FQCN (Fully Qualified Collection Name)

```yaml
# Nên dùng FQCN để rõ ràng và ổn định
tasks:
  # Thay vì: apt:
  - name: Install nginx
    ansible.builtin.apt:
      name: nginx
      state: present

  # Thay vì: service:
  - name: Restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted

  # Collection bên ngoài
  - name: Manage EC2
    amazon.aws.ec2_instance:
      name: web1
      instance_type: t3.micro
      image_id: ami-xxx
      region: us-east-1

  - name: Manage K8s
    kubernetes.core.k8s:
      definition:
        apiVersion: v1
        kind: Pod
        metadata:
          name: mypod
```

### Bước 3: Tạo Collection riêng

```bash
# Khởi tạo
ansible-galaxy collection init mycompany.mycollection

# Cấu trúc
mycompany-mycollection/
├── galaxy.yml
├── README.md
├── meta/
│   └── runtime.yml
├── plugins/
│   ├── modules/
│   │   └── my_module.py
│   ├── inventory/
│   │   └── my_inventory.py
│   └── filter/
│       └── my_filter.py
├── roles/
│   └── myrole/
└── tests/
```
```yaml
# galaxy.yml
namespace: mycompany
name: mycollection
version: 1.0.0
readme: README.md
authors:
  - Your Name <[email protected]>
description: My custom collection
license:
  - MIT
tags:
  - mycompany
  - infrastructure
dependencies:
  community.general: ">=5.0.0"
repository: https://github.com/mycompany/mycollection
documentation: https://mycompany.github.io/mycollection
homepage: https://github.com/mycompany/mycollection
issues: https://github.com/mycompany/mycollection/issues
build_ignore:
  - .git
  - .github
```
```bash
# Build & publish
ansible-galaxy collection build
ansible-galaxy collection publish mycompany-mycollection-1.0.0.tar.gz --api-key=$GALAXY_API_KEY
```

### Bước 4: Custom Module (Python)

```python
# plugins/modules/custom_facts.py
#!/usr/bin/env python3
DOCUMENTATION = """
module: custom_facts
short_description: Gather custom application facts
version_added: "1.0.0"
description:
  - Returns custom application info
options:
  name:
    description: Application name
    required: true
    type: str
author:
  - Your Name
"""

EXAMPLES = """
- name: Get app facts
  custom_facts:
    name: myapp
  register: result
"""

RETURN = """
ansible_facts:
  description: Custom facts
  type: dict
"""

from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(
            name=dict(type='str', required=True),
        )
    )
    name = module.params['name']
    facts = {
        'app_name': name,
        'app_version': '1.2.3',
        'app_port': 8080,
    }
    module.exit_json(changed=False, ansible_facts={'app_info': facts})

if __name__ == '__main__':
    main()
```
```yaml
# Dùng trong playbook
- name: Test custom module
  hosts: localhost
  tasks:
    - name: Gather custom facts
      mycompany.mycollection.custom_facts:
        name: myapp
      register: result

    - name: Print
      ansible.builtin.debug:
        msg: "{{ result }}"
```

### Bước 5: Custom Filter Plugin

```python
# plugins/filter/format_uptime.py
def format_uptime(seconds):
    """Convert seconds to human-readable uptime."""
    days = seconds // 86400
    hours = (seconds % 86400) // 3600
    minutes = (seconds % 3600) // 60
    secs = seconds % 60
    return f"{days}d {hours}h {minutes}m {secs}s"

class FilterModule(object):
    def filters(self):
        return {'format_uptime': format_uptime}
```
```yaml
# Dùng
- name: Show uptime
  ansible.builtin.debug:
    msg: "Uptime: {{ ansible_uptime_seconds | format_uptime }}"
```

### 🎯 Bài tập P8
1. Cài role `geerlingguy.nginx`, dùng trong playbook
2. Cài collection `community.docker`, dùng module `docker_container`
3. Tạo 1 custom filter plugin đơn giản
4. Build 1 collection đơn giản

---

<a id="p9"></a>
## P9. Testing với Molecule

### Bước 1: Cài Molecule

```bash
pip install molecule molecule-docker docker ansible-lint pytest-testinfra
```

### Bước 2: Khởi tạo

```bash
cd roles/myrole
molecule init scenario default --driver-name docker
```
```bash
# Cấu trúc
roles/myrole/
├── molecule/
│   └── default/
│       ├── molecule.yml         # Cấu hình
│       ├── converge.yml         # Playbook apply role
│       ├── prepare.yml          # Chuẩn bị env
│       ├── verify.yml           # Verify kết quả
│       └── tests/
│           └── test_default.py  # Pytest test
├── tasks/
├── defaults/
└── ...
```

### Bước 3: Cấu hình molecule

```yaml
# molecule/default/molecule.yml
---
dependency:
  name: galaxy

driver:
  name: docker

platforms:
  - name: ubuntu-2204
    image: geerlingguy/docker-ubuntu2204-ansible:latest
    pre_build_image: true
    privileged: true
    command: /lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
  - name: rocky-9
    image: geerlingguy/docker-rocky9-ansible:latest
    pre_build_image: true

provisioner:
  name: ansible
  inventory:
    group_vars:
      all:
        test_variable: foo
  playbooks:
    converge: converge.yml
    prepare: prepare.yml

verifier:
  name: testinfra
  options:
    -vvv
```

### Bước 4: Tests với pytest-testinfra

```python
# molecule/default/tests/test_default.py
import os
import testinfra.utils.ansible_runner

testinfra_hosts = testinfra.utils.ansible_runner.AnsibleRunner(
    os.environ['MOLECULE_INVENTORY_FILE']
).get_hosts('all')


def test_nginx_installed(host):
    nginx = host.package("nginx")
    assert nginx.is_installed


def test_nginx_running(host):
    service = host.service("nginx")
    assert service.is_running
    assert service.is_enabled


def test_nginx_listening(host):
    socket = host.socket("tcp://0.0.0.0:80")
    assert socket.is_listening


def test_config_file(host):
    config = host.file("/etc/nginx/nginx.conf")
    assert config.exists
    assert config.user == "root"
    assert config.group == "root"
    assert config.mode == 0o644


def test_index_page(host):
    page = host.file("/var/www/html/index.html")
    assert page.exists
    assert page.contains("Hello from")


def test_nginx_response(host):
    cmd = host.run("curl -s http://localhost/")
    assert cmd.rc == 0
    assert "Hello" in cmd.stdout
```

### Bước 5: Workflow Molecule

```bash
cd roles/myrole

# Chạy tất cả: create -> prepare -> converge -> idempotence -> verify -> destroy
molecule test

# Từng bước
molecule create               # Tạo instances
molecule prepare              # Chuẩn bị
molecule converge             # Apply role
molecule idempotence          # Chạy lại, không thay đổi
molecule verify               # Test
molecule destroy              # Cleanup
molecule reset                # Destroy + create

# Debug
molecule converge -- --check --diff
molecule login -h ubuntu-2204  # SSH vào instance
```

### Bước 6: Test nhiều scenario

```bash
molecule init scenario -d docker minimal
molecule init scenario -d docker rhel

molecule test -s minimal
molecule test -s rhel

# Matrix CI
molecule test -s default -- --tags "fast"
```

### Bước 7: Test với docker driver (CI)

```yaml
# .github/workflows/ci.yml
name: Molecule Test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          pip install ansible molecule molecule-docker docker
          sudo apt-get update
          sudo apt-get install -y docker.io
          sudo usermod -aG docker $USER

      - name: Run molecule
        run: |
          cd roles/${{ matrix.role }}
          molecule test

    strategy:
      matrix:
        role: [nginx, postgresql, common]
```

### Bước 8: Lint với ansible-lint

```bash
ansible-lint playbook.yaml
ansible-lint roles/

# File cấu hình
cat > .ansible-lint <<EOF
---
profile: production
skip_list:
  - role-name        # Tắt check role naming
  - yaml[line-length]
EOF

# Tự động fix một số lỗi
ansible-lint --fix playbook.yaml
```

### 🎯 Bài tập P9
1. Khởi tạo molecule cho role nginx
2. Viết test testinfra kiểm tra nginx cài và chạy
3. Chạy `molecule test` thành công
4. Cấu hình ansible-lint, fix hết warning

---

<a id="p10"></a>
## P10. Production với Ansible Tower / AAP

### Bước 1: Hiểu kiến trúc AAP

```
┌────────────────────────────────────────────────────────┐
│                  Ansible Automation                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  Controller  │  │  Automation  │  │   Private    │ │
│  │   (Tower)    │  │     Hub      │  │  Automation  │ │
│  │              │  │  (Galaxy)    │  │     Hub      │ │
│  │ • Web UI     │  │ • Content    │  │              │ │
│  │ • REST API   │  │   Collections│  │ • On-prem    │ │
│  │ • Schedules  │  │ • Roles      │  │ • Registry   │ │
│  │ • RBAC       │  │              │  │              │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│           │                                             │
│           ▼                                             │
│  ┌──────────────────────────────────────────────────┐  │
│  │      Execution Nodes (Ansible Runner)            │  │
│  │      - Tự động scale                              │  │
│  │      - Container isolated                         │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### Bước 2: Cài AWX (open-source Tower)

```bash
# Minikube + AWX
minikube start --cpus=4 --memory=8g --addons=ingress

# Cài operator
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/devel/deploy/awx-operator.yaml

# Cấu hình AWX
cat > awx.yaml <<EOF
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
  nodeport_port: 30080
  ingress_type: none
  postgres_storage_class: standard
  web_resource_cpu: 1
  web_resource_memory: 2Gi
  task_resource_cpu: 1
  task_resource_memory: 2Gi
EOF

kubectl apply -f awx.yaml
kubectl get awx -w   # Đợi READY=True

# Lấy password admin
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode
```

### Bước 3: Tổ chức Project & Inventory

```bash
# Cấu trúc repo GitOps cho Ansible
ansible-repo/
├── inventories/
│   ├── production/
│   │   ├── hosts
│   │   └── group_vars/
│   └── staging/
│       └── hosts
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── database.yml
├── roles/
│   ├── nginx/
│   ├── postgresql/
│   └── common/
├── collections/
│   └── requirements.yml
└── files/
```

### Bước 4: Workflow job template

```yaml
# Workflow: Deploy multi-tier
- name: Deploy Application
  workflow_nodes:
    - identifier: pre_check
      unified_job_template:
        name: Pre-check
      success_nodes:
        - identifier: deploy_web

    - identifier: deploy_web
      unified_job_template:
        name: Deploy Web
      success_nodes:
        - identifier: deploy_db
        - identifier: run_tests

    - identifier: deploy_db
      unified_job_template:
        name: Deploy Database
      success_nodes:
        - identifier: smoke_test

    - identifier: run_tests
      unified_job_template:
        name: Run Tests

    - identifier: smoke_test
      unified_job_template:
        name: Smoke Test
      all_parents_must_converge: true
```

### Bước 5: CI/CD Integration

```yaml
# .github/workflows/deploy.yml
name: Deploy via AWX
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Sync to AWX
        run: |
          curl -X POST "https://awx.example.com/api/v2/projects/5/update/" \
            -H "Authorization: Bearer ${{ secrets.AWX_TOKEN }}" \
            -H "Content-Type: application/json"

      - name: Launch job template
        run: |
          JOB_ID=$(curl -X POST "https://awx.example.com/api/v2/job_templates/7/launch/" \
            -H "Authorization: Bearer ${{ secrets.AWX_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"limit": "web", "extra_vars": {"env": "production"}}' \
            | jq -r .job)

          echo "Job started: $JOB_ID"

          # Đợi job hoàn thành
          while true; do
            STATUS=$(curl -s "https://awx.example.com/api/v2/jobs/$JOB_ID/" \
              -H "Authorization: Bearer ${{ secrets.AWX_TOKEN }}" \
              | jq -r .status)
            echo "Status: $STATUS"
            if [ "$STATUS" = "successful" ] || [ "$STATUS" = "failed" ]; then
              break
            fi
            sleep 10
          done

          # Fail nếu job failed
          if [ "$STATUS" = "failed" ]; then exit 1; fi
```

### Bước 6: REST API cơ bản

```bash
# Lấy token
TOKEN=$(curl -u admin:password -X POST \
  "https://awx.example.com/api/v2/tokens/" \
  -H "Content-Type: application/json" \
  | jq -r .token)

# List inventories
curl -H "Authorization: Bearer $TOKEN" \
  "https://awx.example.com/api/v2/inventories/" | jq

# Launch job
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://awx.example.com/api/v2/job_templates/7/launch/" \
  -d '{"limit": "web", "extra_vars": {"version": "2.0"}}'

# Tham khảo API
# https://awx.example.com/api/v2/
# https://awx.example.com/api/v2/job_templates/
# https://awx.example.com/api/v2/organizations/
```

### Bước 7: Callback / Webhook

```bash
# Cho phép host tự pull config (pull mode)
# Trên managed node, cài ansible-runner hoặc callback plugin

# Cài ansible-runner service
pip install ansible-runner

# Service chạy như một endpoint
ansible-runner-service

# Host tự gọi:
curl -X POST http://control:5001/api/v1/job_events/ \
  -H "Content-Type: application/json" \
  -d '{"host": "web1", "event": "boot"}'
```

### 🎯 Bài tập P10
1. Cài AWX trên minikube
2. Tạo Project sync từ Git
3. Tạo Inventory + Credential + Job Template
4. Chạy job template qua API

---

<a id="p11"></a>
## P11. Troubleshooting

### Bước 1: Tăng verbosity

```bash
# Verbosity levels
ansible-playbook playbook.yaml -v              # Hiển thị kết quả task
ansible-playbook playbook.yaml -vv             # + Connection details
ansible-playbook playbook.yaml -vvv            # + Debug info (chi tiết nhất)
ansible-playbook playbook.yaml -vvvv           # + Plugin source

# Debug kết nối SSH
ANSIBLE_DEBUG=1 ansible-playbook playbook.yaml -vvv

# Kiểm tra command thực sự gửi đi
ANSIBLE_KEEP_REMOTE_FILES=1 ansible-playbook playbook.yaml
# Sau đó xem file tạm trên managed node
# ~/.ansible/tmp/ansible-XXXXXXX/

# Check mode + diff (xem thay đổi gì)
ansible-playbook playbook.yaml --check --diff -v
```

### Bước 2: Module debug

```yaml
- name: Debug values
  hosts: web
  tasks:
    # In biến
    - name: Print variable
      ansible.builtin.debug:
        var: my_var
      vars:
        my_var: "Hello"

    # In message
    - name: Print message
      ansible.builtin.debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          IP: {{ ansible_default_ipv4.address }}
          Memory: {{ ansible_memtotal_mb }} MB

    # Print kết quả register
    - name: Run command
      ansible.builtin.command: uptime
      register: result

    - name: Show result
      ansible.builtin.debug:
        var: result

    # In từng key
    - name: Show stdout
      ansible.builtin.debug:
        msg: "{{ result.stdout }}"

    # Conditional debug
    - name: Debug when fail
      ansible.builtin.debug:
        msg: "Task failed: {{ ansible_failed_result }}"
      when: ansible_failed_result is defined
```

### Bước 3: Lỗi thường gặp

#### ❌ Lỗi 1: SSH Permission denied
```
UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh
```

```bash
# Checklist:
ssh -i key.pem user@host                    # Test SSH thủ công
ssh -vvv user@host                          # Verbose
ls -la key.pem                              # Permission 600?
ansible all -m ping -vvv                    # Verbose Ansible

# Thêm vào inventory:
ansible_ssh_private_key_file=/path/to/key
ansible_ssh_pass=password
ansible_user=ubuntu
ansible_become=yes
ansible_become_method=sudo
```

#### ❌ Lỗi 2: Module not found
```
ERROR! couldn't resolve module/action 'docker_container'
```

```bash
# Cài collection
ansible-galaxy collection install community.docker

# Hoặc dùng FQCN
community.docker.docker_container:

# Hoặc thêm vào requirements.yml
```

#### ❌ Lỗi 3: Host không tồn tại trong inventory
```
ERROR! No hosts matched
```

```bash
ansible-inventory --list-hosts              # Kiểm tra inventory
ansible-inventory -i inventory.yaml --list-hosts web
ansible all --list-hosts --limit=web1
```

#### ❌ Lỗi 4: Permission denied (sudo)
```
FAILED! => {"msg": "Missing sudo password"}
```

```yaml
# Trong inventory
ansible_become=yes
ansible_become_method=sudo
ansible_become_password=password     # Hoặc dùng --ask-become-pass

# Hoặc dùng passwordless sudo
echo "user ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/user
```

#### ❌ Lỗi 5: Jinja2 template error
```
ERROR! template error while templating string: unexpected '}'
```

```bash
# Escape literal {{ }} trong template: dùng {% raw %}
{% raw %}
  literal {{ not_jinja }}
{% endraw %}
```

#### ❌ Lỗi 6: Variable undefined
```
The task includes an option with an undefined variable
```

```yaml
# Dùng default
- name: Task
  ansible.builtin.debug:
    msg: "{{ my_var | default('default_value') }}"

# Hoặc fail có chủ đích
- name: Task
  ansible.builtin.debug:
    msg: "{{ my_var }}"
  vars:
    my_var: "{{ required_var | mandatory }}"
```

#### ❌ Lỗi 7: Slow performance
```bash
# Tăng forks
ansible-playbook playbook.yaml -f 50

# Enable pipelining trong ansible.cfg
[ssh_connection]
pipelining = True

# Dùng fact caching
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/facts_cache
fact_caching_timeout = 86400

# Hoặc Redis
fact_caching = redis
fact_caching_connection = redis://localhost:6379
```

#### ❌ Lỗi 8: Host key verification failed
```
Failed to connect to the host via ssh: host_known
```

```ini
# ansible.cfg
[defaults]
host_key_checking = False

# Hoặc
ansible-playbook playbook.yaml --ssh-extra-args="-o StrictHostKeyChecking=no"
```

#### ❌ Lỗi 9: Python interpreter not found
```
The python interpreter is not discovered
```

```ini
# ansible.cfg
[defaults]
interpreter_python = /usr/bin/python3

# Hoặc auto discover
interpreter_python = auto_silent

# Hoặc trong inventory
ansible_python_interpreter=/usr/bin/python3
```

#### ❌ Lỗi 10: Handler không chạy
```yaml
# Lỗi thường gặp: handler chỉ chạy khi task báo changed
# Nếu muốn handler LUÔN chạy, dùng force_handlers
- name: Always run handlers
  hosts: web
  force_handlers: yes
  tasks:
    - name: Risky task
      ansible.builtin.command: /opt/risky.sh
      # Nếu fail, handler vẫn chạy nhờ force_handlers
```

### Bước 4: Debug checklist tổng quát

```bash
#!/bin/bash
# debug.sh - Script debug Ansible
PLAYBOOK=$1
HOST=$2

echo "=== 1. Ansible version ==="
ansible --version | head -3

echo "=== 2. Inventory ==="
ansible-inventory --list-hosts all
ansible-inventory --host $HOST 2>/dev/null

echo "=== 3. Ping test ==="
ansible $HOST -m ping -vv

echo "=== 4. Fact gathering ==="
ansible $HOST -m setup | head -30

echo "=== 5. Syntax check ==="
ansible-playbook $PLAYBOOK --syntax-check

echo "=== 6. Dry run ==="
ansible-playbook $PLAYBOOK --check --diff --limit $HOST

echo "=== 7. Run 1 task ==="
ansible-playbook $PLAYBOOK --limit $HOST --start-at-task="<task_name>"

echo "=== 8. With debug ==="
ansible-playbook $PLAYBOOK --limit $HOST -vvv 2>&1 | tail -50
```

### Bước 5: Production troubleshooting tools

```bash
# Check callback - JSON output
ANSIBLE_STDOUT_CALLBACK=json ansible-playbook playbook.yaml

# Profile tasks
ANSIBLE_CALLBACKS_ENABLED=profile_tasks ansible-playbook playbook.yaml

# Timer
ANSIBLE_CALLBACKS_ENABLED=timer ansible-playbook playbook.yaml
```

### 🎯 Bài tập P11
1. Cố tình gõ sai tên module, fix bằng collection install
2. SSH fail vì permission key, fix
3. Template lỗi vì biến undefined, fix bằng `| default`
4. Tăng verbosity, đọc output từ `-vvv`

---

## 🎓 Lộ trình trở thành Chuyên gia

### 🗓️ Tháng 1-2: Nền tảng
- Hoàn thành P1-P3
- Thi **Red Hat Ansible Automation** (EX294 / DO374)
- Thực hành viết playbook cho hạ tầng cá nhân

### 🗓️ Tháng 3-4: Trung cấp
- Hoàn thành P4-P6
- Tạo vài role production-quality
- Áp dụng Ansible vào 1 dự án thực

### 🗓️ Tháng 5-6: Nâng cao
- Hoàn thành P7-P9
- Setup dynamic inventory cho cloud
- CI/CD với Molecule trong GitHub Actions

### 🗓️ Tháng 7-8: Chuyên gia
- Hoàn thành P10-P11
- Cài & vận hành AWX/AAP
- Thi **Red Hat Specialist in Ansible Automation** (EX457)

### 📚 Tài liệu tham khảo
- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Awesome Ansible](https://github.com/awesome-ansible/awesome-ansible)
- [Jeff Geerling's Blog](https://www.jeffgeerling.com/) - geerlingguy
- [Ansible Lint Rules](https://ansible.readthedocs.io/projects/lint/)

### 🛠️ Project cá nhân nên làm
1. **Home Lab**: Setup K3s + Ansible AWX, deploy multi-tier app
2. **Cloud Automation**: Tự động provision AWS/GCP bằng Ansible
3. **Role Collection**: Tạo 5 role production-quality, publish lên Galaxy
4. **GitOps**: Tích hợp Ansible + GitLab CI/CD
5. **CKA + Ansible**: Quản lý K8s cluster bằng Ansible

---

> **💡 Mẹo cuối**: Ansible càng viết càng phải đơn giản. Mỗi role nên làm **1 việc duy nhất** và làm tốt. Luôn test với Molecule trước khi apply production. Đọc code của **geerlingguy** trên GitHub - đó là chuẩn vàng.

---

*Tạo bởi tài liệu học Ansible - Chúc bạn thành công! 🚀*
