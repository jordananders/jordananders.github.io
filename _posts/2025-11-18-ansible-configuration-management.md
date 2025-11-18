---
layout: default
title:  "Configuration Management with Ansible for ML Infrastructure"
date:   2025-11-18 12:00:00
categories: DevOps Ansible ConfigurationManagement
---

Ansible provides agentless configuration management ideal for ML infrastructure - from GPU driver installation to model serving setup.

## Project Structure

```
ansible/
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── ml_servers.yml
│   └── staging/
├── roles/
│   ├── common/
│   ├── nvidia-driver/
│   ├── docker/
│   ├── kubernetes/
│   └── ml-server/
├── playbooks/
│   ├── site.yml
│   ├── ml-cluster.yml
│   └── gpu-setup.yml
└── ansible.cfg
```

## Inventory Configuration

### Dynamic Inventory

```yaml
# inventories/production/hosts.yml
all:
  children:
    ml_servers:
      hosts:
        ml-gpu-[01:10].example.com:
      vars:
        gpu_type: nvidia-a100
        gpu_count: 8

    feature_store:
      hosts:
        redis-[01:03].example.com:
      vars:
        redis_cluster: true

    model_registry:
      hosts:
        mlflow-01.example.com:

  vars:
    ansible_user: deploy
    ansible_python_interpreter: /usr/bin/python3
```

### Group Variables

```yaml
# group_vars/ml_servers.yml
nvidia_driver_version: "535.86.10"
cuda_version: "12.2"
docker_nvidia_runtime: true

# ML serving configuration
model_server_port: 8080
model_cache_dir: /var/cache/models
max_concurrent_requests: 100

# Resource limits
memory_limit_gb: 64
gpu_memory_fraction: 0.9
```

## GPU Setup Role

### Tasks

```yaml
# roles/nvidia-driver/tasks/main.yml
---
- name: Install NVIDIA driver prerequisites
  apt:
    name:
      - build-essential
      - dkms
      - linux-headers-{{ ansible_kernel }}
    state: present
    update_cache: yes

- name: Add NVIDIA package repository
  apt_key:
    url: https://nvidia.github.io/libnvidia-container/gpgkey
    state: present

- name: Add NVIDIA container toolkit repo
  apt_repository:
    repo: "deb https://nvidia.github.io/libnvidia-container/stable/ubuntu22.04/$(ARCH) /"
    state: present
    filename: nvidia-container-toolkit

- name: Install NVIDIA driver
  apt:
    name: "nvidia-driver-{{ nvidia_driver_version }}"
    state: present
  notify: reboot server

- name: Install CUDA toolkit
  apt:
    name: "cuda-{{ cuda_version }}"
    state: present

- name: Install NVIDIA container toolkit
  apt:
    name: nvidia-container-toolkit
    state: present
  notify: restart docker

- name: Configure Docker for NVIDIA runtime
  template:
    src: daemon.json.j2
    dest: /etc/docker/daemon.json
  notify: restart docker
  when: docker_nvidia_runtime

- name: Verify GPU is accessible
  command: nvidia-smi
  register: nvidia_smi_output
  changed_when: false

- name: Display GPU information
  debug:
    var: nvidia_smi_output.stdout_lines
```

### Templates

```json
{# roles/nvidia-driver/templates/daemon.json.j2 #}
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    },
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m",
        "max-file": "3"
    },
    "storage-driver": "overlay2"
}
```

### Handlers

```yaml
# roles/nvidia-driver/handlers/main.yml
---
- name: reboot server
  reboot:
    reboot_timeout: 300

- name: restart docker
  service:
    name: docker
    state: restarted
```

## ML Server Role

### Tasks

```yaml
# roles/ml-server/tasks/main.yml
---
- name: Create model cache directory
  file:
    path: "{{ model_cache_dir }}"
    state: directory
    owner: mluser
    group: mluser
    mode: '0755'

- name: Install Python ML dependencies
  pip:
    name:
      - torch=={{ torch_version }}
      - transformers=={{ transformers_version }}
      - fastapi
      - uvicorn
    virtualenv: /opt/ml-server/venv
    virtualenv_command: python3 -m venv

- name: Copy model server configuration
  template:
    src: config.yaml.j2
    dest: /opt/ml-server/config.yaml
    owner: mluser
    group: mluser
    mode: '0644'
  notify: restart ml-server

- name: Create systemd service
  template:
    src: ml-server.service.j2
    dest: /etc/systemd/system/ml-server.service
  notify:
    - reload systemd
    - restart ml-server

- name: Ensure ML server is running
  service:
    name: ml-server
    state: started
    enabled: yes

- name: Download initial model
  command: >
    /opt/ml-server/venv/bin/python -c
    "from transformers import AutoModel; AutoModel.from_pretrained('{{ default_model }}')"
  args:
    creates: "{{ model_cache_dir }}/{{ default_model }}"
  become_user: mluser
```

### Configuration Template

```yaml
{# roles/ml-server/templates/config.yaml.j2 #}
server:
  host: 0.0.0.0
  port: {{ model_server_port }}
  workers: {{ ansible_processor_vcpus // 2 }}

model:
  cache_dir: {{ model_cache_dir }}
  default: {{ default_model }}
  max_batch_size: 32

gpu:
  device_ids: [{% for i in range(gpu_count) %}{{ i }}{% if not loop.last %}, {% endif %}{% endfor %}]
  memory_fraction: {{ gpu_memory_fraction }}

logging:
  level: INFO
  format: json
```

### Systemd Service

```ini
{# roles/ml-server/templates/ml-server.service.j2 #}
[Unit]
Description=ML Inference Server
After=network.target docker.service

[Service]
Type=simple
User=mluser
Group=mluser
WorkingDirectory=/opt/ml-server
Environment="PATH=/opt/ml-server/venv/bin:/usr/local/bin:/usr/bin"
Environment="CUDA_VISIBLE_DEVICES={{ range(gpu_count) | list | join(',') }}"
ExecStart=/opt/ml-server/venv/bin/uvicorn main:app --config config.yaml
Restart=always
RestartSec=5
MemoryLimit={{ memory_limit_gb }}G

[Install]
WantedBy=multi-user.target
```

## Playbooks

### Main Site Playbook

```yaml
# playbooks/site.yml
---
- name: Configure all servers
  hosts: all
  become: yes
  roles:
    - common

- name: Configure ML GPU servers
  hosts: ml_servers
  become: yes
  roles:
    - nvidia-driver
    - docker
    - ml-server

- name: Configure feature store
  hosts: feature_store
  become: yes
  roles:
    - redis-cluster

- name: Configure model registry
  hosts: model_registry
  become: yes
  roles:
    - mlflow
```

### GPU Cluster Setup

```yaml
# playbooks/gpu-cluster.yml
---
- name: Setup GPU cluster for ML training
  hosts: ml_servers
  become: yes
  vars:
    prometheus_node_exporter: true
    nvidia_dcgm_exporter: true

  pre_tasks:
    - name: Check if NVIDIA GPU present
      command: lspci | grep -i nvidia
      register: gpu_check
      failed_when: false
      changed_when: false

    - name: Fail if no GPU detected
      fail:
        msg: "No NVIDIA GPU detected on {{ inventory_hostname }}"
      when: gpu_check.rc != 0

  roles:
    - nvidia-driver
    - docker
    - { role: kubernetes, when: k8s_enabled | default(false) }

  post_tasks:
    - name: Run GPU validation
      include_tasks: tasks/validate-gpu.yml

    - name: Report cluster status
      debug:
        msg: "GPU cluster setup complete. {{ ansible_play_hosts | length }} nodes configured."
      run_once: true
```

### Validation Tasks

```yaml
# playbooks/tasks/validate-gpu.yml
---
- name: Run nvidia-smi
  command: nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv
  register: gpu_info
  changed_when: false

- name: Verify CUDA
  command: nvcc --version
  register: cuda_version
  changed_when: false

- name: Test PyTorch GPU access
  command: >
    python3 -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}, Devices: {torch.cuda.device_count()}')"
  register: pytorch_check
  changed_when: false

- name: Display validation results
  debug:
    msg:
      - "GPU Info: {{ gpu_info.stdout }}"
      - "CUDA: {{ cuda_version.stdout_lines[-1] }}"
      - "PyTorch: {{ pytorch_check.stdout }}"
```

## Custom Modules

### GPU Health Check Module

```python
#!/usr/bin/python
# library/nvidia_gpu_check.py

from ansible.module_utils.basic import AnsibleModule
import subprocess
import json

def main():
    module = AnsibleModule(
        argument_spec=dict(
            min_memory_gb=dict(type='int', required=False, default=0),
            expected_count=dict(type='int', required=False, default=1)
        )
    )

    try:
        result = subprocess.run(
            ['nvidia-smi', '--query-gpu=name,memory.total,temperature.gpu',
             '--format=csv,noheader,nounits'],
            capture_output=True, text=True, check=True
        )

        gpus = []
        for line in result.stdout.strip().split('\n'):
            name, memory, temp = line.split(', ')
            gpus.append({
                'name': name,
                'memory_mb': int(memory),
                'temperature': int(temp)
            })

        # Validations
        if len(gpus) < module.params['expected_count']:
            module.fail_json(
                msg=f"Expected {module.params['expected_count']} GPUs, found {len(gpus)}"
            )

        min_memory = module.params['min_memory_gb'] * 1024
        for gpu in gpus:
            if gpu['memory_mb'] < min_memory:
                module.fail_json(
                    msg=f"GPU {gpu['name']} has insufficient memory"
                )

        module.exit_json(
            changed=False,
            gpus=gpus,
            gpu_count=len(gpus)
        )

    except subprocess.CalledProcessError as e:
        module.fail_json(msg=f"nvidia-smi failed: {e.stderr}")

if __name__ == '__main__':
    main()
```

## AWX/Tower Integration

### Job Template

```yaml
# awx-job-template.yml
name: Deploy ML Infrastructure
project: ML Platform
playbook: playbooks/gpu-cluster.yml
inventory: Production ML
credentials:
  - SSH Key
  - Vault Password
extra_vars:
  nvidia_driver_version: "535.86.10"
  cuda_version: "12.2"
job_tags: ""
skip_tags: ""
verbosity: 1
```

### Workflow Template

```yaml
# awx-workflow.yml
name: ML Infrastructure Deployment
nodes:
  - name: Validate Inventory
    job_template: Inventory Validation
    success_nodes:
      - name: Deploy GPU Drivers
        job_template: GPU Driver Install
        success_nodes:
          - name: Deploy ML Server
            job_template: ML Server Setup
            success_nodes:
              - name: Run Tests
                job_template: Integration Tests
```

## Best Practices

1. **Idempotent tasks**: Ensure playbooks can run multiple times
2. **Use roles**: Organize code into reusable components
3. **Vault secrets**: Encrypt sensitive data
4. **Tags**: Enable selective execution
5. **Check mode**: Test before applying changes
6. **Handlers**: Use for service restarts
7. **Variables precedence**: Understand override order
8. **Testing**: Use Molecule for role testing

## Resources

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [NVIDIA Ansible Collection](https://galaxy.ansible.com/nvidia/gpu_operator)
- [Molecule Testing](https://molecule.readthedocs.io/)

---

*Questions about Ansible configuration? [Let me know](mailto:jordan@jordananderson.us).*
