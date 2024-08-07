# certificates

Currently all the existing certs are located under **/srv/cert/EasyRSA-3.0.4** on node **devops01(172.20.0.2)**.

### init PKI and generate ca
* mkdir -p /srv/cert && cd /srv/cert
* wget -P ~/ https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz
* cd ~
* tar xvf EasyRSA-3.0.4.tgz
* cd ~/EasyRSA-3.0.4/
* cp vars.example vars
* cat vars

    ```
    set_var EASYRSA_REQ_COUNTRY    "cn"
    set_var EASYRSA_REQ_PROVINCE   "beijing"
    set_var EASYRSA_REQ_CITY       "beijing"
    set_var EASYRSA_REQ_ORG        "capital"
    set_var EASYRSA_REQ_EMAIL      "devops@capital.com"
    set_var EASYRSA_REQ_OU         "devops"
    ```
* ./easyrsa init-pki

  Run this script with the init-pki option to initiate the **public key infrastructure** on the CA server.
  
  
* ./easyrsa build-ca nopass

  This will build the CA and create two important files — **ca.crt** and **ca.key** — which make up the public and private sides of an SSL certificate.

  set common name to capital: 
  `Common Name (eg: your user, host, or server name) [Easy-RSA CA]:capital`

### generate gitlib certificate


As we have disabled letencrypt, we need to prepare the gitlab certs ourselves, follow [generate certificate](./cert.md) to create certificate for gitlab:

```
cd /srv/EasyRSA-3.0.4

./easyrsa --subject-alt-name="IP:10.10.9.29" gen-req 10.10.9.29 nopass

./easyrsa --subject-alt-name="IP:10.10.9.29" sign-req client 10.10.9.29

# 确认证书是否包含san信息
openssl x509 -in pki/issued/10.10.9.29.crt -noout -text | grep -A1 "Subject Alternative Name"

root@devops01:/srv/EasyRSA-3.0.4# ls pki/private/
10.10.9.29.key  ca.key  client.key  ldap.key  ldap.key.kF6IHtj4Vx  server.key

root@devops01:/srv/EasyRSA-3.0.4# ls pki/issued/
10.10.9.29.crt  client.crt  ldap.crt  server.crt

```

编辑 `/srv/cert/EasyRSA-3.0.4/x509-types`，增加 clientAuth 信息:
```
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = clientAuth,serverAuth
```

否则 gitlab-runner register 报错：
```
ERROR: Verifying runner... failed                   
runner=PkAqrYs6R status=couldn't execute POST against https://10.10.9.29:1443/api/v4/runners/verify: Post "https://10.10.9.29:1443/api/v4/runners/verify": tls: failed to verify certificate: x509: certificate specifies an incompatible key usage
PANIC: Failed to verify the runner.
```

copy the file to folder */data/gitlab/config/ssl* :
```
rm -rf /srv/gitlab/config/ssl/10.10.9.29.* && cp pki/issued/10.10.9.29.crt pki/private/10.10.9.29.key /srv/gitlab/config/ssl/
root@devops01:/data/gitlab/config/ssl# ls
10.10.9.29.crt  10.10.9.29.key
```

### add ca to linux

```
cp /srv/cert/EasyRSA-3.0.4/pki/ca.crt  /etc/pki/ca-trust/source/anchors/devops-ca.crt
sudo update-ca-trust
sudo systemctl restart gitlab-runner
```

### note

gitlab-runner 要求证书中必须有san信息。