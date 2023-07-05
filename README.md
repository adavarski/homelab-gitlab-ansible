
Gitlab server install via ansible
$ cd gitlab-ansible
$ ansible-playbook -i ./inventory.ini gitlab.yml


## devops-server-gitlab-install

### Install GitLab 

```
### Setup DNS server:
$ grep -ri gitlab.devops /etc/bind/* 
/etc/bind/forward.davar.com:gitlab.devops  IN       A       192.168.1.99
/etc/bind/reverse.davar.com:99       IN      PTR     gitlab.devops.davar.com.
$ sudo systemctl restart bind9

### Install GitLab on devops server: 
root@devops:~# grep 100 /etc/resolv.conf
nameserver 192.168.1.100

$ ansible-playbook -i ./inventory.ini gitlab.yml
```

### gitlab-runner setup (docker-based)
# curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb"
# dpkg -i gitlab-runner_amd64.deb
# cp gitlab-runner/config.toml /etc/gitlab-runner/config.toml 
# systemctl restart gitlab-runner 


### Gitlab-Runners Notes/Examples:

We can have for example 2(n) shared gitlab-runners: with docker & linux tags

1. --executor = "shell" --> --tag-list linux

2. --executor "docker" --> --tag-list docker 

3. --executor kubernetes (TBD) 


Ref: https://gitlab.devops.davar.com/admin/runners/4 ---> Tags view/edit

Ref: https://docs.gitlab.com/runner/register/ 

Ref: shell executor example (tag:linux): https://docs.gitlab.com/runner/executors/shell.html

```
...
[[runners]]
  name = "shell executor runner"
  executor = "shell"
  shell = "bash"
  ...
  --tag-list linux
...
```

Ref: docker executor example: 

```
      gitlab-runner register --executor docker  \
      ...
      --tag-list docker
...

```

Examples:

```
1.k8s app deploy (pipelines deploy stage) example:

1.1.using shell executor (tags: linux)

run on gitlab-runner -> executor:shell -> tag:linux. gitlab-runner = shell executor, not docker executor to use prconfigured KUBECONFIG file (file: /etc/k8s/prod) -->  kubectl --kubeconfig /etc/k8s/prod --token="$KUBE_TOKEN" -n $K8SNAMESPACE set image $K8SDEPLOYMENT $K8SREPLICA=$CI_REGISTRY_IMAGE:${CI_PIPELINE_ID}

prod_k8s_deploy:
  stage: deploy
  variables:
    GIT_STRATEGY: none
  only:
    - production
  tags:
    - linux
  when: manual
  environment:
    name: prod
  script:
    - kubectl --kubeconfig /etc/k8s/prod --token="$KUBE_TOKEN" -n $K8SNAMESPACE set image $K8SDEPLOYMENT $K8SREPLICA=$CI_REGISTRY_IMAGE:${CI_PIPELINE_ID}
    - kubectl --kubeconfig /etc/k8s/prod_new --token="$KUBE_NEW_TOKEN" -n $K8SNAMESPACE set image $K8SDEPLOYMENT $K8SREPLICA=$CI_REGISTRY_IMAGE:${CI_PIPELINE_ID}

1.2. using docker executor (tags: docker)

K8S_DEV variable value is k8s KUBECONFIG file content:

  script:
    - wget -q https://dl.k8s.io/release/v1.22.0/bin/linux/amd64/kubectl -O /usr/local/bin/kubectl && chmod 755 /usr/local/bin/kubectl
    - mkdir -p /etc/k8s/
    - echo "$K8S_DEV" > /etc/k8s/k8s-dev
    - kubectl --kubeconfig /etc/k8s/k8s-dev -n $K8SNAMESPACE set image $K8SDEPLOYMENT $K8SREPLICA=$CI_REGISTRY_IMAGE:${CI_COMMIT_REF_NAME}-${CI_PIPELINE_ID}

2.AWS example:

default:
  tags:
    - docker

3. ansible example (k8s deploy via nasible playbook:kubeadm) : k8s chmod 0600 $SSH_PRIVATE_KEY -> https://docs.gitlab.com/runner/executors/shell.html

3.1. shell executor example (tags: linux): 
 script:
    - cd ${CI_PROJECT_DIR}
    - cat ${CI_PROJECT_DIR}/inventory/* > ${CI_PROJECT_DIR}/inventory.ini
    - chmod 0600 $SSH_PRIVATE_KEY
    - ansible-playbook --private-key $SSH_PRIVATE_KEY -i ${CI_PROJECT_DIR}/inventory.ini playbook.yml

3.2. docker executor example (tags: docker) 

  script:
    - cd ${CI_PROJECT_DIR}
    - cat ${CI_PROJECT_DIR}/inventory/* > ${CI_PROJECT_DIR}/inventory.ini
    - mkdir -p ~/.ssh  
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 400 ~/.ssh/id_rsa
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    - export ANSIBLE_HOST_KEY_CHECKING=False
    - ansible-playbook --private-key ~/.ssh/id_rsa -u root -i ${CI_PROJECT_DIR}/inventory.ini ./postgresql_cluster/deploy_pgcluster.yml
```

### Notes: self signed certs and gitlab-runner resolver (shared gitlab-runner: docker executor examples); terraform state;  GitLab docker registry, docker insecure registry; GitLab CI/CD pipelines: Ansible; etc.  

GitLab CI/CD examples: 

- Ref1: https://github.com/adavarski/devops-server-docker-ansible
- Ref2: https://github.com/adavarski/devops-server-postgres-ha-prod
- Ref3: https://github.com/adavarski/devops-server-db-proxy

```
1.For a shared runner, as Administrator go to the GitLab Admin Area and click Overview > Runners (get token: eztv9hLB4tP81jVy5WkD)

      curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb"
      dpkg -i gitlab-runner_amd64.deb


      gitlab-runner register --executor docker  \
        -u https://gitlab.devops.davar.com/ \
        --tls-ca-file=/etc/gitlab/ssl/gitlab.devops.davar.com.crt \
        --tls-key-file=/etc/gitlab/ssl/gitlab.devops.davar.com.key \
        --non-interactive \
        -r eztv9hLB4tP81jVy5WkD \
        --docker-privileged=true \
        --docker-pull-policy=always \
        --docker-shm-size=268435456 \
        --docker-volumes='/cache' \
        --docker-image="docker:19.03.12"

Edit file and Fix 

- gitlab-runner resolver fix:

    extra_hosts = ["gitlab.devops.davar.com:192.168.1.99"]
 
- Add "/var/run/docker.sock:/var/run/docker.sock" and tls_verify = false to build docker images on gitlab-runner via GitLab CI/CD pipeline (Ref: error during connect: Post http://docker:2375/v1.40/auth: dial tcp: lookup docker on 192.168.1.1:53: no such host) --->  Ref:  https://github.com/adavarski/devops-server-db-proxy

root@devops:/etc/k8s# cat /etc/gitlab-runner/config.toml 
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "devops"
  url = "https://gitlab.devops.davar.com/"
  token = "3f2gjUryD7_PZyws5TTD"
  tls-ca-file = "/etc/gitlab/ssl/gitlab.devops.davar.com.crt"
  tls-key-file = "/etc/gitlab/ssl/gitlab.devops.davar.com.key"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "docker:19.03.12"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    pull_policy = ["always"]
    shm_size = 268435456
    extra_hosts = ["gitlab.devops.davar.com:192.168.1.99"]

root@devops:~/.ssh# systemctl restart gitlab-runner

Ref3: https://github.com/adavarski/devops-server-db-proxy (for k8s resolver fixes):

$ curl -sfL https://get.k3s.io | sh -
$ sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/k3s-config
$ sudo chmod 755 ~/.kube/k3s-config
$ sed -i "s/127.0.0.1/192.168.1.100/"  ~/.kube/k3s-config
$ export KUBECONFIG=~/.kube/k3s-config
$ kubectl cluster-info
$ sudo cp  /var/lib/rancher/k3s/server/manifests/coredns.yaml ./coredns-fixes.yaml
$ vi coredns-fixes.yaml 
$ sudo chown $USER: coredns-fixes.yaml 
$ sudo diff coredns-fixes.yaml /var/lib/rancher/k3s/server/manifests/coredns.yaml 
75,79d74
<     davar.com:53 {
<         errors
<         cache 30
<         forward . 192.168.1.100
<     }
$ kubectl apply -f coredns-fixes.yaml
serviceaccount/coredns unchanged
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
clusterrole.rbac.authorization.k8s.io/system:coredns unchanged
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
clusterrolebinding.rbac.authorization.k8s.io/system:coredns unchanged
configmap/coredns unchanged
deployment.apps/coredns configured
service/kube-dns unchanged

2.Terraform state (GitLab CI/CD pipelines)

###! Docs: https://docs.gitlab.com/ee/administration/terraform_state
gitlab_rails['terraform_state_enabled'] = true
gitlab_rails['terraform_state_storage_path'] = "/var/opt/gitlab/gitlab-rails/shared/terraform_state"
#gitlab_rails['terraform_state_object_store_enabled'] = true
#gitlab_rails['terraform_state_object_store_remote_directory'] = 'davar-gitlab-terraform'

root@devops:~/.ssh# gitlab-ctl reconfigure

GitLab CI/CD pipelines (add this):

terraform {
  backend "http" {
     skip_cert_verification = true
  }
}

3.GitLab Docker registry

gitlab.rb

    - { regex: "registry_external_url", line: "registry_external_url 'https://{{ domain }}:2053'" }

GitLab CI/CD pipelines(example):

--skip-tls-verify
    - /kaniko/executor --skip-tls-verify --context ${CI_PROJECT_DIR} --dockerfile ${CI_PROJECT_DIR}/Dockerfile --destination ${CI_REGISTRY_IMAGE}:latest

Docker daemon setup:

root@devops:~/.ssh# cat /etc/docker/daemon.json 
{ "insecure-registries" : ["gitlab.devops.davar.com:2053"] }

4.Ansible (GitLab CI/CD pipelines)

Add private key variable SSH_PRIVATE_KEY to gitlab (individual project or group)

# Paste the PRIVATE key into a gitlab variable (example: SSH_PRIVATE_KEY). Pay attention to the linebreak at the end when pasting !!!

configure:ansible:
  stage: configure
  dependencies:
    - apply:terraform
  image: gitlab.devops.davar.com:2053/root/docker-ansible:latest
  script:
    - cd ${CI_PROJECT_DIR}
    - cat ${CI_PROJECT_DIR}/inventory/* > ${CI_PROJECT_DIR}/inventory.ini
    - mkdir -p ~/.ssh  
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 400 ~/.ssh/id_rsa
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    - export ANSIBLE_HOST_KEY_CHECKING=False
    - ansible-playbook --private-key ~/.ssh/id_rsa -u root -i ${CI_PROJECT_DIR}/inventory.ini ./postgresql_cluster/deploy_pgcluster.yml
  allow_failure: true

```

Ref (HOWTO):

- https://otodiginet.com/software/how-to-install-gitlab-community-edition-ce-on-ubuntu-20-04-lts/
- https://www.linuxhelp.com/how-to-install-gitlab-on-linux-mint-20
- https://docs.gitlab.com/ee/administration/terraform_state & https://gitlab.devops.davar.com/help/user/infrastructure/iac/terraform_state.md
- https://docs.gitlab.com/runner/register/ & https://docs.gitlab.com/runner/executors/shell.html
- https://docs.gitlab.com/ee/user/packages/container_registry/

REF: gitlab-runners-infra: cloudinit.yaml example (for GitLab CI/CD pipelines with terraform on Hetzner Cloud/AWS/etc.) 
```
#cloud-config                                                                                                                                                                                                [40/92]
groups:
- docker
users:
- name: gitlab-runner
  groups: docker
apt:
  sources:
    docker.list:
      source: 'deb [arch=amd64] https://download.docker.com/linux/ubuntu $RELEASE stable'
      keyid: 0EBFCD88 

package_upgrade: true
package_update: true

packages:
- debian-archive-keyring
- apt-transport-https
- ca-certificates
- software-properties-common
- docker-ce

write_files:
  - owner: root:root
    path: /etc/cron.d/your_cronjob
    content: "* 5 * * * root (/usr/bin/docker ps --filter status=dead --filter status=exited -aq   |  /usr/bin/xargs /usr/bin/docker rm -v 2> /dev/null) || true"
  - owner: root:root
    path: /root/register.sh
    content: |
      curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb"
      dpkg -i gitlab-runner_amd64.deb
      ip route add 172.17.0.0/16 via 172.16.0.1 dev ens10
      ip route add 172.18.0.0/16 via 172.16.0.1 dev ens10
      ip route add 172.19.0.0/16 via 172.16.0.1 dev ens10
      gitlab-runner register --executor docker  \
        -u https://gitlab.gostudent.cloud/ \
        --non-interactive \
        -r $gitlab_registration_token$ \
        --docker-privileged=true \
        --docker-pull-policy=always \
        --docker-shm-size=268435456 \
        --docker-volumes='/cache' \
        --docker-image="docker:19.03.12"
runcmd:
  - [/bin/bash, /root/register.sh]
  
### main.tf  

VMs terraform (example: Hetzner Cloud):

# SSH key to provision the runner
data "hcloud_ssh_keys" "all_keys" {
}

# Get the network id to attach the runner to
data "hcloud_network" "network" {
  name = "vpc"
}

data "local_file" "cloudinit" {
  filename = "cloudinit.yml"
}

# Create the bastion server to run the gitlab runner on and to create docker-machine instances from
resource "hcloud_server" "infra_runner" {
  count       = var.num_runners
  name        = "gitlab-infra-runner-${count.index}"
  image       = "ubuntu-20.04"
  server_type = "cx21"

  ssh_keys    = data.hcloud_ssh_keys.all_keys.ssh_keys.*.id
  user_data   = replace(data.local_file.cloudinit.content, "$gitlab_registration_token$", var.gitlab_registration_token)
}

resource "hcloud_server_network" "wireguard_node_network" {
  count      = var.num_runners
  server_id  = hcloud_server.infra_runner[count.index].id
  network_id = data.hcloud_network.network.id
}

### vars.tf
variable "hcloud_token" {
  type        = string
  description = "The api token to use for connecting to Hetzner Cloud."
}

variable "gitlab_registration_token" {
  type        = string
  description = "The token to register the runner with on gitlab."
}

variable "num_runners" {
  type        = number
  description = "The number of runners to be launched."
  default     = 1
}

### terraform.tfvars
hcloud_token    = "XXXXXXX"

### versions.tf
terraform {
  required_providers {
    hcloud = {
      source = "hetznercloud/hcloud"
    }
  }
  required_version = ">= 0.13"
}

### terraform.tf
terraform {
  backend "http" {
  }
}

# Configure the Hetzner Cloud Provider
provider "hcloud" {
  token   = var.hcloud_token
}

### .gitignore
# Terraform local state (including secrets in backend configuration)
.terraform
terraform.tfstate.*.backup
# Ansible retry-files
*.retry

# Do not index inventory.ini files - they will be created automatically when needed
inventory.ini

### README.md
Gitlab Infrastructure Runner
=========

This gitlab runner is a docker type runner prepared to run and deploy infrastructure projects.
The runner is deployed attached to the shared services network and leverages the site-to-site VPN to connect to resources in all environments.

The runner is designed to function as *group* level runner, reserved for infrastructure projects only.


Local Use

To use the repository locally, you must first initialize the remote terraform state.
Please replace `GITLAB_USERNAME` and `GITLAB_API_TOKEN` with your respective credentials.

When running terraform commands you will be asked for the `gitlab_registration_token` which you need to have access to.

terraform init \
    -backend-config="address=https://gitlab.gostudent.cloud/api/v4/projects/50/terraform/state/default" \
    -backend-config="lock_address=https://gitlab.gostudent.cloud/api/v4/projects/50/terraform/state/default/lock" \
    -backend-config="unlock_address=https://gitlab.gostudent.cloud/api/v4/projects/50/terraform/state/default/lock" \
    -backend-config="username=GITLAB_USERNAME" \
    -backend-config="password=GITLAB_API_TOKEN" \
    -backend-config="lock_method=POST" \
    -backend-config="unlock_method=DELETE" \
    -backend-config="retry_wait_min=5"

```

