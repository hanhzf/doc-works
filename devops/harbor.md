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
