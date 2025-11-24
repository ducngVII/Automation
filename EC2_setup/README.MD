### Create role for node_exporter
# for inventory structure
ansible-inventory --graph 
# to check is server connected
ansible all -m ping 
# create a skeleton for node_exporter, after this cmd u will have a role dir
ansible-galaxy init roles/node-exporter  

# paste all of this to ./roles/node-exporter/defaults/main.yml
node_exporter_version: "1.1.2"
node_exporter_bin: /usr/local/bin/node_exporter
node_exporter_user: node-exporter
node_exporter_group: "{{ node_exporter_user }}"
node_exporter_dir_conf: /etc/node_exporter

# paste this to ./roles/node-exporter/tasks/main.yml
- name: check if node exporter exist
  stat:
    path: "{{ node_exporter_bin }}"
  register: __check_node_exporter_present
- name: create node exporter user
  user:
    name: "{{ node_exporter_user }}"
    append: true
    shell: /usr/sbin/nologin
    system: true
    create_home: false
- name: create node exporter config dir
  file:
    path: "{{ node_exporter_dir_conf }}"
    state: directory
    owner: "{{ node_exporter_user }}"
    group: "{{ node_exporter_group }}"
- name: if node exporter exist get version
  shell: "cat /etc/systemd/system/node_exporter.service | grep Version | sed s/'.*Version '//g"
  when: __check_node_exporter_present.stat.exists == true
  changed_when: false
  register: __get_node_exporter_version
  
- name: download and unzip node exporter if not exist
  unarchive:
    src: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
    dest: /tmp/
    remote_src: yes
    validate_certs: no
- name: move the binary to the final destination
  copy:
    src: "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
    dest: "{{ node_exporter_bin }}"
    owner: "{{ node_exporter_user }}"
    group: "{{ node_exporter_group }}"
    mode: 0755
    remote_src: yes
  when: __check_node_exporter_present.stat.exists == false or not __get_node_exporter_version.stdout == node_exporter_version
- name: clean
  file:
    path: /tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/
    state: absent
- name: install service
  template:
    src: node_exporter.service.j2
    dest: /etc/systemd/system/node_exporter.service
    owner: root
    group: root
    mode: 0755
  notify: reload_daemon_and_restart_node_exporter
- meta: flush_handlers
- name: service always started
  systemd:
    name: node_exporter
    state: started
    enabled: yes

# create node_exorter.service.j2 in ./roles/node-exporter/templates
[Unit]
Description=Node Exporter Version {{ node_exporter_version }}
After=network-online.target
[Service]
User={{ node_exporter_user }}
Group={{ node_exporter_user }}
Type=simple
ExecStart={{ node_exporter_bin }}
[Install]
WantedBy=multi-user.target

# handler file in ./roles/node-exporter/handlers/main.yml
- name: reload_daemon_and_restart_node_exporter
  systemd:
    name: node_exporter
    state: restarted
    daemon_reload: yes
    enabled: yes


### Create role for prometheus
ansible-galaxy init roles/prometheus

# ./roles/prometheus/defaults/main.yml
prometheus_dir_configuration: "/etc/prometheus"
prometheus_retention_time: "365d"
prometheus_scrape_interval: "30s"
prometheus_node_exporter: true
prometheus_node_exporter_group: "all"
prometheus_env: "production"
prometheus_var_config:
  global:
    scrape_interval: "{{ prometheus_scrape_interval }}"
    evaluation_interval: 5s
    external_labels:
      env: '{{ prometheus_env }}'
  scrape_configs:
    - job_name: prometheus
      scrape_interval: 5m
      static_configs:
        - targets: ['{{ inventory_hostname }}:9090']

# ./roles/prometheus/tasks/main.yml
- name: update and install prometheus
  apt:
    name: prometheus
    state: latest
    update_cache: yes
    cache_valid_time: 3600
- name: prometheus args
  template:
    src: prometheus.j2
    dest: /etc/default/prometheus
    mode: 0644
    owner: root
    group: root
  notify: restart_prometheus
- name: prometheus configuration file
  template:
    src: prometheus.yml.j2
    dest: "{{ prometheus_dir_configuration }}/prometheus.yml"
    mode: 0755
    owner: prometheus
    group: prometheus
  notify: reload_prometheus
- name: start prometheus
  systemd:
    name: prometheus
    state: started
    enabled: yes

# Create prometheus.yaml.j2 in templates
#jinja2: lstrip_blocks: "True"
{{ prometheus_var_config | to_nice_yaml(indent=2) }}
{% if prometheus_node_exporter_group %}
- job_name: node
  scrape_interval: 15s
  metrics_path: /metrics
  static_configs:
  - targets:
{% for server in groups[prometheus_node_exporter_group] %}
    - '{{ server }}:9100'
{% endfor %}
{% endif %}

# prometheus.j2 file templates
ARGS="--web.enable-lifecycle --storage.tsdb.retention.time={{ prometheus_retention_time }} --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries

# handlers for prometheus ./roles/prometheus/handlers/main.yml
- name: restart_prometheus
  systemd:
    name: prometheus
    state: restarted
    enabled: yes
    daemon_reload: yes
- name: reload_prometheus
  uri:
    url: http://localhost:9090
    method: POST
    status_code: 200


### Create role for grafana
ansible-galaxy init roles/grafana

# ./roles/grafana/tasks/main.yml
- name: install gpg
  apt:
    name: gnupg,software-properties-common
    state: present
    update_cache: yes
    cache_valid_time: 3600
- name: add gpg hey
  apt_key:
    url: "https://packages.grafana.com/gpg.key"
    validate_certs: no
- name: add repository
  apt_repository:
    repo: "deb https://packages.grafana.com/oss/deb stable main"             
    state: present
    validate_certs: no
- name: install grafana
  apt:
    name: grafana
    state: latest
    update_cache: yes
    cache_valid_time: 3600
- name: start service grafana-server
  systemd:
    name: grafana-server
    state: started
    enabled: yes
- name: wait for service up
  uri:
    url: "http://127.0.0.1:3000"
    status_code: 200
  register: __result
  until: __result.status == 200
  retries: 120
  delay: 1
- name: change admin password for grafana gui
  shell : "grafana-cli admin reset-admin-password {{ grafana_admin_password }}"
  register: __command_admin
  changed_when: __command_admin.rc !=0

# in ./roles/grafana/defaults/main.yml set up grafana_admin_password
grafana_admin_password:

### Create a yml file 
- name: install monitoring stack
  hosts: monitorserver
  become: yes
  roles:
  - prometheus
  - grafana
- name: install node-exporter
  hosts: nodeservers
  become: yes
  roles:
  - node-exporter

### test
ansible-playbook -i inventory.txt ur_file.yml