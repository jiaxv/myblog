---
title: "树莓派 2B + Raspberry Pi OS (Legacy) 32 位"
date: 2026-07-06
draft: false
---

# 树莓派 2B + Raspberry Pi OS (Legacy) 32 位 Bookworm 完整部署手册

适配机型：树莓派 2B | 系统：Bookworm 32-bit Lite | 全程分步记录，重刷系统可直接对照操作

---

## 一、准备工作

1.  硬件：树莓派 2B、Micro SD 卡（≥8G）、读卡器、数据线（支持数据传输）、安卓手机
2.  电脑：Windows/Linux/macOS，用于烧录系统
3.  工具：**Raspberry Pi Imager**（官方烧录工具，推荐）

---

## 二、系统镜像下载 & 烧录 SD 卡

### 1\. 下载镜像

1.  官网地址：[https://www.raspberrypi.com/software/operating-systems/](https://link.wtturl.cn/?target=https%3A%2F%2Fwww.raspberrypi.com%2Fsoftware%2Foperating-systems%2F&scene=im&aid=497858&lang=zh "autolink")
2.  选择：**Raspberry Pi OS (Legacy, 32-bit) Lite**（Bookworm 精简版，无桌面）
3.  下载 `.zip` 镜像压缩包，无需手动解压

### 2\. 烧录系统（Raspberry Pi Imager）

1.  打开 Raspberry Pi Imager，点击 **Choose OS**

    -   依次选择：`Raspberry Pi OS (Legacy)` → `Raspberry Pi OS (Legacy) Lite (32-bit)`

2.  点击 **Choose Storage**，选中你的 SD 卡（⚠️ 注意盘符，避免选错硬盘）
3.  【可选预配置（免进系统再改）】点击右下角齿轮图标，提前设置：

    -   设置主机名、默认用户名 / 密码（示例：用户名`jiaxf`）
    -   开启 `Enable SSH`（选择密码登录）
    -   时区、键盘布局按实际选择

4.  点击 **Write**，确认后开始烧录，等待完成并弹出 SD 卡。

### 3\. 首次上电

1.  将烧好的 SD 卡插入树莓派 2B，接好电源、网线
2.  上电开机，首次启动会自动扩容分区，等待 1~2 分钟进入命令行

---

## 三、基础网络配置（固定有线 IP）

系统默认使用 `NetworkManager`，使用 `nmcli`/`nmtui` 配置，目标参数：

-   IP：`192.168.1.239/24`
-   网关：`192.168.1.1`
-   DNS：`223.5.5.5`、`114.114.114.114`

### 方式 1：命令行 nmcli（推荐，适合脚本）

bash

运行

```
# 添加静态IP连接
sudo nmcli connection add \
  type ethernet \
  ifname eth0 \
  con-name "eth0-static" \
  autoconnect yes \
  ipv4.addresses 192.168.1.239/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "223.5.5.5 114.114.114.114" \
  ipv4.method manual

# 激活配置
sudo nmcli connection up eth0-static
```

### 方式 2：文本界面 nmtui（可视化操作）

1.  执行命令进入 TUI 界面：

    bash

    运行

    ```
    sudo nmtui
    ```

2.  选择 `Edit a connection` → 选中有线网卡 → `Edit`
3.  `IPv4 CONFIGURATION` 改为 `Manual`
4.  填写地址、网关、DNS，**取消勾选** `Never use this network for default route`
5.  保存退出，执行激活：

    bash

    运行

    ```
    sudo nmtui connect "Wired connection 1"
    ```

### 验证配置

bash

运行

```
ip a show eth0
ip r
```

能看到 `192.168.1.239/24` 即为配置成功。

---

## 四、替换为中科大软件源

### 1\. 一键换源脚本

bash

运行

```
#!/bin/bash
echo "===== 切换为中科大 Bookworm 源 ====="
# 备份原有源文件
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo cp /etc/apt/sources.list.d/raspi.list /etc/apt/sources.list.d/raspi.list.bak

# 替换主源
sudo tee /etc/apt/sources.list >/dev/null <<'EOF'
deb http://mirrors.ustc.edu.cn/raspbian/raspbian/ bookworm main contrib non-free rpi
# deb-src http://mirrors.ustc.edu.cn/raspbian/raspbian/ bookworm main contrib non-free rpi
EOF

# 替换树莓派专属源
sudo tee /etc/apt/sources.list.d/raspi.list >/dev/null <<'EOF'
deb http://mirrors.ustc.edu.cn/raspberrypi/debian/ bookworm main
# deb-src http://mirrors.ustc.edu.cn/raspberrypi/debian/ bookworm main
EOF

# 更新软件缓存
sudo apt update
echo "✅ 换源完成"
```

### 2\. 补充说明

-   出现 `arm64` 架构跳过提示、密钥警告均为**正常现象**，不影响使用
-   树莓派 2B 为 32 位`armhf`，源自动跳过 64 位包，无需处理

---

## 五、配置安卓 USB 网络共享（自动双网切换）

实现：插手机自动走 USB 网络，拔手机自动切回有线网

### 1\. 前置要求

1.  安卓手机开启**移动数据** + **USB 网络共享**
2.  使用**数据数据线**连接手机与树莓派 USB 口

### 2\. 配置 usb0 网卡（设置路由优先级）

bash

运行

```
# 删除旧配置（如有）
sudo nmcli connection delete usb-tether 2>/dev/null

# 新建连接，设置metric=50（优先级高于有线eth0）
sudo nmcli connection add \
  type ethernet \
  ifname usb0 \
  con-name usb-tether \
  autoconnect yes \
  ipv4.method auto \
  ipv4.route-metric 50

# 激活连接
sudo nmcli connection up usb-tether
```

### 3\. 路由优先级说明

-   `usb0 metric=50`（高优先级，插手机优先走手机网络）
-   `eth0 metric=100`（低优先级，拔手机自动回落至有线）

### 4\. 验证

bash

运行

```
ip r
ping 223.5.5.5
```

能正常 ping 通公网即配置生效。

---

## 六、开启 SSH 远程连接

### 方式 1：raspi-config 图形工具

bash

运行

```
sudo raspi-config
```

1.  选择 `3 Interface Options` → `I2 SSH`
2.  选择 `Yes` 开启，退出即可

### 方式 2：纯命令行

bash

运行

```
# 安装SSH服务（Lite版默认未装）
sudo apt install -y openssh-server
# 开机自启 + 立即启动
sudo systemctl enable --now ssh
```

### 验证

bash

运行

```
sudo systemctl status ssh
```

显示 `active (running)` 即为正常。

其他电脑执行 `ssh 用户名@192.168.1.239` 即可远程登录。

---

## 七、安装纯 startx 版 LXDE 桌面 + 完整中文支持

特点：**无 lightdm、开机默认命令行 (CUI)、省内存、中文显示 + 拼音输入法**

### 1\. 一键安装脚本

bash

运行

```
#!/bin/bash
echo "===== 安装纯startx LXDE + 完整中文 ====="
sudo apt update -y

# 安装Xorg+LXDE最小组件+中文字体
sudo apt install -y --no-install-recommends \
xserver-xorg xinit xserver-xorg-video-fbdev \
lxde-core lxsession lxpanel pcmanfm openbox \
lxterminal raspberrypi-ui-mods \
fonts-wqy-zenhei

# 安装中文输入法 fcitx+谷歌拼音
sudo apt install -y fcitx fcitx-googlepinyin

# 生成中文locale
sudo sed -i 's/# zh_CN.UTF-8 UTF-8/zh_CN.UTF-8 UTF-8/' /etc/locale.gen
sudo locale-gen
sudo update-locale LANG=zh_CN.UTF-8 LC_ALL=zh_CN.UTF-8

# 写入输入法环境变量
echo -e "\n# 中文输入法" >> ~/.bashrc
echo 'export GTK_IM_MODULE=fcitx' >> ~/.bashrc
echo 'export QT_IM_MODULE=fcitx' >> ~/.bashrc
echo 'export XMODIFIERS=@im=fcitx' >> ~/.bashrc

# 强制开机进入命令行，禁用多余图形服务
sudo systemctl set-default multi-user.target
sudo systemctl disable lightdm 2>/dev/null

echo "✅ 安装完成，请执行 sudo reboot 重启生效"
```

### 2\. 使用方法

1.  重启系统：`sudo reboot`
2.  开机默认进入命令行，输入以下命令进入桌面：

    bash

    运行

    ```
    startx
    ```

3.  输入法切换：`Ctrl + 空格`
4.  桌面点击「注销」，退回命令行，完全释放内存。

---

## 八、系统升级说明（避坑重点）

执行升级查看：

bash

运行

```
apt list --upgradable
```

1.  **`linux-image-rpi-v8 (arm64)`**：64 位内核，树莓派 2B 为 32 位架构，**无需升级**
2.  **`rpi-eeprom`**：仅树莓派 4/5 系列使用，2B 无 EEPROM 固件，**无需升级**
3.  常规安全升级（只更新有效包）：

    bash

    运行

    ```
    sudo apt update
    sudo apt upgrade -y
    ```

---

## 九、最终整体状态汇总

表格

| 项目 | 配置详情 |
| --- | --- |
| 系统 | Raspberry Pi OS Legacy 32-bit Bookworm Lite |
| 有线 IP | 静态 192.168.1.239/24，网关 192.168.1.1，DNS 223.5.5.5/114.114.114.114 |
| 软件源 | 中科大镜像源 |
| 网络 | 有线 (metric100)+USB 手机共享 (metric50)，插拔自动切换 |
| 远程 | SSH 已开启，支持密码登录 |
| 桌面 | LXDE 纯 startx 启动，无 lightdm，开机默认 CUI |
| 中文 | 文泉驿字体 + fcitx 谷歌拼音，显示 / 输入全中文 |
| 适用场景 | 日常运维、硬件开发、GPIO 调试、低内存稳定运行 |