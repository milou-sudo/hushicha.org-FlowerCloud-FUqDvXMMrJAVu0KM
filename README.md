
Nacos 是 Dynamic Naming and Configuration Service 的首字母简称，一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。


Nacos 是构建以**服务**为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。更多的功能特性介绍请查看 [Nacos 概览](https://github.com)。


在本文中，我将为您提供一份全面的实战指南，详细地指导您如何在 Kubernetes 集群中部署以集群模式运行的 Nacos 服务。


### 实战服务器配置(架构1:1复刻小规模生产环境，配置略有不同)




| 主机名 | IP | CPU | 内存 | 系统盘 | 数据盘 | 用途 |
| --- | --- | --- | --- | --- | --- | --- |
| ksp\-registry | 192\.168\.9\.90 | 4 | 8 | 40 | 200 | Harbor 镜像仓库 |
| ksp\-control\-1 | 192\.168\.9\.91 | 4 | 8 | 40 | 100 | KubeSphere/k8s\-control\-plane |
| ksp\-control\-2 | 192\.168\.9\.92 | 4 | 8 | 40 | 100 | KubeSphere/k8s\-control\-plane |
| ksp\-control\-3 | 192\.168\.9\.93 | 4 | 8 | 40 | 100 | KubeSphere/k8s\-control\-plane |
| ksp\-worker\-1 | 192\.168\.9\.94 | 8 | 16 | 40 | 100 | k8s\-worker/CI |
| ksp\-worker\-2 | 192\.168\.9\.95 | 8 | 16 | 40 | 100 | k8s\-worker |
| ksp\-worker\-3 | 192\.168\.9\.96 | 8 | 16 | 40 | 100 | k8s\-worker |
| ksp\-storage\-1 | 192\.168\.9\.97 | 4 | 8 | 40 | 400\+ | ElasticSearch/Longhorn/Ceph/NFS |
| ksp\-storage\-2 | 192\.168\.9\.98 | 4 | 8 | 40 | 300\+ | ElasticSearch/Longhorn/Ceph |
| ksp\-storage\-3 | 192\.168\.9\.99 | 4 | 8 | 40 | 300\+ | ElasticSearch/Longhorn/Ceph |
| ksp\-gpu\-worker\-1 | 192\.168\.9\.101 | 4 | 16 | 40 | 100 | k8s\-worker(GPU NVIDIA Tesla M40 24G) |
| ksp\-gpu\-worker\-2 | 192\.168\.9\.102 | 4 | 16 | 40 | 100 | k8s\-worker(GPU NVIDIA Tesla P100 16G) |
| ksp\-gateway\-1 | 192\.168\.9\.103 | 2 | 4 | 40 |  | 自建应用服务代理网关/VIP：192\.168\.9\.100 |
| ksp\-gateway\-2 | 192\.168\.9\.104 | 2 | 4 | 40 |  | 自建应用服务代理网关/VIP：192\.168\.9\.100 |
| ksp\-mid | 192\.168\.9\.105 | 4 | 8 | 40 | 100 | 部署在 k8s 集群之外的服务节点（Gitlab 等） |
| 合计 | 15 | 68 | 152 | 600 | 2100\+ |  |


### 实战环境涉及软件版本信息


* 操作系统：**openEuler 22\.03 LTS SP3 x86\_64**
* KubeSphere：**v3\.4\.1**
* Kubernetes：**v1\.28\.8**
* KubeKey: **v3\.1\.1**
* MySQL：**v5\.7\.44**
* Nacos： **v2\.4\.2\.1**


## 1\. 部署方案规划


### 1\.1 部署架构图


![](https://opsxlab-1258881081.cos.ap-beijing.myqcloud.com//ksp-nacos-cluster.png)


### 1\.2 准备 Nacos 部署资源


* 创建部署资源根目录



```
mkdir /srv/nacos
cd /srv/nacos

```

* 获取**官方资源配置清单**



```
# wget 方式（推荐）
wget https://codeload.github.com/nacos-group/nacos-k8s/zip/refs/heads/master -O nacos-k8s-master.zip

# git 方式
git clone https://github.com/nacos-group/nacos-k8s.git

```

* 获取**初始化数据库文件**



```
wget https://raw.githubusercontent.com/alibaba/nacos/refs/heads/master/distribution/conf/mysql-schema.sql

```

### 1\.3 准备 MySQL


Nacos 需要使用 MySQL，本文使用更加贴近生产的 MySQL 主从复制方案部署 MySQL，具体可以参考 [一文搞定，在 Kubernetes 集群上部署主从复制 MySQL](https://github.com) 。



> **提示：** 也可以使用官方提供的资源清单 `deploy/mysql/mysql-nfs.yaml` ，部署单机版的 MySQL 服务。


**Step 1：导入 MySQL 初始化数据**


1. 进入 MySQL 主节点容器内部。



```
$ kubectl exec -it mysql-source-0 -- mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.44-log MySQL Community Server (GPL)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>

```

2. 创建数据库及 Nacos 用户。



```
-- 创建数据库
mysql> CREATE DATABASE IF NOT EXISTS `nacos` DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 创建用户
mysql> CREATE USER 'nacos'@'%' IDENTIFIED BY 'ChangeMe';

-- 赋予权限
mysql> GRANT ALL PRIVILEGES ON `nacos`.* TO 'nacos'@'%';

-- 刷新权限
mysql> FLUSH PRIVILEGES;

```

3. 导入数据（无需登录容器内部）。



```
# 进入数据库初始化 sql 文件目录
$ cd /srv/nacos/

# 导入数据
kubectl exec -i mysql-source-0 -- mysql -S /var/lib/mysql/mysql.sock -u nacos -pChangeMe nacos < mysql-schema.sql

```

**Step 2：查看导入的数据**


1. 登陆 MySQL 主节点容器内部。



```
$ kubectl exec -it mysql-source-0 -- mysql -u nacos -p
Enter password:

```

2. 查看数据库。



```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nacos              |
+--------------------+
2 rows in set (0.01 sec)

```

3. 查看表。



```
# 切换数据库
mysql> use nacos;
Database changed

# 查看表
mysql> show tables;
+----------------------+
| Tables_in_nacos      |
+----------------------+
| config_info          |
| config_info_aggr     |
| config_info_beta     |
| config_info_tag      |
| config_tags_relation |
| group_capacity       |
| his_config_info      |
| permissions          |
| roles                |
| tenant_capacity      |
| tenant_info          |
| users                |
+----------------------+
12 rows in set (0.00 sec)

```

从结果中可以看到执行 `mysql-schema.sql` 后，自动创建了 Nacos 服务的相关表和数据。


### 1\.4 准备持久化存储


本实战环境使用 NFS 作为 K8s 集群的持久化存储，如果是新集群可以参考[《探索 Kubernetes 持久化存储之 NFS 终极实战指南》](https://github.com) 部署 NFS 存储。


**提示：** 也可以使用官方提供的 `deploy/nfs/` 目录下的资源清单，部署单机版的 NFS 服务。


## 2\. 集群模式 Nacos 部署


### 2\.1 修改配置文件


1. 解压部署代码。



```
$ cd /srv/nacos
$ unzip nacos-k8s-master.zip
$ cd nacos-k8s-master/deploy/nacos

```

2. 编辑 **nacos\-pvc\-nfs.yaml**，修改数据库配置。



```
data:
  mysql.host: "mysql-source-headless.default.svc.cluster.local"		# 数据库地址，本文使用 MySQL 服务在 k8s 集群内的 DNS 域名
  mysql.db.name: "nacos"
  mysql.port: "3306"
  mysql.user: "nacos"
  mysql.password: "ChangeMe"

```

3. 修改 StoreClass 名称（**可选，自建 NFS 存储时使用**）。


默认的配置文件使用的 StoreClass 名称为 **managed\-nfs\-storage**，使用下面的命令修改为实际的值。



```
$ sed -i 's/managed-nfs-storage/nfs-sc/g' nacos-pvc-nfs.yaml

```

4. 删除 serviceAccountName（**可选，自建 NFS 存储时使用**）。



```
sed -i '/serviceAccountName/d' nacos-pvc-nfs.yaml

```

5. 修改镜像地址（**可选，镜像下载受限或是离线部署时可用**）。



```
sed -i 's#nacos/nacos-peer-finder-plugin:1.1#registry.opsxlab.cn:8443/nacos/nacos-peer-finder-plugin:1.1#g' nacos-pvc-nfs.yaml
sed -i 's#nacos/nacos-server:latest#registry.opsxlab.cn:8443/nacos/nacos-server:v2.4.2.1#g' nacos-pvc-nfs.yaml

```

6. 开启鉴权配置（**建议**）。


Nacos 默认配置没有开启鉴权，**生产环境建议开启**。在 `containers.env` 部分增加下面的内容：



```
- name: NACOS_AUTH_ENABLE
  value: "true"
- name: NACOS_AUTH_TOKEN
  value: "SecretKeyYzJlMTMxOTU5ZTljZTkxZGQ2MDcwZGIxMzU1YTFkMjg="
- name: NACOS_AUTH_IDENTITY_KEY
  value: "serverIdentity"
- name: NACOS_AUTH_IDENTITY_VALUE
  value: "ChangeMe"

```

**注意：** 自定义 `NACOS_AUTH_TOKEN` 密钥时，推荐将配置项设置为**Base64 编码**的字符串，且**原始密钥长度不得低于32字符**。


可以执行下面的命令生成TOKEN 密钥：



```
echo -n $(openssl rand -hex 16) | base64 -w0

```

### 2\.2 部署 Nacos 集群


1. 执行下面的命令，创建 Nacos。



```
$ kubectl create -f nacos-pvc-nfs.yaml

```

**正确执行后，输出结果如下 :**



```
$ kubectl create -f nacos-pvc-nfs.yaml
service/nacos-headless created
configmap/nacos-cm created
statefulset.apps/nacos created

```

2. 验证 Nacos 节点状态。



```
$ kubectl get pod -l app=nacos -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
nacos-0   1/1     Running   0          25s   10.233.96.233   ksp-worker-3              
nacos-1   1/1     Running   0          25s   10.233.94.125   ksp-worker-1              
nacos-2   1/1     Running   0          25s   10.233.68.221   ksp-worker-2              

```

### 2\.3 配置 K8s 集群外部访问


我们采用 NodePort 方式在 Kubernetes 集群中对外发布 Nacos 服务，以便管理员能够访问图形化控制台，同时也为集群外的应用提供服务，指定的端口为 **31848**。


使用 `vi` 编辑器，新建 NodePort 服务资源清单文件 `nacos-external.yaml`，并输入以下内容：



```
kind: Service
apiVersion: v1
metadata:
  name: nacos-external
  labels:
    app: nacos-external
spec:
  ports:
    - protocol: TCP
      port: 8848
      targetPort: 8848
      nodePort: 31848
  selector:
    app: nacos
  type: NodePort

```

### 2\.4 设置管理员密码


自 **2\.4\.0** 版本开始，Nacos构建时不再提供管理员用户`nacos`的默认密码，需要在首次开启鉴权后，通过 API 或 Nacos 控制台进行管理员用户`nacos`的密码初始化。


本文选择 Nacos 控制台的方式初始化密码，当 Nacos 集群开启鉴权后，访问 Nacos 控制台时，会校验管理员用户`nacos`的密码是否已经初始化，若发现未初始化密码时，则会跳转至初始化密码的页面进行初始化。


![](https://opsxlab-1258881081.cos.ap-beijing.myqcloud.com//nacos-2.4.2-register.png)


在该页面密码文本框内输入自定义密码，然后点击提交即可。



> **注意：** 若密码文本框内未输入自定义密码或输入空白密码，Nacos 将会生成随机密码，请保存好生成的随机密码。


初始化成功后会弹窗提示初始化成功，并明文展示指定的密码或随机生成的密码，请保存好此密码。


![](https://opsxlab-1258881081.cos.ap-beijing.myqcloud.com//nacos-2.4.2-register-tishi.png)


点击「确定」后，会跳转到登录页面，并弹出权限认证失败的提示框。


![](https://opsxlab-1258881081.cos.ap-beijing.myqcloud.com//nacos-2.4.2-login-error.png)


点击「确定」后，输入 nacos 用户名和对应的密码。


![](https://opsxlab-1258881081.cos.ap-beijing.myqcloud.com//nacos-2.4.2-login.png)


登录成功后，进入「配置管理」页面。


![](https://opsxlab-1258881081.cos.ap-beijing.myqcloud.com//nacos-2.4.2-config-management.png)


## 3\. 验证测试 Nacos 服务是否正确配置


使用 curl 命令，在 K8s 集群外的机器上调用 Nacos API 接口，通过 Nacos 对外服务对应的 NodePort 端口，验证测试 Nacos 服务是否正常。


### 3\.1 获取 Token


首先，使用用户名和密码登陆 nacos，若用户名和密码正确，会返回 Token 信息。



```
curl -X POST 'http://192.168.9.91:31848/nacos/v1/auth/login' -d 'username=nacos&password=ChangeMe'

```

**正确执行后，返回的结果如下：**



```
{"accessToken":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTcyNzcwOTk0Mn0.Ki2kgZyh_dj_Zfb9HKPCkKr1cgWfi3szQS4hlZPIwkI","tokenTtl":18000,"globalAdmin":true,"username":"nacos"}

```

### 3\.2 服务注册



```
curl -X POST 'http://192.168.9.91:31848/nacos/v1/ns/instance?accessToken=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTcyNzcwOTk0Mn0.Ki2kgZyh_dj_Zfb9HKPCkKr1cgWfi3szQS4hlZPIwkI&serviceName=nacos.naming.serviceName&ip=192.168.9.81&port=8080'

```

### 3\.3 服务发现



```
curl -X GET 'http://192.168.9.91:31848/nacos/v1/ns/instance/list?accessToken=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTcyNzcwOTk0Mn0.Ki2kgZyh_dj_Zfb9HKPCkKr1cgWfi3szQS4hlZPIwkI&serviceName=nacos.naming.serviceName'

```

**正确执行后，返回的结果如下：**



```
{"name":"DEFAULT_GROUP@@nacos.naming.serviceName","groupName":"DEFAULT_GROUP","clusters":"","cacheMillis":10000,"hosts":[],"lastRefTime":1727692102280,"checksum":"","allIPs":false,"reachProtectionThreshold":false,"valid":true}[

```

### 3\.4 发布配置



```
curl -X POST "http://192.168.9.91:31848/nacos/v1/cs/configs?accessToken=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTcyNzcwOTk0Mn0.Ki2kgZyh_dj_Zfb9HKPCkKr1cgWfi3szQS4hlZPIwkI&dataId=nacos.cfg.dataId&group=test&content=helloWorld"

```

### 3\.5 获取配置



```
curl -X GET "http://192.168.9.91:31848/nacos/v1/cs/configs?accessToken=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTcyNzcwOTk0Mn0.Ki2kgZyh_dj_Zfb9HKPCkKr1cgWfi3szQS4hlZPIwkI&dataId=nacos.cfg.dataId&group=test"

```

**正确执行后，返回的结果如下：**



```
helloWorld

```

### 3\.6 Nacos 控制台查看


* 配置管理列表


![](https://opsxlab-1258881081.cos.ap-beijing.myqcloud.com//nacos-2.4.2-configuration-test.png)


* 配置详情


![](https://opsxlab-1258881081.cos.ap-beijing.myqcloud.com//nacos-2.4.2-configdetail-test.png)


至此，我们完成了在 KubeSphere 管理的 Kubernetes 集群上手动部署 Nacos 集群的全过程，后续的配置管理请根据实际应用的需求进行配置。


**免责声明：**


* 笔者水平有限，尽管经过多次验证和检查，尽力确保内容的准确性，**但仍可能存在疏漏之处**。敬请业界专家大佬不吝指教。
* 本文所述内容仅通过实战环境验证测试，读者可学习、借鉴，但**严禁直接用于生产环境**。**由此引发的任何问题，作者概不负责！**


本文内容首发：运维有术。


## 关于 KubeSphere


KubeSphere （[https://kubesphere.io](https://github.com):[veee加速器](https://liuyunzhuge.com)）是在 Kubernetes 之上构建的开源容器平台，提供全栈的 IT 自动化运维的能力，简化企业的 DevOps 工作流。


KubeSphere 已被 Aqara 智能家居、本来生活、东方通信、微宏科技、东软、华云、新浪、三一重工、华夏银行、四川航空、国药集团、微众银行、紫金保险、去哪儿网、中通、中国人民银行、中国银行、中国人保寿险、中国太平保险、中国移动、中国联通、中国电信、天翼云、中移金科、Radore、ZaloPay 等海内外数万家企业采用。KubeSphere 提供了开发者友好的向导式操作界面和丰富的企业级功能，包括 Kubernetes 多云与多集群管理、DevOps (CI/CD)、应用生命周期管理、边缘计算、微服务治理 (Service Mesh)、多租户管理、可观测性、存储与网络管理、GPU support 等功能，帮助企业快速构建一个强大和功能丰富的容器云平台。


✨ GitHub：[https://github.com/kubesphere](https://github.com)
💻 官网（中国站）：[https://kubesphere.io/zh](https://github.com)
🙋 论坛：[https://ask.kubesphere.io/forum/](https://github.com)
👨‍💻‍ 微信群：请搜索添加群助手微信号 kubesphere


