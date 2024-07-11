### SELinux的类型及其作用

在SELinux（Security-Enhanced Linux）中，类型（Type）是访问控制的核心部分。每个进程和文件都有一个类型标签，通过策略来定义这些类型之间的交互规则。

#### 什么是类型？

类型（Type）是SELinux策略的一部分，标识资源的安全属性。类型用于定义哪些进程可以访问哪些资源，以及可以进行何种操作。类型标签应用于进程、文件、目录、设备等各种对象。

### `my_web_container_t` 类型的示例

假设我们定义了一个新的类型`my_web_container_t`，用于一个特定的Web服务器容器。这个类型标签会限制该容器进程的访问权限，确保其只能访问被授权的资源。

#### 创建和使用自定义类型

1. **定义类型**：
   创建策略文件`my_web_container.te`，内容如下：
   ```te
   policy_module(my_web_container, 1.0)

   type my_web_container_t;
   type my_web_container_exec_t;
   init_daemon_domain(my_web_container_t, my_web_container_exec_t)
   ```

   - `type my_web_container_t;`：定义了一个新的类型标签`my_web_container_t`。
   - `type my_web_container_exec_t;`：定义了一个用于可执行文件的类型标签`my_web_container_exec_t`。
   - `init_daemon_domain(my_web_container_t, my_web_container_exec_t)`：设置了进程类型`my_web_container_t`的初始域，将`my_web_container_exec_t`类型的可执行文件标识为该域的入口点。

2. **编译和安装策略模块**：
   ```bash
   checkmodule -M -m -o my_web_container.mod my_web_container.te
   semodule_package -o my_web_container.pp -m my_web_container.mod
   semodule -i my_web_container.pp
   ```

3. **运行容器并应用类型**：
   ```bash
   docker run -it --name my_web_container --security-opt label=type:my_web_container_t nginx
   ```

### 资源限制示例

#### 文件系统访问控制

通过定义和应用特定的类型标签，SELinux可以限制容器对文件系统的访问。例如：

1. **为特定目录设置文件类型**：
   ```bash
   semanage fcontext -a -t httpd_sys_content_t '/var/www/html(/.*)?'
   restorecon -Rv /var/www/html
   ```

2. **限制容器访问**：
   通过SELinux策略，可以确保运行在`my_web_container_t`类型下的进程只能访问被标记为`httpd_sys_content_t`类型的文件和目录。

#### 网络访问控制

类似地，SELinux可以限制进程的网络访问。例如，限制某些类型的进程只能使用特定的端口或网络接口。

### 如何实现限制

SELinux通过策略文件来定义类型之间的交互规则，这些规则可以非常细粒度地控制资源访问。例如：

```te
allow my_web_container_t httpd_sys_content_t:file { read write };
```

这条规则允许类型为`my_web_container_t`的进程对类型为`httpd_sys_content_t`的文件进行读写操作。
