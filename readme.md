# KubeSpringBoot 部署指南

## 项目概述

KubeSpringBoot 是一个基于 Kubernetes 的 Spring Boot 应用部署框架，支持一键部署应用服务及相关依赖组件（MySQL、Redis、MinIO 等），提供完整的容器化应用生命周期管理能力。


## 核心组件

### 应用服务
- **Spring Boot 应用**：自动识别并部署指定目录下的 JAR 包，支持单节点/集群模式自适应部署

### 依赖服务
- **MySQL**：关系型数据库，支持主从架构部署，自动执行初始化 SQL 脚本
- **Redis**：缓存服务，支持单节点和集群模式，通过 ConfigMap 管理配置
- **MinIO**：对象存储服务，用于文件存储管理，支持单节点和分布式部署


## 前置要求

- Kubernetes 集群，最低版本 `v1.29.0+`
- Helm，最低版本 `v3.0.0+`
- Ansible，用于执行自动化部署脚本
- Docker，用于构建应用镜像
- JAR 包：需将待部署的 Spring Boot 应用 JAR 包放置在项目根目录


## 部署步骤

### 1. 获取项目代码
```bash
git clone <项目仓库地址>
cd KubeSpringBoot
```

### 2. 配置部署选项
修改 `roles/<组件名>/vars/main.yml` 文件配置组件参数，关键配置项包括：
- 命名空间（默认：`spring-boot`）
- 端口映射（如应用默认 NodePort：30880）
- 资源限制（CPU/内存）
- 存储路径与大小

### 3. 执行部署
默认部署 Spring Boot 应用，如需部署其他组件，修改 `install.yml` 启用对应角色：
```yaml
# install.yml 示例（取消注释启用组件）
roles:
  - mysql
  - redis
  - minio
  - app
```

执行部署命令：
```bash
ansible-playbook -i inventory install.yml
```
> 注：`inventory` 为包含 Kubernetes 集群节点信息的 Ansible inventory 文件


## 组件部署细节

### 应用部署（app 角色）
1. 自动识别项目根目录下的 JAR 包
2. 构建 Docker 镜像（基于 openjdk:8-jdk-alpine）
3. 根据集群节点数量自动选择部署模式：
   - 单节点：Deployment + PV/PVC 存储
   - 多节点：StatefulSet + 无头服务

### 数据库部署（mysql 角色）
- 支持主从架构，初始化 SQL 脚本路径：`roles/mysql/files/init.sql`
- 数据持久化存储到节点目录，自动清理历史资源（Deployment、StatefulSet 等）
- 部署模式自适应：根据可用节点数自动选择单点（仅主库）或多点（主从复制）

### 缓存部署（redis 角色）
- 支持单节点/集群模式，通过 ConfigMap 配置密码、内存限制等
- 集群模式自动初始化主从关系，单节点模式使用 Deployment + PV/PVC
- 外部访问通过 NodePort 暴露，默认端口：30379

### 对象存储部署（minio 角色）
- 支持单节点（standalone）和分布式（distributed）模式
- 控制台与 API 分别通过 NodePort 暴露，数据持久化到节点目录
- 自动清理历史资源，确保部署环境干净


## 访问方式

| 组件           | 访问地址                          | 说明                                  |
|----------------|-----------------------------------|---------------------------------------|
| Spring Boot 应用 | http://<节点IP>:30880             | 应用默认外部端口                      |
| Redis          | redis-cli -h <节点IP> -p 30379    | 需使用配置的密码（默认：`!qaz2Wsx`）  |
| MySQL          | mysql -h <节点IP> -P 30306        | 初始密码需从 Secret 中获取            |
| MinIO API      | http://<节点IP>:<minio_nodeport>  | 默认端口可在 vars 配置中修改          |
| MinIO 控制台   | http://<节点IP>:<console_nodeport>| 初始账号密码在 Secret 中配置          |


## 许可证

项目中各组件遵循各自的开源许可证，核心框架遵循 Apache License 2.0，详情参见各组件的 LICENSE 文件。