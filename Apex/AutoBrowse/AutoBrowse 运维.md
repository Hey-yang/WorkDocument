---
title: AutoBrowse
tags:
  - Apex
  - 部署
  - 运维
---

# AutoBrowse 运维手册

> [!info] 快速索引
> | 场景 | 章节 |
> | --- | --- |
> | 本地起后端 / 前端 | [[#1. 本地开发环境]] |
> | 建库、改表结构 | [[#2. 数据库]] |
> | 打浏览器插件包、发版 | [[#3. 浏览器插件打包]] |
> | 前端免登录调试 | [[#4. 前端调试登录态注入]] |
> | 上线部署（含新服务器） | [[#5. 服务器部署]] |
> | doc-recognition 服务 | [[#6. doc-recognition 项目]] |

> [!warning] 本文含明文密钥
> 文档中保留了 AMO API Secret、数据库密码、调试用 JWT。请勿外传或提交到公开仓库；换环境时记得同步替换。

---

## 1. 本地开发环境

### 1.1 后端

```bash
cd autobrowse-backend
python -m venv venv
venv\Scripts\activate          # Windows
pip install -r requirements.txt
python main.py
```

### 1.2 前端

```bash
cd autobrowse-frontend
npm install
npm run dev
```

### 1.3 Docker 部署

```bash
docker-compose up -d
```

---

## 2. 数据库

### 2.1 Alembic 迁移

```bash
cd autobrowse-backend

# 生成迁移脚本
alembic revision --autogenerate -m "Initial migration"

# 应用迁移
alembic upgrade head

# 回滚一个版本
alembic downgrade -1
```

### 2.2 初始化 MySQL（建库 + 建用户）

```sql
-- 创建数据库
CREATE DATABASE IF NOT EXISTS autobrowse
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

-- 使用数据库
USE autobrowse;

-- 创建用户并授权
CREATE USER IF NOT EXISTS 'autobrowse'@'%' IDENTIFIED BY 'autobrowsePwd_2025';
GRANT ALL PRIVILEGES ON autobrowse.* TO 'autobrowse'@'%';
FLUSH PRIVILEGES;
```

---

## 3. 浏览器插件打包

> [!tip] 两类插件
> - `--type execution`：任务运行插件
> - `--type recorder`：录制插件
>
> 发版前先把 `--version` 递增；`--browser` 可选 `chrome` / `firefox` / `both`。

### 3.1 常规打包（根目录 `build_extension.py`）

```bash
# 任务运行插件
python build_extension.py --type execution --version 1.0.27

# 录制插件
python build_extension.py --type recorder --version 1.0.18
```

### 3.2 只打 Firefox 包

```bash
python build_extension.py --type recorder  --version 1.0.1  --browser firefox
python build_extension.py --type execution --version 1.0.26 --browser firefox
```

### 3.3 Firefox 签名并发布到 AMO

> [!note]
> 签名耗时较长，需要等待。

```bash
python build_extension.py --type execution --version 1.0.1 --browser firefox --sign \
  --amo-api-key user:19903508:572 \
  --amo-api-secret 9a6cf0f3c6802199624d61ca6c8be9cc0794a8df823c5fcf38c64da929a70891

python build_extension.py --type recorder --version 1.0.5 --browser firefox --sign \
  --amo-api-key user:19903508:572 \
  --amo-api-secret 9a6cf0f3c6802199624d61ca6c8be9cc0794a8df823c5fcf38c64da929a70891
```

### 3.4 打包 + 上传 + 发布（推荐）

> [!note] 脚本位置
> 老版本脚本在项目根目录 `build_extension.py`，新版本迁到了 `scripts/build_extension.py`，参数一致。

```bash
# 执行插件 1.0.37：上传 + 发布（签名时间较长，需等待）
python .\build_extension.py --type execution --version 1.0.37 --browser both --upload --publish

# 录制插件 1.0.33：上传 + 发布
python .\build_extension.py --type recorder --version 1.0.33 --browser both --upload --publish
```

```bash
# 带更新说明的完整发版
python .\scripts\build_extension.py --type execution --version 1.1.6 --browser both --upload --publish --release-notes "优化多tab页功能"
python .\scripts\build_extension.py --type execution --version 1.1.10 --browser both --upload --publish --release-notes "优化截图功能"
python .\scripts\build_extension.py --type recorder  --version 1.0.38 --browser both --upload --publish --release-notes "发布录制插件 1.0.38"
```

> [!tip] 斜杠命令
> 在 Claude Code 里可以直接用 `/build-extension`，例如：
> `/build-extension 打包执行插件版本为1.0.37，上传+发布，签名时间会有点长需要等待`

---

## 4. 前端调试登录态注入

浏览器控制台执行，直接以 admin 身份调试，跳过登录：

```javascript
userStore.setToken('eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NjA1MzA0MjAsInN1YiI6IjEifQ.zAonY1wWSVoPZ1qsd-IyTUxRXNlZh9mVXAKFAiZrPDE')

userStore.userInfo = {
  "id": 1,
  "username": "admin",
  "email": "admin@autobrowse.com",
  "role": 1,
  "status": 1,
  "created_at": ""
}
```

---

## 5. 服务器部署

> [!info] 说明
> 本节先用通用路径把流程走一遍，两台实际服务器的具体路径见 [[#5.2 新服务器 10.8.0.32]] 和 [[#5.3 /data 服务器]]。

### 5.1 通用步骤（所有服务器都要做）

**① 拉代码**

```bash
git clone git@github.com:Shortcut-sh02/autobrowse.git
```

**② 修脚本换行符并赋权**

> [!warning] Windows 上编辑过的 `.sh` 会带 `\r`，不处理会报 `bad interpreter`。

```bash
sudo vim /path/to/release_sh/autobrowse.sh
sudo sed -i "s/\r//" /path/to/release_sh/autobrowse.sh && \
  sudo chmod +x /path/to/release_sh/autobrowse.sh
```

**③ 执行发布脚本**

```bash
/path/to/release_sh/autobrowse.sh
```

**④ 配置 Nginx**

```bash
sudo vim /etc/nginx/conf.d/autobrowse.conf

sudo nginx -t                 # 先校验配置
sudo nginx -s reload          # 平滑重载
sudo systemctl restart nginx  # 必要时重启
```

### 5.2 新服务器 10.8.0.32

```bash
cd /home/apexroot/project
git clone git@github.com:Shortcut-sh02/autobrowse.git
cd /home/apexroot/project/autobrowse
```

发布脚本路径：`/home/apexroot/project/release_sh/autobrowse.sh`

```bash
sudo vim /home/apexroot/project/release_sh/autobrowse.sh
sudo sed -i "s/\r//" /home/apexroot/project/release_sh/autobrowse.sh && \
  sudo chmod +x /home/apexroot/project/release_sh/autobrowse.sh

/home/apexroot/project/release_sh/autobrowse.sh
```

Nginx：

```bash
sudo vim /etc/nginx/conf.d/autobrowse.conf
sudo nginx -t
sudo systemctl restart nginx
sudo nginx -s reload
```

### 5.3 /data 服务器

```bash
cd /data
git clone git@github.com:fengchun-yang_shortcut/autobrowse.git autobrowse
cd /data/autobrowse
```

发布脚本路径：`/data/release_sh/autobrowse.sh`

```bash
sudo vim /data/release_sh/autobrowse.sh
sudo sed -i "s/\r//" /data/release_sh/autobrowse.sh && \
  sudo chmod +x /data/release_sh/autobrowse.sh

/data/release_sh/autobrowse.sh
```

Nginx：

```bash
sudo vim /etc/nginx/conf.d/autobrowse.conf

sudo nginx -t
sudo nginx -s reload
sudo systemctl restart nginx
```

### 5.4 MySQL 容器与业务容器同网络

> [!bug] 现象
> 服务连不上数据库。原因：`docker0` 默认 bridge 网络在新服务器上不可用，MySQL 容器不在业务网络里。

```bash
# 查看所有网络
docker network ls

# 把 xj_mysqldb 容器接入业务网络
docker network connect autobrowse_autobrowse-network xj_mysqldb
```

### 5.5 进入 MySQL 容器

> [!note] 密码正确时也可能需要手工输入

```bash
docker exec -it xj_mysqldb /bin/bash
mysql -uroot -papexRootPwd_2025

# 或者一条命令直接连
docker exec -it xj_mysqldb mysql -uroot -papexRootPwd_2025
```

---

## 6. doc-recognition 项目

### 启动步骤

1. **确认 Node 版本 ≥ 24**

   ```bash
   node -v
   ```

2. **安装依赖**

   ```bash
   npm install
   ```

3. **填写 `.env`**

   打开项目根目录的 `.env`，把 `BEDROCK_API_KEY=` 后面填上真实的 Bedrock key。
   `CALLER_KEYS` 和 `BEDROCK_REGION` 已经配好。

4. **启动**

   ```bash
   node --env-file=.env src/server.ts
   ```

5. **验证健康检查**

   ```bash
   curl localhost:8080/healthz
   ```

> [!warning] `npm start` 不会读取 `.env`，必须用第 4 步的命令启动。

### 测试一次识别（可选）

```bash
curl -s -X POST localhost:8080/v1/recognitions \
  -H "Authorization: Bearer dev-key" \
  -H "Content-Type: application/json" \
  -d "{\"type\":\"operation-list\",\"image\":\"$(base64 -w0 sheet.jpg)\"}"
```

> `dev-key` 是 `.env` 里 `CALLER_KEYS` 设的密钥；`sheet.jpg` 换成你的操作单照片。
