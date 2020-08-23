## docker-ce安装

## 1.限制容器的内存使用量
#### 描述

默认情况下，Docker主机上的所有容器均等地共享资源。 通过使用Docker主机的资源管理功能（例如内存限制），您可以控制容器可能消耗的内存量。
默认情况下，容器可以使用主机上的所有内存。 您可以使用内存限制机制来防止由于一个容器消耗主机的所有资源而导致的服务拒绝，从而使同一主机上的其他容器无法执行其预期的功能。 对内存没有限制可能会导致一个问题，即一个容器很容易使整个系统不稳定并因此无法使用。
#### 加固
仅使用所需的内存来运行容器。 始终使用'--memory'参数运行容器。 您应该按以下方式启动容器：
```bash
docker run --interactive --tty --memory 256m <Container Image Name or ID>
```
## 2.为Docker启用内容信任 
#### 描述

默认情况下禁用内容信任。 您应该启用它。
内容信任提供了将数字签名用于发送到远程Docker注册表和从远程Docker注册表接收的数据的功能。 这些签名允许客户端验证特定图像标签的完整性和发布者。 这确保了容器图像的出处

#### 加固建议

要在bash shell中启用内容信任，请输入以下命令：export DOCKER_CONTENT_TRUST=1 或者，在您的配置文件中设置此环境变量，以便在每次登录时启用内容信任。 内容信任目前仅适用于公共Docker Hub的用户。 当前不适用于Docker Trusted Registry或私有注册表。

## 3.将容器的根文件系统挂载为只读
#### 描述
容器的根文件系统应被视为“黄金映像”，并且应避免对根文件系统的任何写操作。 您应该显式定义用于写入的容器卷。
您不应该在容器中写入数据。 属于容器的数据量应明确定义和管理。 在管理员控制他们希望开发人员在何处写入文件和错误的许多情况下，这很有用。
#### 加固建议

添加“ --read-only”标志，以允许将容器的根文件系统挂载为只读。 可以将其与卷结合使用，以强制容器的过程仅写入要保留的位置。 您应该按以下方式运行容器：
```bash
docker run --interactive --tty --read-only --volume <writable-volume> <Container 
```
## 4.限制容器之间的网络流量
#### 描述

默认情况下，同一主机上的容器之间允许所有网络通信。 如果不需要，请限制所有容器间的通信。 将需要相互通信的特定容器链接在一起。默认情况下，同一主机上所有容器之间都启用了不受限制的网络流量。 因此，每个容器都有可能读取同一主机上整个容器网络上的所有数据包。 这可能会导致意外和不必要的信息泄露给其他容器。 因此，限制容器间的通信。

#### 加固建议

在守护程序模式下运行docker并传递'--icc = false'作为参数。 例如，
```
/usr/bin/dockerd --icc=false
若使用systemctl管理docker服务则需要编辑` /usr/lib/systemd/system/docker.service

文件中的`ExecStart`参数添加 `--icc=false`选项
然后重启docker服务
systemctl daemon-reload systemctl restart docker  
```
## 5.审核Docker文件和目录
#### 描述
除了审核常规的Linux文件系统和系统调用之外，还审核所有与Docker相关的文件和目录。 Docker守护程序以“ root”特权运行。 其行为取决于某些关键文件和目录。如 /var/lib/docker、/etc/docker、docker.service、 docker.socket、/usr/bin/docker-containerd、/usr/bin/docker-runc等文件和目录
#### 加固建议
在/etc/audit/audit.rules与/etc/audit/rules.d/audit.rules文件中添加以下行：
```
-w /var/lib/docker -k docker
-w /etc/docker -k docker
-w /usr/lib/systemd/system/docker.service -k docker
-w /usr/lib/systemd/system/docker.socket -k docker
-w /usr/bin/docker-containerd -k docker
-w /usr/bin/docker-runc -k docker
然后，重新启动audit程序。 例如

service auditd restart
```
