# Horo Helm Chart

这个Helm Chart用于在Kubernetes集群中部署Horo应用套件。

## 包含组件

- **horo-api**: Web API服务，监听在8080端口（必选）
- **horo-storage-api**: 存储API服务，监听在8080端口（必选）
- **horo-ui**: Web前端应用，监听在80端口（必选）
- **horo-storage-ui**: 存储Web前端应用，监听在80端口（可选）
- **数据库备份**: 定时备份数据库并上传到对象存储（可选，备份文件格式：backup-YYYY-MM-DD-HH_MM_SS.sql.age）

## 安装

```bash
# 添加仓库（示例）
# helm repo add your-repo https://your-helm-repo.example.com

# 安装chart
helm install horo ./horo

# 或者指定自定义values文件
helm install horo ./horo -f custom-values.yaml
```

## 配置说明

### 主要配置选项

```yaml
# 数据库配置
database:
  type: mysql  # 数据库类型（目前仅支持mysql）
  host: mysql  # 数据库主机名
  database: horo  # 数据库名
  user: root  # 数据库用户名
  password: password  # 数据库密码

# 日志级别配置
log:
  level: info  # 可选值: debug, info, warn, error

# 存储服务登录信息
storage:
  user: admin  # 管理员用户名
  password: admin  # 管理员密码

# 星历表文件配置
ephemeris:
  baseUrl: "https://raw.githubusercontent.com/aloistr/swisseph/master/ephe"  # 星历表文件基础URL
  files:  # 需要下载的星历表文件列表
    - "semo_18.se1"
    - "semom48.se1"
    - "sepl_18.se1"
    - "seplm48.se1"
  starFile:  # 恒星文件
    url: "https://raw.githubusercontent.com/wlhyl/horo-api/main/sefstars.txt"

# 数据库备份配置（可选）
backup:
  enabled: false  # 是否启用备份
  schedule: "0 2 * * *"  # cron表达式，默认每天凌晨2点（Asia/Shanghai时区）
  retention: 5  # 保留最近的备份数量
  image:
    repository: "ubuntu"  # 备份使用的镜像
    tag: "22.04"          # 镜像标签
    pullPolicy: IfNotPresent
  age:
    publicKey: ""  # age加密公钥
  s3:
    endpoint: ""  # S3兼容的对象存储端点
    region: ""  # 区域
    bucket: ""  # 存储桶名
    accessKeyId: ""  # 访问密钥ID
    secretAccessKey: ""  # 访问密钥密码
    prefix: "backup/"  # 备份文件路径前缀

# Storage UI 前端服务（可选）
storageUi:
  enabled: false  # 默认禁用，可以设置为true启用

# Ingress 配置
ingress:
  enabled: false  # 默认禁用
  className: "nginx"
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  host: "horo.example.com"  # 主机名
  paths:
    - path: /api
      pathType: Prefix
      targetService: api  # 用于逻辑判断的服务类型标识，对应服务名称为 [release-name]-api
    - path: /api/horo-admin
      pathType: Prefix
      targetService: storageApi  # 用于逻辑判断的服务类型标识，对应服务名称为 [release-name]-storage-api
    - path: /
      pathType: Prefix
      targetService: ui  # 用于逻辑判断的服务类型标识，对应服务名称为 [release-name]-ui
    - path: /horo-admin
      pathType: Prefix
      targetService: storageUi  # 用于逻辑判断的服务类型标识，对应服务名称为 [release-name]-storage-ui，仅在启用storageUi时生效
  tls:
    enabled: false  # 默认禁用HTTPS
    secretName: horo-tls-secret
```
### 使用示例

启用Storage UI和Ingress：

```yaml
# custom-values.yaml
# 启用Storage UI后，其对应的Ingress路径才会生效
storageUi:
  enabled: true

ingress:
  enabled: true
  host: "horo.example.com"  # 设置主机名
```
启用数据库备份：

```yaml
# custom-values.yaml
backup:
  enabled: true
  schedule: "0 2 * * *"  # 每天凌晨2点执行备份（Asia/Shanghai时区）
  retention: 5  # 保留最近5份备份
  age:
    publicKey: "age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"  # age加密公钥
  s3:
    endpoint: "https://your-s3-endpoint.com"
    region: "us-east-1"
    bucket: "your-backup-bucket"
    accessKeyId: "your-access-key-id"
    secretAccessKey: "your-secret-access-key"
    prefix: "horo/backup/"  # 备份文件路径前缀
```

### 自定义备份镜像

为了避免每次备份时都安装必要的工具，您可以创建一个预装了所有工具的自定义镜像。以下是一个示例Dockerfile：

```dockerfile
FROM ubuntu:22.04

# 添加时区文件
RUN apt-get update && apt-get install -y tzdata

# 安装必要的工具
RUN apt-get install -y \
    mariadb-client age wget \
    && rm -rf /var/lib/apt/lists/*

RUN  wget https://dl.min.io/client/mc/release/linux-amd64/mc -O /usr/bin/mc && \
      chmod +x /usr/bin/mc


# 设置工作目录
WORKDIR /backup

CMD ["bash"]
```

构建并推送镜像后，在values.yaml中指定您的自定义镜像：

```yaml
backup:
  image:
    repository: "your-registry/backup-tools"
    tag: "latest"
```

## 卸载

```bash
helm uninstall horo
``` 

# 许可证
项目使用GPL-3.0 许可证 ([LICENSE](LICENSE))。

