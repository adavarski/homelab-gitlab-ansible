
## Gitlab server with Docker Registry & Terraform state enabled

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
```
# curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb"
# dpkg -i gitlab-runner_amd64.deb
# cp gitlab-runner/config.toml /etc/gitlab-runner/config.toml 
# systemctl restart gitlab-runner 
```

### [GitLab Runners Notes & CI/CD Examples](./README-gitlab-runner-NOTES.md)
