# harbor

* get onlne install package
  ```
  wget https://github.com/goharbor/harbor/releases/download/v2.7.0/harbor-online-installer-v2.7.0.tgz
  tar zxf harbor-online-installer-v2.7.0.tgz -C /srv/harbor/install
  cp harbor.yml.tmpl harbor.yml
  ```
* prepare certificates with EasyRsa, please refer to `1-cert.md`
* edit harbor configuration file
  ```
  hostname: 10.10.9.29

  # https related config
  https:
    # https port for harbor, default is 443
    port: 2443
    # The path of cert and key files for nginx
    certificate: /srv/harbor/cert/harbor.crt
    private_key: /srv/harbor/cert/harbor.key
    
  harbor_admin_password: zhu88jie
  ```
  the default auth is `admin/zhu88jie`

* install harbor
  ```
  cd /srv/harbor/install
  ./prepare # this command will download harbor images
  docker compose up -d
  ```
* open harbor browser
  ```
  https://10.10.9.29:2443
  ```
* push image to harbor

  edit `/etc/docker/daemon.json` to trust unauthorized ca of harbor:

  ```
  {
    "registry-mirrors":  [
      "https://docker.itelyou.cf",
      "https://huecker.io",
      "https://hub.atomgit.com"
    ],
    "insecure-registries":[
      "10.10.9.29:2443"
    ]
  }
  ```

  login harbor and push image:

  ```
  docker login https://10.10.9.29:2443
  docker tag golang:1.22-docker 10.10.9.29:2443/library/golang:1.22-docker
  docker push 10.10.9.29:2443/library/golang:1.22-docker
  ```
* update gitlab-runner's docker image if needed

  配置 `gitlab-runner` 时的image指定私有仓库的 baseimage
  
  ```
  vim /etc/gitlab-runner/config.toml
  
  [runners.docker]
  tls_verify = false
  image = "10.10.9.29:2443/library/golang:1.22-docker"
  ```

    restart gitlab runner:
    `service gitlab-runner restart`
  
## harbor ldap 认证配置

```
1.	LDAP URL：ldap://10.10.9.39
2.	LDAP搜索DN：cn=readonly,dc=capitaleco,dc=com
3.	LDAP搜索密码：mypassword
4.	LDAP基础DN：ou=people,dc=capitaleco,dc=com
5.	LDAP用户UID：mail
6.	LDAP过滤器：(objectClass=inetOrgPerson)
```