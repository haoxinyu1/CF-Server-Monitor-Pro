# CF-Server-Monitor-Pro Serverless 探针增强版

基于 Cloudflare Workers 和 D1 数据库构建的轻量级服务器探针大盘。
项目不需要额外部署服务端 VPS，Cloudflare Worker 负责页面、接口和数据写入，D1 负责保存节点状态与设置。

## 核心特性

### 监控展示

- 多节点服务器大盘，展示服务器总数、在线/离线数、总流量和实时网速。
- 支持 CPU、内存、磁盘、负载、进程数、TCP/UDP 连接数、系统版本、CPU 型号等指标。
- 支持 IPv4/IPv6 连通性检测。
- 支持国内三网和字节节点延迟检测。
- 支持按分组展示节点。
- 支持价格、到期时间、带宽、流量配额等自定义信息。
- 支持月度流量统计，每月 1 号自动重置。

### 安全增强

- 前台默认不公开，可通过环境变量 `IS_PUBLIC` 控制。
- 后台路径可通过环境变量 `ADMIN_PATH` 自定义，不再固定暴露 `/admin`。
- 前台页面不再显示后台入口按钮，避免公开后台路径。
- 每台服务器使用独立上报密钥 `report_secret`，不再共用 `API_SECRET`。
- `API_SECRET` 只用于后台 Basic Auth 登录。
- `/api/server` 不再返回完整数据库字段，只返回详情页必要字段。
- 动态写入 HTML 的字段已做转义处理，降低 XSS 风险。

### 安装与兼容性

- 后台自动生成每台服务器的安装命令。
- 后台自动生成卸载命令。
- Linux Agent 默认每 30 秒上报一次，降低 Worker 和 D1 免费额度消耗。
- 支持 systemd 系统。
- 支持 Alpine/OpenRC 系统。
- Alpine 缺少依赖时会明确提示安装依赖，不再假提示安装成功。

## Cloudflare 免费额度参考

### Workers Free

- 请求数：100,000 次/天。
- CPU 时间：10 ms/请求。
- 内存：128 MB。
- 环境变量：64 个/Worker。

### D1 Free

- 数据库数量：10 个/账号。
- 单库大小：500 MB。
- 账号总存储：5 GB。
- 读取：5,000,000 行/天。
- 写入：100,000 行/天。

当前 Agent 上报间隔为 30 秒。

估算消耗：

| 节点数量 | Worker 请求/天 | D1 写入/天 |
| --- | ---: | ---: |
| 1 台 | 2,880 | 2,880 |
| 5 台 | 14,400 | 14,400 |
| 10 台 | 28,800 | 28,800 |

10 台以内使用 30 秒上报间隔，通常低于 Cloudflare 免费额度。

## 部署指南

### 1. 创建 D1 数据库

1. 登录 Cloudflare 控制台。
2. 进入 `Workers & Pages`。
3. 打开 `D1 SQL Database`。
4. 创建一个数据库，例如 `probe-db`。

脚本内置自动建表和自动迁移逻辑，只需要先创建并绑定 D1 数据库即可。

### 2. 创建 Worker

1. 在 `Workers & Pages` 中创建新的 Worker。
2. 进入该 Worker 的设置页面。
3. 打开 `Variables and Secrets`。
4. 绑定 D1 数据库。

D1 绑定配置：

| 类型 | 名称 | 值 |
| --- | --- | --- |
| D1 数据库绑定 | `DB` | 选择你的 D1 数据库 |

### 3. 设置环境变量

必填变量：

| 变量名 | 说明 |
| --- | --- |
| `API_SECRET` | 后台 Basic Auth 密码 |

建议变量：

| 变量名 | 示例 | 说明 |
| --- | --- | --- |
| `IS_PUBLIC` | `false` | 是否公开前台大盘。建议设为 `false` |
| `ADMIN_PATH` | `/my-secure-admin` | 后台路径。也支持不带 `/`，例如 `my-secure-admin` |

推荐配置：

```text
API_SECRET=一串强随机密码
IS_PUBLIC=false
ADMIN_PATH=/my-secure-admin
```

路径规则：

| `ADMIN_PATH` | 后台页面 | 后台 API | 安装脚本 |
| --- | --- | --- | --- |
| `/my-secure-admin` | `/my-secure-admin` | `/my-secure-admin/api` | `/my-secure-admin/install.sh` |
| `my-secure-admin` | `/my-secure-admin` | `/my-secure-admin/api` | `/my-secure-admin/install.sh` |

`ADMIN_PATH` 不建议继续使用默认值 `/admin`。

### 4. 部署代码

1. 打开 Worker 的代码编辑页面。
2. 将 `10台VPS以内选这个已更新三网延迟.js` 的内容复制进去。
3. 点击部署。
4. 访问你设置的后台路径。

## 使用说明

### 访问后台

如果设置：

```text
ADMIN_PATH=/my-secure-admin
```

后台地址就是：

```text
https://你的Worker域名/my-secure-admin
```

登录信息：

| 字段 | 值 |
| --- | --- |
| 用户名 | `admin` |
| 密码 | `API_SECRET` 的值 |

### 添加节点

1. 登录后台。
2. 输入节点名称。
3. 点击添加新服务器。
4. 后台会自动为该节点生成独立上报密钥。
5. 复制该节点的安装命令。
6. 登录 VPS 后执行安装命令。

### 安装探针

后台会为每台服务器生成专属安装命令，格式类似：

```bash
curl -sL https://你的Worker域名/你的后台路径/install.sh | bash -s 节点ID 节点独立上报密钥
```

不要手动把 `API_SECRET` 当作节点上报密钥使用。

### Alpine 安装依赖

Alpine 默认没有 systemd，脚本会自动使用 OpenRC。

建议 Alpine 安装前先执行：

```bash
apk add --no-cache bash curl procps iproute2 coreutils
```

然后再执行后台复制的安装命令。

### 卸载探针

后台每个节点都会显示对应的卸载命令，直接复制到 VPS 执行即可。

卸载命令同时兼容：

- systemd。
- Alpine/OpenRC。

它会清理：

- `/etc/systemd/system/cf-probe.service`
- `/etc/init.d/cf-probe`
- `/usr/local/bin/cf-probe.sh`
- `/run/cf-probe.pid`
- `/var/log/cf-probe.log`
- `/var/log/cf-probe.err`

如果你只需要 systemd 手动卸载，也可以执行：

```bash
systemctl stop cf-probe.service
systemctl disable cf-probe.service
rm -f /etc/systemd/system/cf-probe.service
rm -f /usr/local/bin/cf-probe.sh
systemctl daemon-reload
```

## 后台设置说明

### 公开访问

前台是否公开主要由环境变量 `IS_PUBLIC` 控制。

建议：

```text
IS_PUBLIC=false
```

如果设置为 `false`，访问前台也需要输入 Basic Auth。

### 前台展示开关

后台可以控制这些信息是否在前台显示：

- 价格。
- 到期时间。
- 带宽。
- 流量配额。

### 自定义背景图

支持两种方式：

- 上传本地图片，自动转为 Base64。
- 填写 `http` 或 `https` 图片 URL。

示例随机背景接口：

```text
https://imgapi.cn/api.php?fl=dongman&=4k
```

### Telegram 离线通知

配置步骤：

1. 在 Telegram 找 `@BotFather` 创建机器人并获取 Bot Token。
2. 在 Telegram 找 `@userinfobot` 获取 Chat ID。
3. 登录探针后台。
4. 在 Telegram 离线告警设置中填写 Bot Token 和 Chat ID。
5. 将开启离线通知设置为启用。
6. 保存全局设置。

离线判断：

- 节点超过 120 秒未上报，会触发离线通知。
- 节点恢复后，会触发恢复通知。
- 同一次离线状态只会通知一次，避免重复刷屏。

## 常见问题

### 前端为什么不再自动刷新

旧版本首页有：

```html
<meta http-equiv="refresh" content="5">
```

会导致首页每 5 秒整页刷新。

当前版本已删除整页刷新。

详情页仍会每 3 秒局部请求 `/api/server` 更新图表，不会整页刷新。

### 不打开页面还会消耗 Cloudflare 吗

会。

只要 VPS 上的 Agent 还在运行，它就会每 30 秒请求一次 Worker 的 `/update`，并写入 D1。

页面访问只会增加额外读请求，不影响 Agent 自身上报。

### 为什么设置了 ADMIN_PATH 但后台路径没变

确认已经重新部署 Worker。

`ADMIN_PATH` 支持两种写法：

```text
ADMIN_PATH=myadmin
ADMIN_PATH=/myadmin
```

两者都会生效为：

```text
/myadmin
```

如果线上仍能访问旧的 `/admin`，通常是代码未重新部署，或 Worker 仍在运行旧版本。

### Alpine 报 systemctl 不存在怎么办

当前版本已支持 OpenRC。

如果是旧版本报错：

```text
systemctl: command not found
/etc/systemd/system/cf-probe.service: No such file or directory
```

请重新部署最新 Worker，然后重新复制后台安装命令。

Alpine 建议先安装依赖：

```bash
apk add --no-cache bash curl procps iproute2 coreutils
```

## 安全建议

- `API_SECRET` 使用强随机密码，不要使用简单密码。
- `ADMIN_PATH` 不要使用默认 `/admin`。
- `IS_PUBLIC` 建议设为 `false`。
- 不要公开后台路径。
- 不要把安装命令发给不可信的人。
- 如果怀疑泄露，删除节点后重新添加，生成新的节点上报密钥。
- 如果 Telegram Bot Token 泄露，请在 BotFather 重新生成。

## License

MIT License
