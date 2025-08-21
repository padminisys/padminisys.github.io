# Ansible Development Getting Started Guide

This guide provides step-by-step instructions for developing Ansible collections with Padmini Systems. Follow these commands to set up your development environment and create a complete Ansible collection.

## Prerequisites

- Git configured with SSH access to GitHub
- Python 3.8+ installed
- Basic understanding of Ansible concepts

## Step-by-Step Development Process

### 0. Initial Setup: Clone Repository

```bash
# Clone the empty GitHub repository
git clone git@github.com:padminisys/kubernetes.git
cd kubernetes
```

### 1. Development Environment Setup

```bash
# Create and activate Python virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Upgrade pip and install required tools
python -m pip install --upgrade pip
pip install "ansible-core>=2.16" ansible-dev-tools ansible-lint
```

### 2. Scaffold Ansible Collection

```bash
# Initialize the collection structure
ansible-galaxy collection init padminisys.kubernetes --init-path .

# Move collection files to root and clean up
shopt -s dotglob && mv padminisys/kubernetes/* . && rm -rf padminisys
```

### 3. Create Roles and Example Playbooks

#### Create kubernetes_setup Role

```bash
# Initialize the kubernetes_setup role
ansible-galaxy role init roles/kubernetes_setup

# Create the main tasks file
cat > roles/kubernetes_setup/tasks/main.yml <<'YML'
# SPDX-License-Identifier: MIT-0
---
- name: Setup Kubernetes cluster
  ansible.builtin.debug:
    msg: "Setting up Kubernetes cluster for {{ k8s_cluster_name | default('default-cluster') }}"

- name: Install Kubernetes packages
  ansible.builtin.debug:
    msg: "Installing Kubernetes version {{ k8s_version | default('1.28') }}"

- name: Configure Kubernetes master node
  ansible.builtin.debug:
    msg: "Configuring master node with {{ k8s_pod_network_cidr | default('10.244.0.0/16') }}"
YML

# Create role metadata
cat > roles/kubernetes_setup/meta/main.yml <<'YML'
# SPDX-License-Identifier: MIT-0
---
galaxy_info:
  author: Padminisys Team
  description: Kubernetes cluster setup and configuration
  company: Padmini Systems
  license: MIT
  min_ansible_version: "2.16"
  platforms:
    - name: Debian
      versions: ["all"]
    - name: EL
      versions: ["all"]
  galaxy_tags: [kubernetes, k8s, cluster, setup]
dependencies: []
YML
```

#### Create cilium_setup Role

```bash
# Initialize the cilium_setup role
ansible-galaxy role init roles/cilium_setup

# Create the main tasks file
cat > roles/cilium_setup/tasks/main.yml <<'YML'
# SPDX-License-Identifier: MIT-0
---
- name: Setup Cilium CNI
  ansible.builtin.debug:
    msg: "Installing Cilium CNI version {{ cilium_version | default('1.14') }}"

- name: Configure Cilium networking
  ansible.builtin.debug:
    msg: "Configuring Cilium with cluster pool CIDR {{ cilium_cluster_pool_cidr | default('10.0.0.0/8') }}"

- name: Enable Cilium features
  ansible.builtin.debug:
    msg: "Enabling Cilium features: {{ cilium_features | default(['hubble', 'encryption']) | join(', ') }}"
YML

# Create role metadata
cat > roles/cilium_setup/meta/main.yml <<'YML'
# SPDX-License-Identifier: MIT-0
---
galaxy_info:
  author: Padminisys Team
  description: Cilium CNI setup and configuration for Kubernetes
  company: Padmini Systems
  license: MIT
  min_ansible_version: "2.16"
  platforms:
    - name: Debian
      versions: ["all"]
    - name: EL
      versions: ["all"]
  galaxy_tags: [cilium, cni, networking, kubernetes]
dependencies:
  - role: kubernetes_setup
YML
```

#### Create worker_setup Role

```bash
# Initialize the worker_setup role
ansible-galaxy role init roles/worker_setup

# Create the main tasks file
cat > roles/worker_setup/tasks/main.yml <<'YML'
# SPDX-License-Identifier: MIT-0
---
- name: Setup Kubernetes worker nodes
  ansible.builtin.debug:
    msg: "Setting up worker node {{ inventory_hostname }}"

- name: Join worker to cluster
  ansible.builtin.debug:
    msg: "Joining worker to cluster {{ k8s_cluster_name | default('default-cluster') }}"

- name: Configure worker node resources
  ansible.builtin.debug:
    msg: "Configuring worker with {{ worker_cpu_limit | default('4') }} CPUs and {{ worker_memory_limit | default('8Gi') }} memory"
YML

# Create role metadata
cat > roles/worker_setup/meta/main.yml <<'YML'
# SPDX-License-Identifier: MIT-0
---
galaxy_info:
  author: Padminisys Team
  description: Kubernetes worker node setup and configuration
  company: Padmini Systems
  license: MIT
  min_ansible_version: "2.16"
  platforms:
    - name: Debian
      versions: ["all"]
    - name: EL
      versions: ["all"]
  galaxy_tags: [kubernetes, worker, node, setup]
dependencies:
  - role: kubernetes_setup
YML
```

#### Create Example Playbooks

```bash
# Create examples directory
mkdir -p examples

# Create comprehensive example playbook
cat > examples/kubernetes_cluster.yml <<'YML'
---
- name: Setup complete Kubernetes cluster
  hosts: k8s_masters
  gather_facts: true
  collections: [padminisys.kubernetes]
  roles:
    - role: kubernetes_setup
      vars:
        k8s_cluster_name: "production-cluster"
        k8s_version: "1.28"
        k8s_pod_network_cidr: "10.244.0.0/16"
    - role: cilium_setup
      vars:
        cilium_version: "1.14"
        cilium_cluster_pool_cidr: "10.0.0.0/8"
        cilium_features: ["hubble", "encryption", "l7-proxy"]

- name: Setup worker nodes
  hosts: k8s_workers
  gather_facts: true
  collections: [padminisys.kubernetes]
  roles:
    - role: worker_setup
      vars:
        k8s_cluster_name: "production-cluster"
        worker_cpu_limit: "8"
        worker_memory_limit: "16Gi"
YML

# Create individual role examples
cat > examples/kubernetes_setup.yml <<'YML'
---
- name: Example - Kubernetes Setup
  hosts: localhost
  gather_facts: false
  collections: [padminisys.kubernetes]
  roles:
    - role: kubernetes_setup
      vars:
        k8s_cluster_name: "dev-cluster"
        k8s_version: "1.28"
        k8s_pod_network_cidr: "10.244.0.0/16"
YML

cat > examples/cilium_setup.yml <<'YML'
---
- name: Example - Cilium Setup
  hosts: localhost
  gather_facts: false
  collections: [padminisys.kubernetes]
  roles:
    - role: cilium_setup
      vars:
        cilium_version: "1.14"
        cilium_cluster_pool_cidr: "10.0.0.0/8"
        cilium_features: ["hubble", "encryption"]
YML

cat > examples/worker_setup.yml <<'YML'
---
- name: Example - Worker Setup
  hosts: localhost
  gather_facts: false
  collections: [padminisys.kubernetes]
  roles:
    - role: worker_setup
      vars:
        k8s_cluster_name: "dev-cluster"
        worker_cpu_limit: "4"
        worker_memory_limit: "8Gi"
YML
```

### 4. Required Metadata Files

#### Collection Metadata

```bash
# Create galaxy.yml for collection metadata
cat > galaxy.yml <<'YML'
namespace: padminisys
name: kubernetes
version: 1.0.0
readme: README.md
authors: [Padminisys Team]
description: "Complete Kubernetes cluster setup and management collection with Cilium CNI."
license: MIT
tags: [kubernetes, k8s, cilium, cni, cluster, setup]
repository: "https://github.com/padminisys/kubernetes"
issues: "https://github.com/padminisys/kubernetes/issues"
build_ignore:
  - .git
  - .github
  - .venv
  - __pycache__
  - "*.tar.gz"
  - "*.zip"
  - ".vscode"
  - docs/
  - molecule/
  - tests/
YML

# Create runtime metadata
mkdir -p meta
cat > meta/runtime.yml <<'YML'
# SPDX-License-Identifier: MIT-0
---
requires_ansible: ">=2.16.0"
YML
```

#### Documentation

```bash
# Create comprehensive README
cat > README.md <<'MD'
# padminisys.kubernetes

Complete Kubernetes cluster setup and management collection with Cilium CNI support.

## Description

This Ansible collection provides comprehensive automation for setting up and managing Kubernetes clusters with the following components:

- **kubernetes_setup**: Core Kubernetes cluster initialization and configuration
- **cilium_setup**: Cilium CNI installation and configuration with advanced networking features
- **worker_setup**: Worker node joining and resource configuration

## Roles

### kubernetes_setup
Sets up the Kubernetes control plane and core components.

**Variables:**
- `k8s_cluster_name`: Name of the Kubernetes cluster (default: "default-cluster")
- `k8s_version`: Kubernetes version to install (default: "1.28")
- `k8s_pod_network_cidr`: Pod network CIDR (default: "10.244.0.0/16")

### cilium_setup
Installs and configures Cilium CNI with advanced networking features.

**Variables:**
- `cilium_version`: Cilium version to install (default: "1.14")
- `cilium_cluster_pool_cidr`: Cluster pool CIDR (default: "10.0.0.0/8")
- `cilium_features`: List of features to enable (default: ["hubble", "encryption"])

### worker_setup
Configures and joins worker nodes to the Kubernetes cluster.

**Variables:**
- `k8s_cluster_name`: Name of the cluster to join (default: "default-cluster")
- `worker_cpu_limit`: CPU limit for worker node (default: "4")
- `worker_memory_limit`: Memory limit for worker node (default: "8Gi")

## Example Usage

```yaml
- name: Setup complete Kubernetes cluster
  hosts: k8s_masters
  collections: [padminisys.kubernetes]
  roles:
    - kubernetes_setup
    - cilium_setup

- name: Setup worker nodes
  hosts: k8s_workers
  collections: [padminisys.kubernetes]
  roles:
    - worker_setup
```

## Requirements

- Ansible >= 2.16
- Python >= 3.8
- Target systems: Debian/Ubuntu or RHEL/CentOS

## Installation

```bash
ansible-galaxy collection install padminisys.kubernetes
```

## License

MIT
MD

# Create ansible-lint configuration
cat > .ansible-lint <<'YML'
skip_list:
  - yaml[line-length]
  - fqcn-builtins
  - name[casing]
YML
```

### 5. Quality Assurance and Build

```bash
# Run linting
ansible-lint

# Build the collection
ansible-galaxy collection build

# Verify build contents
ls -lh padminisys-kubernetes-*.tar.gz
tar -tzf padminisys-kubernetes-*.tar.gz | grep 'meta/runtime.yml'
```

### 6. Local Testing

#### Install and Test Collection

```bash
# Install the built collection locally
ansible-galaxy collection install padminisys-kubernetes-*.tar.gz --force

# Test individual roles
ansible-playbook examples/kubernetes_setup.yml -i localhost, -c local
ansible-playbook examples/cilium_setup.yml -i localhost, -c local
ansible-playbook examples/worker_setup.yml -i localhost, -c local

# Test complete cluster setup
ansible-playbook examples/kubernetes_cluster.yml -i localhost, -c local
```

#### Alternative Development Mode (No Installation Required)

```bash
# Set collection path for development
export ANSIBLE_COLLECTIONS_PATHS="$(pwd):$HOME/.ansible/collections:/usr/share/ansible/collections"

# Run playbooks directly
ansible-playbook examples/kubernetes_cluster.yml -i localhost, -c local
```

### 7. Publishing to Ansible Galaxy

#### Manual Publishing (One-time)

```bash
# Create API token in Ansible Galaxy (https://galaxy.ansible.com/me/preferences)
# NEVER commit the token to version control
export GALAXY_API_KEY='<your_galaxy_api_token>'

# Publish to Galaxy
ansible-galaxy collection publish padminisys-kubernetes-*.tar.gz --api-key "$GALAXY_API_KEY"
```

### 8. Automated CI/CD with GitHub Actions

```bash
# Create GitHub Actions workflow
mkdir -p .github/workflows
cat > .github/workflows/release.yml <<'YML'
name: Build & Publish Kubernetes Collection
on:
  push:
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install "ansible-core>=2.16" ansible-dev-tools ansible-lint
      
      - name: Run ansible-lint
        run: ansible-lint
      
      - name: Build collection
        run: ansible-galaxy collection build
      
      - name: Test collection locally
        run: |
          ansible-galaxy collection install padminisys-kubernetes-*.tar.gz --force
          ansible-playbook examples/kubernetes_setup.yml -i localhost, -c local

  publish:
    needs: lint-and-test
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install "ansible-core>=2.16" ansible-dev-tools
      
      - name: Build collection
        run: ansible-galaxy collection build
      
      - name: Publish to Galaxy
        env:
          GALAXY_API_KEY: ${{ secrets.GALAXY_API_KEY }}
        run: |
          TARBALL=$(ls -1 *.tar.gz | head -n1)
          ansible-galaxy collection publish "$TARBALL" --api-key "$GALAXY_API_KEY"
YML
```

**Important:** Add your Galaxy API token as a GitHub Actions secret named `GALAXY_API_KEY` in your repository settings.

### 9. Version Control and Release

```bash
# Add all files to git
git add .

# Commit changes
git commit -m "feat: scaffold padminisys.kubernetes collection with k8s, cilium, and worker roles"

# Push to main branch
git push origin main

# Create and push release tag
git tag v1.0.0
git push origin v1.0.0
```

### 10. Install from Ansible Galaxy (After Publishing)

```bash
# Install from Galaxy
ansible-galaxy collection install padminisys.kubernetes

# Use in your playbooks
ansible-playbook ~/.ansible/collections/ansible_collections/padminisys/kubernetes/examples/kubernetes_cluster.yml -i localhost, -c local
```

### 11. Advanced Setup (Optional)

#### Requirements Files for AWX Integration

```bash
# Create collection requirements for AWX
cat > requirements.yml <<'YML'
collections:
  - name: padminisys.kubernetes
    version: '>=1.0.0'
  - name: kubernetes.core
    version: '>=2.4.0'
  - name: community.general
    version: '>=7.0.0'
YML

# Create Python requirements
cat > requirements.txt <<'YML'
ansible-core>=2.16
kubernetes>=24.2.0
PyYAML>=6.0
jinja2>=3.1.0
YML
```

#### Execution Environment for AWX

```bash
# Install ansible-builder
pip install ansible-builder

# Create execution environment definition
cat > execution-environment.yml <<'YML'
version: 3
dependencies:
  galaxy: requirements.yml
  python: requirements.txt
images:
  base_image:
    name: quay.io/ansible/ansible-runner:latest
additional_build_steps:
  prepend: |
    RUN pip3 install --upgrade pip setuptools
  append: |
    RUN ansible-galaxy collection list
YML

# Build execution environment
ansible-builder build -t ghcr.io/padminisys/ee_kubernetes:1.0.0

# Push to registry (requires authentication)
# docker login ghcr.io
# docker push ghcr.io/padminisys/ee_kubernetes:1.0.0
```

## Development Best Practices

1. **Follow the coding standards** outlined in our [development standards](ansible-dev-standards.md)
2. **Test thoroughly** - Always test your roles locally before publishing
3. **Version properly** - Use semantic versioning for releases
4. **Document everything** - Keep README files updated with examples
5. **Use linting** - Run `ansible-lint` before every commit
6. **Modular design** - Keep roles focused on single responsibilities
7. **Variable naming** - Use consistent, descriptive variable names
8. **Error handling** - Include proper error handling and validation

## Troubleshooting

### Common Issues

1. **Collection not found**: Ensure `ANSIBLE_COLLECTIONS_PATHS` is set correctly
2. **Linting errors**: Run `ansible-lint --fix` to auto-fix common issues
3. **Build failures**: Check `galaxy.yml` syntax and required files
4. **Import errors**: Verify all dependencies are listed in `meta/runtime.yml`

### Getting Help

- Check our [development standards](ansible-dev-standards.md) for coding guidelines
- Review example playbooks in the `examples/` directory
- Open issues on the GitHub repository for bugs or feature requests

---

*This guide provides a complete workflow for developing Ansible collections at Padmini Systems. Follow these steps to ensure consistency and quality in your automation projects.*