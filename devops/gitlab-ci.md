# Gitlab Ci 配置

# 添加runner

* [在 Group 上添加runner](https://10.10.9.29:1443/groups/moss/-/runners/new)
* set runner tags to 'golang,moss'
* register runner
    ```
    gitlab-runner register  --url https://10.10.9.29:1443  --token glrt-bnzxnQ9PyDAHb1V2FJHH
    ```