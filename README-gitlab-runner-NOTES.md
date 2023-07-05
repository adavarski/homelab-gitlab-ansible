# Gitlab-Runners Notes & CI/CD Examples:

## Executor examples

We can have for example (n) shared gitlab-runners: with docker, linux, k8s tags

```
--executor = "shell" --> --tag-list linux

--executor "docker" --> --tag-list docker 

--executor kubernetes --> --tag-list k8s 
```

Ref: https://docs.gitlab.com/runner/register/ 

### shell executor example (tag:linux): https://docs.gitlab.com/runner/executors/shell.html

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

### docker executor example: 

```
      gitlab-runner register --executor docker  \
      ...
      --tag-list docker
...

```

### k8s executor example (TBD)


## CI/CD Examples:

```
1.k8s app deploy (pipelines deploy stage) example:

1.1.using shell executor (tags: linux)

run on gitlab-runner -> executor:shell -> tag:linux. gitlab-runner = shell executor, use preconfigured KUBECONFIG file (file: /etc/k8s/prod) -->  kubectl --kubeconfig /etc/k8s/prod --token="$KUBE_TOKEN" -n $K8SNAMESPACE set image $K8SDEPLOYMENT $K8SREPLICA=$CI_REGISTRY_IMAGE:${CI_PIPELINE_ID}

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

3. ansible example (k8s deploy via ansible playbook:kubeadm) : k8s chmod 0600 $SSH_PRIVATE_KEY -> https://docs.gitlab.com/runner/executors/shell.html

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
### Shared gitlab-runner: self signed certs and gitlab-runner resolver fixes, docker executor examples; terraform state;  GitLab docker registry, docker insecure registry; GitLab CI/CD pipelines: Ansible; etc.  

Ref: https://github.com/adavarski/devops-server-postgres-ha-prod

```
### Install Shared gitlab-runner:
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

- gitlab-runner resolver Fix:

    extra_hosts = ["gitlab.devops.davar.com:192.168.1.99"]
 
- Add "/var/run/docker.sock:/var/run/docker.sock" and tls_verify = false to build docker images on gitlab-runner via GitLab CI/CD pipeline (Ref: error during connect: Post http://docker:2375/v1.40/auth: dial tcp: lookup docker on 192.168.1.1:53: no such host) 

# cat /etc/gitlab-runner/config.toml 
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

# systemctl restart gitlab-runner

- k8s resolver fixes:

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

2.Terraform state (GitLab CI/CD pipelines)

###! Docs: https://docs.gitlab.com/ee/administration/terraform_state
gitlab_rails['terraform_state_enabled'] = true
gitlab_rails['terraform_state_storage_path'] = "/var/opt/gitlab/gitlab-rails/shared/terraform_state"
#gitlab_rails['terraform_state_object_store_enabled'] = true
#gitlab_rails['terraform_state_object_store_remote_directory'] = 'davar-gitlab-terraform'

# gitlab-ctl reconfigure

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




