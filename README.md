
# Homelab CI/CD Platform on Proxmox

**Refrence:** `https://github.com/lloyd-theophilus/DevOps-Projects/tree/master/DevOps-Project-06`

Terraform → Ansible → Jenkins → SonarQube Cloud → private registry → k3d (Kubernetes) → Prometheus/Grafana

A self-contained learning platform that runs on one Proxmox laptop. A `git push` to the app repo starts a Jenkins pipeline that scans the code, builds two container images, pushes them to a private registry and deploys them to a k3d Kubernetes cluster. Infrastructure is created with Terraform, with its state stored in an S3-compatible service (Garage) running in the same lab.

> **Placeholders.** Values in `<ANGLE_BRACKETS>` are yours to fill in. Addresses marked *(verify)* were assumed during the build and must be checked against your network.

---

## 1. Architecture

```
GitHub repo ──push──▶ Jenkins master (CT 302) ──schedules──▶ Jenkins agent (CT 303)
                                                                  │
                          ┌───────────────────────────────────────┤
                          ▼                    ▼                  ▼
                 SonarQube Cloud      Registry (registry:2)   k3d cluster
                 (code analysis)      192.168.20.90:5000      kubectl apply -k
                                                                  │
                                                                  ▼
                                          project ns: app + backend + db
                                          monitoring ns: Prometheus + Grafana

Terraform (state in Garage S3) ──provisions──▶ LXC containers on Proxmox
Ansible (controller CT 301)    ──configures──▶ Jenkins master and agent
```

### Components

| Component | Where | Address | Purpose |
|---|---|---|---|
| Proxmox VE (`pve1`) | Lenovo laptop | `<PROXMOX_IP>:8006` | Hypervisor for all containers |
| Ansible controller | LXC 301, VLAN 20 | `<ANSIBLE_IP>` | Runs playbooks |
| Jenkins master | LXC 302, VLAN 20 | `192.168.20.32` | Schedules jobs, never runs builds |
| Jenkins agent (`jenkins-agent`) | LXC 303, VLAN 20 | `192.168.20.33` | Runs all build steps |
| Image registry | Container | `192.168.20.90:5000` | Plain Docker `registry:2`, HTTP, no auth |
| Kubernetes | k3d cluster `k3s-default` inside the `k3s-server` container | `192.168.20.10:6443` | Runs the app and monitoring |
| Terraform state | Garage (S3 API) | `<GARAGE_IP>:3900` | Remote backend, bucket `tfstate` |
| Code analysis | SonarQube Cloud | `https://sonarcloud.io` (or `https://sonarqube.us`) | Static analysis |

Network: VLAN 20 (`192.168.20.0/24`, gateway `192.168.20.1`) on a VLAN-aware bridge `vmbr0`.

### Versions seen during the build

Terraform provider `bpg/proxmox` 0.122.0 · Garage v2.5.0 · k3s v1.35.5+k3s1 · Grafana 13.3.1 · Ubuntu 22.04 containers.

---

## 2. Resource budget (read this first)

The laptop has roughly **7.5 GB usable RAM** and a single disk. Almost every instability in this build was resource starvation, so plan for it.

| Workload | Suggested memory | Notes |
|---|---|---|
| Jenkins master | 2048 MB | Cap the JVM heap (see 6.3) |
| Jenkins agent | 3072 MB, 2 cores | The SonarQube scan needs this headroom |
| k3d cluster (server + 2 agents) | in `k3s-server`, ~3-4 GB | The heaviest consumer |
| Registry | 512-1024 MB | Tiny |
| Garage | 512 MB | Tiny |
| Prometheus + Grafana | ~0.5-1 GB | Kept lean on purpose |

These do not all fit comfortably at once. **Run the CI stack and the cluster at different times**, or stop idle containers before builds.

How to tell the host is struggling (`free -h` alone is misleading):

```bash
vmstat 2 5        # wa (I/O wait) over ~20-30, or steady si/so (swap), means thrashing
dmesg -T | grep -iE "out of memory|oom|killed process" | tail
```

Keep **Lens closed** unless needed. It watches every resource type and added enough load to push the cluster API into timeouts.

---

## 3. Repository layout

```
infra/                          # Terraform
├── providers.tf                # provider + backend
├── variables.tf
├── main.tf                     # module "lxc" with for_each
└── modules/lxc/
    ├── main.tf
    ├── variables.tf
    └── versions.tf             # required_providers (bpg/proxmox)

controller/                     # Ansible
├── inventory.ini
└── site.yml

todo-app/                       # application repo
├── Dockerfile                  # frontend/app image
├── backend/Dockerfile          # backend image
├── Jenkinsfile
├── sonar-project.properties
└── deployments/                # (or manifests/) Kubernetes YAML
    ├── kustomization.yaml
    └── *.yaml

monitoring/                     # NOT inside the deployed folder
├── prometheus-values.yaml
└── grafana.yaml
```

---

## 4. Infrastructure with Terraform

### 4.1 API token (Proxmox web UI)

1. **Datacenter → Permissions → Users → Add**: `terraform@pve`.
2. **Permissions → Add → User Permission**: path `/`, user `terraform@pve`, role `Administrator`, propagate on.
3. **API Tokens → Add**: user `terraform@pve`, ID `tf`, **Privilege Separation off**. Copy the secret once.
4. If Privilege Separation is on, also add an **API Token Permission** for `terraform@pve!tf`.

> The permission must exist for the **user** as well as the token. A token only ever gets a subset of its user's permissions, so a token row alone gives `403 Permission check failed (/vms/<id>, VM.Clone)`.

### 4.2 Provider and variables

```hcl
terraform {
  required_providers {
    proxmox = { source = "bpg/proxmox", version = "~> 0.66" }
  }
}

provider "proxmox" {
  endpoint  = "https://<PROXMOX_IP>:8006/"     # the address you use for the web UI
  api_token = var.pve_api_token                # terraform@pve!tf=<secret>
  insecure  = true                             # self-signed certificate
  ssh { agent = true, username = "root" }
}
```

```bash
export TF_VAR_pve_api_token='terraform@pve!tf=<secret>'
export TF_VAR_ssh_public_key="$(cat ~/.ssh/id_ed25519.pub)"
```

### 4.3 Module and `for_each`

`modules/lxc/versions.tf` must repeat the provider source, or Terraform looks for `hashicorp/proxmox`:

```hcl
terraform {
  required_providers {
    proxmox = { source = "bpg/proxmox" }
  }
}
```

Root `main.tf`:

```hcl
locals {
  containers = {
    ansible-controller = { id = 301, ip = "<ANSIBLE_IP>",   memory = 1024 }
    jenkins-master     = { id = 302, ip = "<ANSIBLE_IP", memory = 2048 }
    jenkins-agent      = { id = 303, ip = "<ANSIBLE_IP", memory = 3072 }
  }
}

module "lxc" {
  source   = "./modules/lxc"
  for_each = local.containers

  name           = each.key
  vmid           = each.value.id
  ip             = each.value.ip
  memory         = each.value.memory
  gateway        = "192.168.20.1"
  node_name      = "pve1"
  template_id    = <CT_TEMPLATE_ID>
  ssh_public_key = var.ssh_public_key
}
```

Copy the container arguments from your working resource into `modules/lxc/main.tf`, replacing hardcoded values with `var.*`.

### 4.4 Refactoring an existing deployment safely

Moving a resource into a module changes its state address, and Terraform will plan to **destroy and recreate** it unless told otherwise. Use `moved` blocks, one per container:

```hcl
moved {
  from = proxmox_virtual_environment_container.node["jenkins-master"]
  to   = module.lxc["jenkins-master"].proxmox_virtual_environment_container.this
}
```

1. `terraform state list` shows the old addresses.
2. `terraform plan` must say "has moved" and **0 to destroy**.
3. Only then `terraform apply`. Take Proxmox snapshots first.

Always run Terraform from the **root**, never inside `modules/lxc/` (a module has no values of its own).

### 4.5 Remote state in Garage

MinIO's community edition no longer publishes binaries (its download URL returns `410 Gone`), so state is stored in **Garage**, a single-binary S3-compatible server.

**Install (in its own small container, as root):**

```bash
apt update && apt install -y wget openssl
wget -O /usr/local/bin/garage https://garagehq.deuxfleurs.fr/_releases/v2.5.0/x86_64-unknown-linux-musl/garage
chmod +x /usr/local/bin/garage
garage --version
```

Use the **prebuilt binary** link. A `.tar.gz` or a Git web page URL gives you source code or an HTML page, not a program (check with `ls -lh`: the binary is many MB).

`/etc/garage.toml` (generate the secret with `openssl rand -hex 32`):

```toml
metadata_dir = "/var/lib/garage/meta"
data_dir = "/var/lib/garage/data"
db_engine = "sqlite"
replication_factor = 1

rpc_bind_addr = "127.0.0.1:3901"
rpc_public_addr = "127.0.0.1:3901"
rpc_secret = "<64-HEX-CHARS>"

[s3_api]
s3_region = "garage"
api_bind_addr = "0.0.0.0:3900"
```

`/etc/systemd/system/garage.service`:

```ini
[Unit]
Description=Garage S3-compatible object storage
After=network.target

[Service]
ExecStart=/usr/local/bin/garage -c /etc/garage.toml server
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
mkdir -p /var/lib/garage/meta /var/lib/garage/data
chmod 600 /etc/garage.toml
systemctl daemon-reload && systemctl enable --now garage
garage status                                   # copy the node ID
garage layout assign -z dc1 -c 4G <NODE_ID>
garage layout apply --version 1
garage bucket create tfstate
garage key create terraform-key
garage bucket allow --read --write --owner tfstate --key terraform-key
garage key info terraform-key --show-secret     # store the secret safely
```

**Backend block** (in the `terraform {}` block of the root config):

```hcl
backend "s3" {
  bucket = "tfstate"
  key    = "homelab/terraform.tfstate"
  region = "garage"

  endpoints = { s3 = "http://<GARAGE_IP>:3900" }

  use_path_style              = true
  skip_credentials_validation = true
  skip_requesting_account_id  = true
  skip_metadata_api_check     = true
  skip_region_validation      = true
  skip_s3_checksum            = true
}
```

```bash
cp terraform.tfstate terraform.tfstate.pre-garage
export AWS_ACCESS_KEY_ID=<key id>
export AWS_SECRET_ACCESS_KEY=<secret>
terraform init -migrate-state        # answer "yes" to copy existing state
terraform plan                       # must show no unexpected changes
```

State locking (`use_lockfile = true`) is **not enabled**: it needs S3 conditional writes, and Garage support was not verified. It is optional for a single operator.

> Do not let this same Terraform configuration manage the container that stores its own state.

---

## 5. SSH and Ansible

Common first-boot SSH problems in cloned or minimal containers:

| Symptom | Cause | Fix |
|---|---|---|
| `ssh.service` fails, restart loop | Missing host keys or `/run/sshd` | `ssh-keygen -A; mkdir -p /run/sshd; systemctl restart ssh`, and `sshd -t` for the real error |
| `Permission denied (publickey)` | Key not installed, password login off | `ssh-copy-id user@host`; if needed `PasswordAuthentication yes` in `sshd_config` **and** in `/etc/ssh/sshd_config.d/*.conf` (cloud-init overrides) |
| nano warns about `//.local/share/nano` | Account home set to `/` | Create a real home, `usermod -d /home/<user> <user>` |
| Host key verification failed in CI | Non-interactive job, host not in `known_hosts` | `ssh-keyscan -H <hosts> >> ~/.ssh/known_hosts` for the job's user, or `ANSIBLE_HOST_KEY_CHECKING=False` (lab only) |

**Passwordless sudo** for the remote user (needed for `become: true` in pipelines; `Missing sudo password` otherwise):

```bash
sudo visudo -f /etc/sudoers.d/<user>
# <user> ALL=(ALL) NOPASSWD:ALL
```

Ansible connects as `ansible_user` from the inventory, or as the user running the job if none is set. Grant sudo to **that** user.

### Jenkins apt repository (signing key rotated)

The Jenkins repository switched to a new signing key. Using the old key gives `NO_PUBKEY 7198F4B714ABFC68`. Use the 2026 key and reference it with `signed-by`:

```yaml
- name: Add Jenkins signing key
  ansible.builtin.get_url:
    url: https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
    dest: /etc/apt/keyrings/jenkins-keyring.asc
    mode: "0644"
    force: true            # get_url skips existing files; this replaces a stale key

- name: Add Jenkins repo
  ansible.builtin.apt_repository:
    repo: "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/"
    filename: jenkins
    state: present
```

Remove leftovers before re-running: `/etc/apt/sources.list.d/jenkins.list` and old keyring files.

---

## 6. Jenkins

### 6.1 Install requirements

- Java 17, 21 or 25 on the master **and the same major version or newer on the agent** (`UnsupportedClassVersionError` means the agent's Java is older).
- `git` on both. `docker.io`, `kubectl` on the agent.
- On the agent's user: member of the `docker` group (`sudo usermod -aG docker jenkins`, then reconnect the node).

### 6.2 Master-agent connection

**Manage Jenkins → Nodes → New Node** (name `jenkins-agent`, label `jenkins-agent`):

- Launch method: SSH. **Host = the agent's IP** (a bare name like `jenkins` fails with "Name or service not known").
- Credential: **SSH Username with private key**. The **private** key goes into Jenkins; the matching **public** key goes into `authorized_keys` of that user on the agent. Use a dedicated key pair without a passphrase.
- Remote root directory: e.g. `/home/jenkins` (must exist and be writable).
- Host key verification: Non verifying for the lab, or pre-seed `known_hosts`.

### 6.3 Keep the master light

- **Manage Jenkins → Nodes → Built-In Node → Number of executors = 0.**
- Pin every job to the agent (`agent { label 'jenkins-agent' }`).
- Cap the master's heap: `sudo systemctl edit jenkins`

```ini
[Service]
Environment="JAVA_OPTS=-Xms256m -Xmx1024m"
```

Add `-Djdk.lang.Process.launchMechanism=VFORK` inside the same `JAVA_OPTS` string if you see `Failed to exec spawn helper` (a restart of Jenkins usually fixes this first).

Scans that ran on the master froze it. The signature was workspaces under `/var/lib/jenkins/workspace/...` (the agent uses `/home/jenkins/...`).

### 6.4 Use a Pipeline job, not Freestyle

A Freestyle job has no stages and ignores the Jenkinsfile. Create **New Item → Pipeline**, then **Definition: Pipeline script from SCM**, Git, your repo URL, branch `*/main`, script path `Jenkinsfile`. Jenkins checks the repo out onto the agent automatically (declarative pipelines run `checkout scm` for you). Files that exist only on the master are invisible to the agent, so keep everything in Git.

### 6.5 Triggers

GitHub cannot deliver webhooks to a private IP. Options:

- **Poll SCM** (simplest, no inbound access needed): `H/2 * * * *`.
- **Relay (smee.io):** a client on your LAN holds an outbound connection and forwards events to `http://192.168.20.32:8080/github-webhook/` (keep the trailing slash). Needs a current Node.js (Ubuntu 22.04's default is too old for `smee-client`). Channels are public, so treat the URL like a password. An unexplained `405` from Jenkins was seen with this relay and never fully diagnosed, so polling is the dependable choice.

Webhook settings if used: content type `application/json`, push events only, empty secret.

---

## 7. SonarQube Cloud

Jenkins side (global, done once):

- **Manage Jenkins → System → SonarQube servers**: name `sonarcloud`, URL `https://sonarcloud.io` (or your region), token credential (Secret text, ID `sonar-token`).
- **Manage Jenkins → Tools → SonarQube Scanner**: name `sonar-scanner`, install automatically.

SonarQube Cloud side: import the repo, choose **With Jenkins**, **turn Automatic Analysis off** (it rejects CI scans), note the organization and project keys.

`sonar-project.properties` in the repo root (the scanner does **not** read organization or project key from Jenkins' global settings):

```properties
sonar.organization=<ORG_KEY>
sonar.projectKey=<PROJECT_KEY>
sonar.sources=.
sonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,**/coverage/**,**/manifests/**
```

Use `sonar.token` (the old `sonar.login` was removed). The wrapper passes the URL and token:

```groovy
withSonarQubeEnv('sonarcloud') {
  sh "${scannerHome}/bin/sonar-scanner -Dsonar.host.url=\$SONAR_HOST_URL -Dsonar.token=\$SONAR_AUTH_TOKEN"
}
```

**Troubleshooting**

- `You must define ... sonar.projectKey, sonar.organization` → add the properties file and push it.
- `Failed to start the bridge server (300s timeout)` → the Node.js helper was starved. Give the agent more RAM and CPU, and exclude `node_modules`. Optionally set `sonar.nodejs.executable=/usr/bin/node` after installing a current Node.
- `curl -I https://sonarcloud.io` showing an HTTP/2 stream error is a network quirk, not a block (`--http1.1` works). If large uploads stall, lower the container MTU to 1400.
- A 307 redirect from the bare domain is normal.

---

## 8. Private registry

```bash
apt update && apt install -y docker.io
docker run -d --restart=always --name registry -p 5000:5000 \
  -v /var/lib/registry:/var/lib/registry registry:2
curl http://localhost:5000/v2/            # {}
```

In an LXC container, enable **Options → Features → Nesting and keyctl** before installing Docker.

The registry speaks plain HTTP, so every client must be told to allow it:

- **Jenkins agent**, `/etc/docker/daemon.json`, then `systemctl restart docker`:

```json
{ "insecure-registries": ["192.168.20.90:5000"] }
```

- **Kubernetes nodes:** `registries.yaml` (see §9).

Check contents: `curl http://192.168.20.90:5000/v2/_catalog` and `/v2/<name>/tags/list`.

> Every image reference must include the port: `192.168.20.90:5000/todo-app:<tag>`. Without `:5000`, Kubernetes tries HTTPS on port 443 (`connection refused`) and the registry mirror never applies.

---

## 9. Kubernetes (k3d)

### 9.1 Create the cluster once, with everything Jenkins needs

`registries.yaml`:

```yaml
mirrors:
  "192.168.20.90:5000":
    endpoint:
      - "http://192.168.20.90:5000"
```

```bash
k3d cluster create k3s-default --agents 2 \
  --api-port 192.168.20.10:6443 \
  --registry-config registries.yaml \
  --k3s-arg "--tls-san=192.168.20.10@server:*"
```

- `--api-port` pins the API address (the default is a random port).
- `--tls-san` puts that address in the API certificate, so remote `kubectl` verifies properly.
- `--registry-config` keeps the registry mirror across rebuilds. Copying the file into node containers with `docker cp` works for a quick test, but needs `docker exec ... mkdir -p /etc/rancher/k3s` first and is lost when the cluster is recreated.

Recreating the cluster **deletes everything in it**, including PVC data (`local-path` volumes live inside the node containers).

### 9.2 Kubeconfig for Jenkins

```bash
k3d kubeconfig get k3s-default > k3d-kubeconfig.yaml
grep server: k3d-kubeconfig.yaml      # must be https://192.168.20.10:6443, not 0.0.0.0
```

Test from the agent as the Jenkins user:

```bash
sudo -u jenkins env KUBECONFIG=/tmp/k3d-kubeconfig.yaml kubectl --request-timeout=60s get nodes
```

Store it in Jenkins as a **Secret file** credential with ID `k3d-kubeconfig`. Replace it whenever the cluster is recreated.

### 9.3 Manifests and Kustomize

The app needs more than a deployment: ConfigMaps, Services, the `todo-app-cache` PVC, the `todo-backend` deployment, the `todo-db` StatefulSet and the ingress. Starting a bare container (`kubectl create deployment`) will not work.

`deployments/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: project
resources:
  - <every YAML file to deploy, including PVC, Services, StatefulSet, Ingress>
images:
  - name: 192.168.20.90:5000/todo-app
    newName: 192.168.20.90:5000/todo-app
    newTag: "latest"
  - name: 192.168.20.90:5000/todo-backend
    newName: 192.168.20.90:5000/todo-backend
    newTag: "latest"
```

Rules that mattered:

- `resources:` lists **everything** to deploy, not only files that carry an image tag. `images:` rewrites tags separately.
- A kustomization can only reference files **at or below its own folder** (no `../`). Put it at a common parent, or use one per folder plus a top-level one that lists the folders.
- `name:` in `images:` must match the manifest's image (without tag) exactly, including the port.
- Quote the tag: `"18"`, not `18`.
- Never point `kubectl apply -f` at a folder holding source code (`package.json` fails with "apiVersion not set").
- Check before deploying: `kubectl kustomize deployments/ | grep -E "^kind:|image:"`.

---

## 10. The pipeline

`Jenkinsfile` (repo root):

```groovy
pipeline {
    agent { label 'jenkins-agent' }
    environment {
        REGISTRY = '192.168.20.90:5000'
    }
    stages {
        stage('SonarQube analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonarcloud') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.host.url=\$SONAR_HOST_URL -Dsonar.token=\$SONAR_AUTH_TOKEN"
                    }
                }
            }
        }
        stage('Build images') {
            steps {
                sh '''
                  docker build -t $REGISTRY/todo-app:$BUILD_NUMBER .
                  docker build -t $REGISTRY/todo-backend:$BUILD_NUMBER backend/
                '''
            }
        }
        stage('Push images') {
            steps {
                sh '''
                  docker push $REGISTRY/todo-app:$BUILD_NUMBER
                  docker push $REGISTRY/todo-backend:$BUILD_NUMBER
                '''
            }
        }
        stage('Deploy') {
            steps {
                withCredentials([file(credentialsId: 'k3d-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                      sed -i "s|newTag:.*|newTag: \\"$BUILD_NUMBER\\"|" deployments/kustomization.yaml
                      kubectl --request-timeout=60s apply -k deployments/
                      kubectl -n project rollout status deployment/todo-app --timeout=180s
                      kubectl -n project rollout status deployment/todo-backend --timeout=180s
                    '''
                }
            }
        }
    }
}
```

Notes:

- Adjust the Dockerfile paths and the kustomization folder to your repo.
- Deploy with the **build number**, never `:latest`. With `IfNotPresent` (the default for non-`latest`-tag-less behaviour of cached images), nodes reuse an image they already hold under the same tag, so a rebuilt `:latest` is silently ignored. A unique tag always forces a pull.
- While a scan is unreliable you can wrap the Sonar stage in `catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') { ... }`. Remove it once scans pass.
- A quality gate (stop on bad results) needs SonarQube Cloud to call back into Jenkins, which a private network blocks, so it is not used.

Verify after a run:

```bash
kubectl -n project get pods
kubectl -n project get deploy -o wide        # IMAGES column shows the new build number
curl http://192.168.20.90:5000/v2/todo-app/tags/list
```

---

## 11. Monitoring

Lean on purpose: Alertmanager, the pushgateway and node-exporter are disabled, so there are no host-level metrics (pod and container metrics remain).

The default Helm repositories (GitHub Pages) were unreachable from this network, so Prometheus is installed from its **OCI chart** and Grafana from a plain manifest.

`monitoring/prometheus-values.yaml`:

```yaml
alertmanager:
  enabled: false
prometheus-pushgateway:
  enabled: false
prometheus-node-exporter:
  enabled: false
kube-state-metrics:
  enabled: true
server:
  retention: 3d
  global:
    scrape_interval: 60s
    evaluation_interval: 60s
  persistentVolume:
    enabled: true
    size: 2Gi
  resources:
    requests: { cpu: 50m, memory: 256Mi }
    limits: { memory: 512Mi }
```

```bash
kubectl create namespace monitoring
helm install prometheus oci://ghcr.io/prometheus-community/charts/prometheus \
  -n monitoring -f monitoring/prometheus-values.yaml
```

`monitoring/grafana.yaml` is a ConfigMap (Prometheus data source), a 1 Gi PVC, a Deployment (image `grafana/grafana`, memory limit **768 Mi**; 256 Mi was too small for a recent Grafana major version) and a Service. The admin password is created separately:

```bash
kubectl -n monitoring create secret generic grafana-admin --from-literal=password='<CHOOSE>'
kubectl apply -f monitoring/grafana.yaml
```

If Grafana cannot write its data volume, add an init container that runs `chown -R 472:472 /var/lib/grafana`.

Access (temporary, keep the terminal open):

```bash
kubectl -n monitoring port-forward svc/grafana 3000:80 --address 0.0.0.0     # http://192.168.20.10:3000
kubectl -n monitoring port-forward svc/prometheus-server 9090:80 --address 0.0.0.0
```

In Prometheus, **Status → Targets** should show targets **UP**. In Grafana, **Data sources → Prometheus → Save & test**, then import a Kubernetes pod/workload dashboard that uses cAdvisor and kube-state-metrics (node-exporter dashboards will be empty).

---

## 12. Troubleshooting index

| Symptom | Likely cause | Fix |
|---|---|---|
| `403 Permission check failed (/vms/N, VM.Clone)` | Permission missing on the **user** | Add user permission for `terraform@pve` |
| Terraform asks for module variables | Running inside `modules/lxc/` | Run from the root |
| `Missing required argument` in module | Variable not passed | Pass it in the `module` block |
| `sshd` won't start | Host keys / `/run/sshd` | `ssh-keygen -A`, `mkdir -p /run/sshd` |
| `Missing sudo password` | Remote user needs a password for sudo | `NOPASSWD` drop-in for the **connecting** user |
| `NO_PUBKEY 7198F4B714ABFC68` | Old Jenkins repo key | 2026 key, `force: true`, `signed-by` |
| Agent log: `UnsupportedClassVersionError` | Agent Java older than master's | Install same or newer Java on the agent |
| Node offline: `Name or service not known` | Hostname instead of IP | Use the IP |
| `playbook could not be found` | Files only on the master | Put them in Git, Pipeline from SCM |
| Jenkins master freezes or restarts | Builds on the master, low RAM | Executors 0, pin jobs, cap heap, add RAM |
| `Failed to exec spawn helper` | JVM can't fork (stale JVM, low memory) | Restart Jenkins; `VFORK` option |
| Pipeline stops after the scan, `SUCCESS` | Freestyle job | Use a Pipeline job |
| `kubectl`/`helm` TLS handshake timeout | Cluster starved (I/O wait, swap) | Reduce load, stop agents, close Lens, wait |
| `ImagePullBackOff`, dial `:443` | Image lacks `:5000` | Fix the image name |
| `ImagePullBackOff`, HTTPS error | Missing `registries.yaml` | Add mirror, restart nodes |
| Old code still runs after rebuild | `:latest` cached on the node | Deploy unique tags |
| Pod `Pending`, PVC not found | Cluster recreated (volumes lost) | Re-apply the PVC manifest |
| `helm repo update`: connection refused | Network blocks GitHub Pages | OCI chart / plain manifests |
| Grafana restarts, port 3000 refused | Memory limit too low | Raise to 768 Mi, wait for first start |

---

## 13. Lessons learned: corrections made along the way

- **Read the first error, not the last.** Restart loops (`ssh`, `jenkins`, `k3s`) bury the cause; `sshd -t`, `journalctl`, `docker logs` and the first red line of a console log show it.
- **Verify the layer you are touching.** "Missing sudo password" looked like SSH but was sudo; `Permission denied` on VM clone looked like a token problem but was a missing user permission.
- **Don't trust a single metric.** `free -h` looked healthy while `vmstat` showed 50-70% I/O wait and constant swap-in.
- **Unverified claims were corrected.** A non-existent `kubectl kustomize edit` command, a dead MinIO download, a wrong example registry IP (`.35` vs the real `.90`) and a premature "content type" theory for the webhook were all replaced with checked alternatives.
- **Automation needs the same user everywhere.** SSH keys, `known_hosts`, sudo rights and the Docker group must all be set for the account the job actually runs as.
- **Immutable tags beat `latest`.** Image caching hid a fix for a long time.
- **Declarative beats manual.** Hand-created test deployments and manual `docker cp` edits were lost on cluster recreation or caused conflicts; the manifests, Jenkinsfile and Terraform in Git are the source of truth.
- **Right tool for the constraint.** When a product or network path was unavailable (MinIO, GitHub Pages, inbound webhooks, JFrog's heavy footprint), a smaller or outbound-only alternative was chosen (Garage, OCI chart, polling/relay, `registry:2`).

---

## 14. Security notes (lab-grade; harden before exposing anything)

- The registry is **plain HTTP with no authentication**. Keep it on the private VLAN; never expose port 5000.
- `NOPASSWD:ALL` sudo and `ANSIBLE_HOST_KEY_CHECKING=False` / non-verifying host keys are lab shortcuts.
- Garage and the Jenkins endpoint use HTTP. Do not expose Jenkins to the internet.
- Never commit tokens, the kubeconfig, private keys, Terraform variables with secrets, or the Grafana password. Store them as Jenkins credentials or Kubernetes Secrets.
- smee.io channels are public; anyone with the URL can read and send events.
- Treat any token or private key that was pasted anywhere public as compromised and rotate it.
- Prefer a narrower role than `Administrator` for the Terraform token once things work.

---

## 15. Backups and recovery

- Proxmox backups/snapshots of: Jenkins master, registry, Garage, `k3s-server`, Ansible controller. Take them while everything works.
- Not in Git and worth protecting: Jenkins configuration (credentials, node, server settings), registry images (rebuildable from the pipeline), Garage data (holds Terraform state).
- `terraform state pull > state-backup.json` occasionally.
- k3d PVC data is lost when the cluster is recreated; the database needs its own backup if the data matters.

---

## 16. Known gaps and unverified items

- **Grafana:** confirm it reaches `Running` and answers on port 3000 after the memory-limit change.
- **State locking:** not enabled (S3 conditional-write support in Garage not verified). Try `use_lockfile = true` and revert if it errors.
- **smee relay:** the `405` from Jenkins was not conclusively explained; polling is the dependable trigger.
- **Addresses marked *(verify)*** and the placeholders (`<PROXMOX_IP>`, `<ANSIBLE_IP>`, `<GARAGE_IP>`, `<CT_TEMPLATE_ID>`) must match your network.
- **Not implemented**: authentication on the registry, TLS, an ingress for Grafana, SonarQube quality-gate enforcement, alerting.

---

## 17. Command cheat sheet

```bash
# Host health
vmstat 2 5; free -h; dmesg -T | grep -iE "oom|killed process" | tail

# Terraform (from the root)
terraform init -migrate-state && terraform plan && terraform apply
terraform state list

# Registry
curl http://192.168.20.90:5000/v2/_catalog
curl http://192.168.20.90:5000/v2/todo-app/tags/list

# Kubernetes
kubectl -n project get pods,deploy,pvc,svc,ingress
kubectl -n project logs deploy/todo-app --previous
kubectl kustomize deployments/ | grep -E "^kind:|image:"
kubectl --request-timeout=120s get ns

# Jenkins service
sudo systemctl status jenkins
sudo journalctl -u jenkins --no-pager | tail -40
```
