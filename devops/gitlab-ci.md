# Gitlab Ci 配置

# 添加runner

* [在 Group 上添加runner](https://10.10.9.29:1443/groups/moss/-/runners/new)
* set runner tags to 'golang,moss'
* register runner
    ```
    [root@scci EasyRSA-3.0.4]# gitlab-runner register  --url https://10.10.9.29:1443  --token glrt-PkAqrYs6RtyrysWGuNU3
    Runtime platform                                    arch=amd64 os=linux pid=5235 revision=9882d9c7 version=17.2.1
    Running in system-mode.

    Enter the GitLab instance URL (for example, https://gitlab.com/):
    [https://10.10.9.29:1443]:
    Verifying runner... is valid                        runner=PkAqrYs6R
    Enter a name for the runner. This is stored only in the local config.toml file:
    [scci]: moss
    Enter an executor: custom, parallels, docker, docker+machine, shell, ssh, virtualbox, docker-windows, kubernetes, docker-autoscaler, instance:
    docker
    Enter the default Docker image (for example, ruby:2.7):
    golang:1.22
    Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

    Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml"
    ```