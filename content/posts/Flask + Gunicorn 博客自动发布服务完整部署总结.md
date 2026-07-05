---
title: "Flask + Gunicorn 博客自动发布服务完整部署总结"
date: 2026-07-05
draft: false
---

# Flask + Gunicorn 博客自动发布服务完整部署总结

> 部署日期：2026-07-05  
> 部署环境：Armbian (N1 盒子)  
> 部署用户：jiaxf

---

## 一、整体架构概览

text

┌─────────────────────────────────────────────────────────────────────────┐
│                          用户浏览器 / 手机                             │
│                    访问 http://N1:5000 上传 .md 文件                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Flask 上传服务 (Gunicorn)                         │
│  1. 接收 .md 文件                                                       │
│  2. 保存到 ~/blog/content/posts/                                       │
│  3. 自动补全 Front Matter (title + date)                               │
│  4. 调用 deploy.sh                                                     │
└────────────────────────────┬────────────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         deploy.sh 部署脚本                             │
│  1. cd ~/blog                                                          │
│  2. git add content/posts/                                             │
│  3. git commit -m "..."                                                │
│  4. git push origin main                                               │
└────────────────────────────┬────────────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     GitHub 仓库 (jiaxv/myblog)                         │
│                     收到新提交，触发 EdgeOne                            │
└────────────────────────────┬────────────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                  腾讯云 EdgeOne Pages (云端构建)                        │
│  1. 拉取最新源码                                                       │
│  2. 执行 hugo -D (生成 public/)                                        │
│  3. 部署到 CDN 节点                                                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        博客网站 (上线)                                  │
│                   https://你的博客域名                                   │
└─────────────────────────────────────────────────────────────────────────┘

---

## 二、各组件职责说明

| 组件                | 职责                   | 关键文件/命令                                 |
| ----------------- | -------------------- | --------------------------------------- |
| **Flask**         | Web 框架，提供上传页面和 API   | `app.py`                                |
| **Gunicorn**      | WSGI 服务器，生产级运行 Flask | `gunicorn -w 2 -b 0.0.0.0:5000 app:app` |
| **Python venv**   | 隔离 Python 依赖环境       | `python3 -m venv venv`                  |
| **deploy.sh**     | 自动化 Git 推送脚本         | `~/deploy.sh`                           |
| **Hugo**          | 静态站点生成器              | 在 EdgeOne 云端执行                          |
| **EdgeOne Pages** | 云端构建 + CDN 部署        | 腾讯云控制台管理                                |
| **GitHub**        | 源码托管 + Webhook 触发    | `git@github.com:jiaxv/myblog.git`       |

---

## 三、目录结构

text

/home/jiaxf/
├── blog/                          # Hugo 博客源码 (Git 仓库)
│   ├── content/
│   │   └── posts/                 # 文章存放目录
│   ├── themes/                    # 主题
│   ├── config.toml               # Hugo 配置
│   └── .gitignore                # 忽略 public/、.frontmatter/ 等
├── flask-upload/                  # Flask 服务目录
│   ├── app.py                    # Flask 应用主文件
│   ├── venv/                     # Python 虚拟环境
│   └── deploy.sh                 # 部署脚本
└── deploy.sh                      # 部署脚本 (与 flask-upload/ 共用)

---

## 四、完整部署步骤

### 4.1 安装系统依赖

bash

# 安装 Python 虚拟环境支持

sudo apt install python3-venv -y

# 安装 Git

sudo apt install git -y

# 安装 Hugo (可选，N1 本身不需要，但便于本地测试)

# 实际构建在 EdgeOne 云端进行

### 4.2 创建 Flask 服务目录和虚拟环境

bash

mkdir -p ~/flask-upload
cd ~/flask-upload
python3 -m venv venv
source venv/bin/activate
pip install flask gunicorn

### 4.3 编写 app.py

完整代码见文末附录 A。核心功能：

- 提供 Web 上传界面

- 保存 `.md` 文件到 `~/blog/content/posts/`

- 自动补全 Front Matter（title + date）

- 调用 `deploy.sh` 推送至 GitHub

### 4.4 编写 deploy.sh

`~/deploy.sh`：

bash

#!/bin/bash
set -e
echo "=== 开始部署 $(date) ==="
cd /home/jiaxf/blog || exit 1
git add content/posts/
git commit -m "自动部署: $(date '+%Y-%m-%d %H:%M:%S')" || echo "无变更可提交"
git push origin main
echo "=== 部署完成 ==="

赋予执行权限：

bash

chmod +x ~/deploy.sh

### 4.5 配置 systemd 服务

`/etc/systemd/system/flask-upload.service`：

ini

[Unit]
Description=Flask Upload Service (Hugo Blog)
After=network.target
[Service]
User=jiaxf
Group=jiaxf
WorkingDirectory=/home/jiaxf/flask-upload
Environment="PATH=/home/jiaxf/flask-upload/venv/bin"
Environment="LANG=zh_CN.UTF-8"
Environment="LC_ALL=zh_CN.UTF-8"
ExecStart=/home/jiaxf/flask-upload/venv/bin/gunicorn -w 2 -b 0.0.0.0:5000 app:app
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target

启动服务：

bash

sudo systemctl daemon-reload
sudo systemctl start flask-upload
sudo systemctl enable flask-upload

### 4.6 配置 SSH 密钥 (用于 Git 推送)

bash

# 生成密钥 (如果还没有)

ssh-keygen -t ed25519 -C "jiaxf@armbian"

# 查看公钥并添加到 GitHub

cat ~/.ssh/id_ed25519.pub

GitHub 添加位置：Settings → SSH and GPG keys → New SSH Key

测试连接：

bash

ssh -T git@github.com

### 4.7 访问测试

- Tailscale IP：`http://100.126.205.77:5000`

- MagicDNS：`http://armbian.elf-tint.ts.net:5000`

上传一个 `.md` 文件，检查：

1. 文件是否保存到 `~/blog/content/posts/`

2. Front Matter 是否自动补全

3. 是否触发 `deploy.sh` 推送

4. EdgeOne 是否自动构建

5. 博客网站是否更新

---

## 五、关键技术点说明

### 5.1 为什么不用 Flask 自带的开发服务器？

| 对比项  | Flask 自带      | Gunicorn  |
| ---- | ------------- | --------- |
| 并发处理 | 单进程，一次只处理一个请求 | 多进程，支持并发  |
| 稳定性  | 容易崩溃          | 自动重启，长期运行 |
| 安全性  | 不适合生产环境       | 经过安全加固    |
| 性能   | 低             | 高         |

### 5.2 为什么用虚拟环境 (venv)？

- 隔离 Python 依赖，不污染系统 Python

- 便于版本管理和迁移

- 符合 Python 社区最佳实践

### 5.3 为什么本地不执行 hugo？

- EdgeOne Pages 在云端执行构建，更高效

- N1 性能有限，不适合处理构建任务

- 推送源码即可，EdgeOne 自动完成后续流程

### 5.4 为什么需要自动补全 Front Matter？

Hugo 依赖 Front Matter 中的 `title` 和 `date` 来生成页面元数据。缺少时，主页列表无法显示标题和日期，文章页也可能出现异常。

---

## 六、常用管理命令

### 6.1 服务管理

bash

# 启动

sudo systemctl start flask-upload

# 停止

sudo systemctl stop flask-upload

# 重启 (修改代码后)

sudo systemctl restart flask-upload

# 查看状态

sudo systemctl status flask-upload

# 查看实时日志

sudo journalctl -u flask-upload -f

# 查看最近 50 行日志

sudo journalctl -u flask-upload -n 50

### 6.2 Git 操作 (手动触发)

bash

cd ~/blog
git add content/posts/
git commit -m "手动更新"
git push origin main

### 6.3 快速测试

bash

# 测试上传服务是否可访问

curl http://100.126.205.77:5000

# 测试 deploy.sh

~/deploy.sh

---

## 七、故障排查指南

| 症状              | 可能原因              | 解决方法                                 |
| --------------- | ----------------- | ------------------------------------ |
| 浏览器无法访问 5000 端口 | 服务未启动或端口被占用       | `sudo systemctl status flask-upload` |
| 上传后提示部署成功但网站无更新 | Git 推送失败          | 检查 SSH 密钥: `ssh -T git@github.com`   |
| EdgeOne 构建失败    | .md 文件格式问题        | 检查 Front Matter 语法                   |
| 文章标题显示为文件名      | 缺少 Front Matter   | 检查 app.py 的补全逻辑                      |
| 标题和日期重复显示       | 正文中手写了标题和日期       | 删除正文中的重复内容                           |
| 服务启动失败          | Gunicorn 未安装或路径错误 | 检查 venv 路径，重新 pip install            |

---

## 八、附录

### 附录 A：完整 app.py 代码

python

#!/usr/bin/env python3

# -*- coding: utf-8 -*-

"""
Flask 上传服务 - 自动补全 Hugo Front Matter
功能：接收 .md 文件，保存到 Hugo 的 content/posts/，自动补全 title 和 date，
      然后调用 deploy.sh 推送至 GitHub，触发 EdgeOne 构建。
"""
import os
import re
import datetime
import subprocess
from flask import Flask, request, render_template_string
app = Flask(__name__)

# ========== 配置 ==========

BLOG_PATH = "/home/jiaxf/blog"
DEPLOY_SCRIPT = "/home/jiaxf/deploy.sh"

# ========== HTML 模板 ==========

UPLOAD_FORM = """
<!DOCTYPE html>

<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>上传 MD 文件到博客</title>
    <style>
        body { font-family: sans-serif; max-width: 600px; margin: 50px auto; padding: 20px; background: #f9f9f9; }
        .container { background: white; padding: 30px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        h2 { margin-top: 0; }
        input[type="file"] { display: block; margin: 15px 0; }
        button { background: #007bff; color: white; border: none; padding: 10px 20px; border-radius: 5px; cursor: pointer; }
        button:hover { background: #0056b3; }
        .msg { margin-top: 15px; padding: 10px; border-radius: 4px; }
        .success { background: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
        .error { background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
    </style>
</head>
<body>
    <div class="container">
        <h2>📄 上传 Markdown 文件</h2>
        <form method="post" enctype="multipart/form-data" action="/upload">
            <input type="file" name="file" accept=".md" required>
            <button type="submit">上传并部署</button>
        </form>
        {% if msg %}
            <div class="msg {% if '成功' in msg %}success{% else %}error{% endif %}">
                {{ msg | safe }}
            </div>
        {% endif %}
    </div>
</body>
</html>
"""
# ========== Front Matter 补全函数 ==========
def ensure_frontmatter(filepath, title=None):
    """检查并补全 Markdown 文件的 Front Matter"""
    if not os.path.exists(filepath):
        return
    with open(filepath, 'r', encoding='utf-8') as f:
        content = f.read()
    if not content.strip():
        new_content = f"---\ntitle: \"{title}\"\ndate: {datetime.date.today()}\ndraft: false\n---\n"
        with open(filepath, 'w', encoding='utf-8') as f:
            f.write(new_content)
        return
    front_matter_match = re.match(r'^---\s*\n(.*?)\n---\s*\n', content, re.DOTALL)
    if front_matter_match:
        front_matter = front_matter_match.group(1)
        lines = front_matter.split('\n')
        has_title = any(line.strip().startswith('title:') for line in lines)
        has_date = any(line.strip().startswith('date:') for line in lines)
        need_update = False
        new_lines = []
        if not has_title:
            new_lines.append(f'title: "{title}"')
            need_update = True
        if not has_date:
            new_lines.append(f'date: {datetime.date.today()}')
            need_update = True
        if need_update:
            front_matter_new = front_matter
            if front_matter and not front_matter.endswith('\n'):
                front_matter_new += '\n'
            front_matter_new += '\n'.join(new_lines) + '\n'
            new_content = content.replace(front_matter_match.group(1), front_matter_new, 1)
            with open(filepath, 'w', encoding='utf-8') as f:
                f.write(new_content)
    else:
        new_content = f"---\ntitle: \"{title}\"\ndate: {datetime.date.today()}\ndraft: false\n---\n\n{content}"
        with open(filepath, 'w', encoding='utf-8') as f:
            f.write(new_content)
# ========== 路由 ==========
@app.route('/')
def index():
    return render_template_string(UPLOAD_FORM, msg="")
@app.route('/upload', methods=['POST'])
def upload_file():
    try:
        file = request.files['file']
        if not file or not file.filename.endswith('.md'):
            return render_template_string(UPLOAD_FORM, msg="❌ 请上传 .md 文件")
        filename = os.path.basename(file.filename)
        save_path = os.path.join(BLOG_PATH, 'content/posts', filename)
        file.save(save_path)
        title = os.path.splitext(filename)[0]
        ensure_frontmatter(save_path, title)
        result = subprocess.run([DEPLOY_SCRIPT], capture_output=True, text=True)
        if result.returncode == 0:
            msg = f"✅ 文件 <strong>{filename}</strong> 上传并部署成功！"
        else:
            msg = f"❌ 部署失败（返回码 {result.returncode}）：<br><pre>{result.stderr}</pre>"
    except Exception as e:
        msg = f"❌ 异常：{str(e)}"
    return render_template_string(UPLOAD_FORM, msg=msg)
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)

### 附录 B：访问地址

| 网络        | 地址                                    | 说明      |
| --------- | ------------------------------------- | ------- |
| Tailscale | `http://100.126.205.77:5000`          | 固定虚拟 IP |
| MagicDNS  | `http://armbian.elf-tint.ts.net:5000` | 域名访问    |
| 内网        | `http://192.168.1.254:5000`           | 仅限同网段   |

### 附录 C：相关服务端口

| 服务          | 端口   | 说明     |
| ----------- | ---- | ------ |
| CHFS WebDAV | 8080 | 文件管理   |
| Flask 上传服务  | 5000 | 博客文章上传 |

---

> **文档结束**  
> 如有问题，可通过日志排查：`sudo journalctl -u flask-upload -f`
