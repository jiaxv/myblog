---
title: "Armbian 部署 CHFS WebDAV 服务完整记录（2026-07-05）"
date: 2026-07-05
draft: false
---

# Armbian 部署 CHFS WebDAV 服务完整记录（2026-07-05）

## 一、部署目标

在 N1 盒子（Armbian 系统）上搭建 WebDAV 服务，作为个人文件存储和同步中心，支持：

- 浏览器 Web 界面访问

- Windows 映射网络驱动器

- Android 客户端（Voyager）连接

- 通过 Tailscale 实现远程安全访问

---

## 二、环境信息

| 项目           | 详情                                                          |
| ------------ | ----------------------------------------------------------- |
| 主机           | N1 盒子                                                       |
| 系统           | Armbian（基于 Debian）                                          |
| 架构           | ARM64 (aarch64)                                             |
| 用户           | jiaxf                                                       |
| 共享目录         | `/home/jiaxf`                                               |
| 服务端口         | 8080                                                        |
| 内网 IP        | 192.168.1.254                                               |
| Tailscale IP | 100.126.205.77                                              |
| MagicDNS     | [armbian.elf-tint.ts.net](https://armbian.elf-tint.ts.net/) |

---

## 三、CHFS 版本选择

**最终选定版本：CHFS 1.1**

### 版本演进过程

- 最初尝试 CHFS 3.1，虽然支持 `--webdav` 参数，但 Web 界面登录存在异常（登录按钮无反应），且对客户端兼容性不佳。

- 降级至 **CHFS 1.1** 后，所有功能正常：
  
  - 浏览器 Web 界面登录正常工作。
  
  - Windows 映射网络驱动器成功。
  
  - Android Voyager 正常连接。
  
  - WebDAV 协议支持稳定。

---

## 四、安装步骤

### 1. 下载 CHFS 1.1（ARM64）

bash

cd /tmp
wget http://iscute.cn/tar/chfs/1.1/chfs-linux-arm64-1.1.zip
unzip -o chfs-linux-arm64-1.1.zip -d /usr/local/bin
chmod 500 /usr/local/bin/chfs

### 2. 验证版本

bash

/usr/local/bin/chfs -version

# 输出应显示 1.1

---

## 五、服务配置

### 1. systemd 服务文件

路径：`/etc/systemd/system/chfs.service`

ini

[Unit]
Description=chfs WebDAV Server
After=network.target
[Service]
User=root
Type=simple
ExecStart=/usr/local/bin/chfs --path="/home/jiaxf" --port=8080 --rule="::|admin:nimda:RWD"
Restart=on-failure
[Install]
WantedBy=multi-user.target

**参数说明**：

- `--path`：共享目录路径（使用双引号包裹）。

- `--port`：监听端口。

- `--rule`：访问控制规则。
  
  - `::` 表示禁止匿名访问。
  
  - `|` 为分隔符。
  
  - `admin:nimda:RWD` 定义用户 `admin`，密码 `nimda`，拥有读写删（RWD）权限。

> 若希望浏览器访问时无需登录即可查看文件（只读），可将规则改为 `admin:nimda:RWD`（去掉 `::|`），但登录切换功能可能受影响。

### 2. 启动与自启

bash

sudo systemctl daemon-reload
sudo systemctl start chfs
sudo systemctl enable chfs
sudo systemctl status chfs   # 确认状态为 active (running)

### 3. 验证监听

bash

sudo ss -tulpn | grep 8080

# 应显示 chfs 监听在 *:8080

---

## 六、WebDAV 端点说明

CHFS 的 WebDAV 服务端点位于 **`/webdav`** 子路径：

| 用途           | 地址                      |
| ------------ | ----------------------- |
| 浏览器 Web 界面   | `http://IP:8080/`       |
| WebDAV 客户端连接 | `http://IP:8080/webdav` |

**重要**：客户端必须使用包含 `/webdav` 的完整地址，否则会返回 `405 Method Not Allowed`。

---

## 七、客户端连接方法

### 1. 浏览器访问（任意设备）

- 访问 `http://100.126.205.77:8080` 或 `http://armbian.elf-tint.ts.net:8080`

- 页面加载后，点击 **“登录”** 按钮，输入用户名 `admin`，密码 `nimda`。

- 登录成功后即可浏览、上传、下载文件。

> 若规则为 `::|admin:nimda:RWD`，访问时页面为空，登录后显示文件列表。  
> 若清除浏览器缓存后登录按钮无反应，请使用无痕模式或重启浏览器。

---

### 2. Windows 映射网络驱动器

**前提**：Windows 默认禁用 HTTP 基本认证，需修改注册表。

#### 步骤：

1. **修改注册表**
   
   - 按 `Win+R`，输入 `regedit`，定位到：  
     `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\WebClient\Parameters`
   
   - 将 `BasicAuthLevel` 的值改为 `2`（允许 HTTP 基本认证）。

2. **重启 WebClient 服务**
   
   - 按 `Win+R`，输入 `services.msc`，找到 `WebClient`，右键“重新启动”。

3. **映射网络驱动器**
   
   - 打开“此电脑” → “映射网络驱动器”。
   
   - 文件夹地址：`http://armbian.elf-tint.ts.net:8080/webdav`（或使用 IP）。
   
   - 勾选“使用其他凭据连接”，点击完成。
   
   - 输入用户名 `admin`，密码 `nimda`，并勾选“记住我的凭据”。

完成后，即可在“此电脑”看到新盘符，像本地磁盘一样操作。

---

### 3. Android 客户端（Voyager）

在 Voyager 中添加新连接，填写以下信息：

| 字段          | 填写内容                                         |
| ----------- | -------------------------------------------- |
| 协议          | WebDAV                                       |
| Host        | `armbian.elf-tint.ts.net` 或 `100.126.205.77` |
| Port        | `8080`                                       |
| Remote Path | `/webdav`                                    |
| Username    | `admin`                                      |
| Password    | `nimda`                                      |

> 若 Voyager 不支持单独设置 Remote Path，可将 Host 填为 `armbian.elf-tint.ts.net/webdav`，端口仍填 `8080`。

**其他推荐客户端**：Solid Explorer、CX 文件管理器。

---

## 八、常见问题与解决方案

| 问题现象                                        | 原因                                  | 解决方法                                             |
| ------------------------------------------- | ----------------------------------- | ------------------------------------------------ |
| 浏览器访问根路径显示空页面，登录按钮无反应                       | 浏览器缓存了旧会话/Cookie                    | 清除浏览器缓存和 Cookie，或使用无痕模式                          |
| 浏览器访问 `/webdav` 返回 `405 Method Not Allowed` | `/webdav` 是 WebDAV 端点，不支持浏览器 GET 请求 | 这是正常现象，客户端使用 PROPFIND 方法访问即可                     |
| Windows 映射失败，提示“网络错误”                       | 未修改注册表或未重启 WebClient 服务             | 按第七节步骤修改注册表并重启服务                                 |
| Voyager 连接失败（即使设置了 Remote Path）             | Voyager 可能忽略 Remote Path 字段         | 将路径合并到 Host 中，如 `armbian.elf-tint.ts.net/webdav` |
| 升级到 CHFS 3.1 后 `--webdav` 参数不识别             | 下载的版本不对，或参数格式错误                     | 降级回 1.1 版本，稳定可靠                                  |
| 登录后仍无法上传文件                                  | 规则中未给用户写入权限                         | 确保规则为 `admin:nimda:RWD`（包含 W 和 D）                |

---

## 九、最终确认状态

- CHFS 1.1 服务正常运行，监听 8080 端口

- systemd 服务已启用，开机自启

- 浏览器可访问 Web 界面，登录正常

- Windows 映射网络驱动器成功

- Android Voyager 正常连接

- Tailscale 远程访问正常（域名和 IP 均可）

---

## 十、参考命令速查

bash

# 服务管理

sudo systemctl start chfs
sudo systemctl stop chfs
sudo systemctl restart chfs
sudo systemctl status chfs

# 查看日志

sudo journalctl -u chfs -f

# 测试 WebDAV（curl）

curl -u admin:nimda -X PROPFIND http://127.0.0.1:8080/webdav -H "Depth: 1"

# 查看版本

/usr/local/bin/chfs -version

# 查看端口监听

sudo ss -tulpn | grep 8080

# Tailscale 信息

tailscale ip
tailscale status

---

**文档结束**  
记录日期：2026-07-05  
部署者：jiaxf
