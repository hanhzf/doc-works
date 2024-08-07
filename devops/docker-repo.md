# 如何配置docker-repo

目前，在国内很多repo不可以访问，需要重新配置镜像地址方可下载镜像。

* 修改docker配置文件 `/etc/docker/daemon.json`
    ```json
    {
        "registry-mirrors":  [
            "https://docker.itelyou.cf",
            "https://huecker.io",
            "https://hub.atomgit.com"
        ]
    }
    ```
* 重启docker
    ```bash
    sudo systemctl restart docker
    ```
