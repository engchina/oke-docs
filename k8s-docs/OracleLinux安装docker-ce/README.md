[返回OKE中文文档集](../../README.md)

[返回kubernetes中文文档集](../README.md)

# OracleLinux安装docker-ce

本文介绍如何在OracleLinux系统上安装docker-ce。

## OracleLinux 7.x

追加docker-ce的yum源，

```
sudo yum-config-manager \
   --add-repo \
   https://download.docker.com/linux/centos/docker-ce.repo
   
sudo bash -c 'cat<< "EOF" >> /etc/yum.repos.d/docker-ce.repo

[centos-extras]
name=Centos extras - $basearch
baseurl=http://mirror.centos.org/centos/7/extras/x86_64
enabled=1
gpgcheck=1
gpgkey=http://centos.org/keys/RPM-GPG-KEY-CentOS-7
EOF'
```

安装最新版本docker-ce，

```
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

(Optional)安装指定版本docker-ce，

```
sudo yum list docker-ce --showduplicates | sort -r
sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io docker-compose-plugin
```

启动docker并设置开始自动启动docker，

```
sudo systemctl start docker
sudo systemctl enable docker
```

将一般用户加入到docker组中，赋予一般用户有使用docker的权限，

示例，赋予`opc`用户有使用docker的权限，

```
sudo usermod -a -G docker opc
```

查看docker启动状况，

```
sudo systemctl status docker
```

输入结果示例，

```
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-10-23 01:08:24 GMT; 9s ago
     Docs: https://docs.docker.com
 Main PID: 13465 (dockerd)
   CGroup: /system.slice/docker.service
           └─13465 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Oct 23 01:08:23 oraclelinux7 dockerd[13465]: time="2022-10-23T01:08:23.934224194Z" level=warning msg="Your kernel does not support cgroup blkio weight_device"
Oct 23 01:08:23 oraclelinux7 dockerd[13465]: time="2022-10-23T01:08:23.934398083Z" level=info msg="Loading containers: start."
Oct 23 01:08:24 oraclelinux7 dockerd[13465]: time="2022-10-23T01:08:24.481145644Z" level=info msg="Default bridge (docker0) is assigned with an IP address 172...P address"
Oct 23 01:08:24 oraclelinux7 dockerd[13465]: time="2022-10-23T01:08:24.571636851Z" level=info msg="Firewalld: interface docker0 already part of docker zone, returning"
Oct 23 01:08:24 oraclelinux7 dockerd[13465]: time="2022-10-23T01:08:24.641489018Z" level=info msg="Loading containers: done."
Oct 23 01:08:24 oraclelinux7 dockerd[13465]: time="2022-10-23T01:08:24.652123740Z" level=warning msg="Not using native diff for overlay2, this may cause degra...r=overlay2
Oct 23 01:08:24 oraclelinux7 dockerd[13465]: time="2022-10-23T01:08:24.652299822Z" level=info msg="Docker daemon" commit=03df974 graphdriver(s)=overlay2 version=20.10.20
Oct 23 01:08:24 oraclelinux7 dockerd[13465]: time="2022-10-23T01:08:24.652440979Z" level=info msg="Daemon has completed initialization"
Oct 23 01:08:24 oraclelinux7 systemd[1]: Started Docker Application Container Engine.
Oct 23 01:08:24 oraclelinux7 dockerd[13465]: time="2022-10-23T01:08:24.691536231Z" level=info msg="API listen on /var/run/docker.sock"
Hint: Some lines were ellipsized, use -l to show in full.
```

## OracleLinux 8.x

追加docker-ce的yum源，

```
sudo dnf config-manager \
   --add-repo \
   https://download.docker.com/linux/centos/docker-ce.repo
```

安装最新版本docker-ce，

```
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

(Optional)安装指定版本docker-ce，

```
sudo dnf list docker-ce --showduplicates | sort -r
sudo dnf install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io docker-compose-plugin
```

启动docker并设置开始自动启动docker，

```
sudo systemctl start docker
sudo systemctl enable docker
```

将一般用户加入到docker组中，赋予一般用户有使用docker的权限，

示例，赋予`opc`用户有使用docker的权限，

```
sudo usermod -a -G docker opc
```

查看docker启动状况，

```
sudo systemctl status docker
```

输入结果示例，

```
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-10-23 01:24:29 GMT; 7s ago
     Docs: https://docs.docker.com
 Main PID: 47499 (dockerd)
    Tasks: 9
   Memory: 29.2M
   CGroup: /system.slice/docker.service
           └─47499 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Oct 23 01:24:28 oraclelinux8 dockerd[47499]: time="2022-10-23T01:24:28.450276400Z" level=warning msg="Your kernel does not support cgroup blkio weight_device"
Oct 23 01:24:28 oraclelinux8 dockerd[47499]: time="2022-10-23T01:24:28.450406546Z" level=info msg="Loading containers: start."
Oct 23 01:24:28 oraclelinux8 dockerd[47499]: time="2022-10-23T01:24:28.967665308Z" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. >
Oct 23 01:24:29 oraclelinux8 dockerd[47499]: time="2022-10-23T01:24:29.091872543Z" level=info msg="Firewalld: interface docker0 already part of docker zone, returning"
Oct 23 01:24:29 oraclelinux8 dockerd[47499]: time="2022-10-23T01:24:29.217679203Z" level=info msg="Loading containers: done."
Oct 23 01:24:29 oraclelinux8 dockerd[47499]: time="2022-10-23T01:24:29.245125422Z" level=warning msg="Not using native diff for overlay2, this may cause degraded performa>
Oct 23 01:24:29 oraclelinux8 dockerd[47499]: time="2022-10-23T01:24:29.245369012Z" level=info msg="Docker daemon" commit=03df974 graphdriver(s)=overlay2 version=20.10.20
Oct 23 01:24:29 oraclelinux8 dockerd[47499]: time="2022-10-23T01:24:29.245585402Z" level=info msg="Daemon has completed initialization"
Oct 23 01:24:29 oraclelinux8 systemd[1]: Started Docker Application Container Engine.
Oct 23 01:24:29 oraclelinux8 dockerd[47499]: time="2022-10-23T01:24:29.326572392Z" level=info msg="API listen on /var/run/docker.sock"
```



完结！



[返回kubernetes中文文档集](../README.md)

[返回OKE中文文档集](../../README.md)



