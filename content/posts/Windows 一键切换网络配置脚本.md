---
title: Windows 一键切换网络配置脚本
date: 2026-06-07T00:36:26+08:00
draft: false
---
﻿# Windows 一键切换网络配置脚本

## 背景
在日常使用中，你可能需要在不同网络环境之间切换 IP 地址配置。例如：
- **静态 IP**：用于特定局域网（如 N1 旁路由网关 `192.168.1.254`）。
- **DHCP**：用于普通家庭/办公网络自动获取 IP。

手动修改不仅繁琐，还容易出错。本脚本可以智能检测当前配置，并**一键切换**到另一种配置。

## 脚本功能
- **自动检测**当前“以太网”适配器的 IP 获取方式（静态或 DHCP）。
- **一键切换**：
  - 若当前为 DHCP，则切换到预设的**静态 IP**。
  - 若当前为静态 IP，则切换到 **DHCP**。
- 静态 IP 配置参数：
  - IP 地址：`192.168.1.95`
  - 子网掩码：`255.255.255.0`
  - 网关：`192.168.1.254`
  - 首选 DNS：`223.5.5.5`
  - 备用 DNS：`114.114.114.114`
- 切换后**显示当前 IP 和默认网关**，方便验证。

## 使用方法
1. 将以下代码保存为 `switch-ip.bat`（编码选择 ANSI 或 UTF-8 with BOM）。
2. **右键点击**该文件，选择 **“以管理员身份运行”**。
3. 等待脚本执行完毕，查看输出结果确认切换成功。

> ⚠️ 必须以管理员权限运行，否则无法修改网络配置。

## 脚本代码
```batch
@echo off
chcp 65001 >nul
setlocal enabledelayedexpansion

set ADAPTER=以太网

:: 使用 PowerShell 检测当前 IP 来源
for /f "tokens=*" %%i in ('powershell -Command "Get-NetIPAddress -InterfaceAlias '%ADAPTER%' -AddressFamily IPv4 | Select-Object -ExpandProperty PrefixOrigin" 2^>nul') do set DHCP_Status=%%i

echo 当前 IP 来源: %DHCP_Status%

if /i "%DHCP_Status%"=="Dhcp" (
    echo 当前为 DHCP，正在切换到静态 IP ...
    netsh interface ip delete address "%ADAPTER%" all >nul 2>&1
    netsh interface ip set address "%ADAPTER%" static 192.168.1.95 255.255.255.0 gateway=192.168.1.254 gwmetric=1
    netsh interface ip set dns "%ADAPTER%" static 223.5.5.5
    netsh interface ip add dns "%ADAPTER%" 114.114.114.114 index=2
    if not errorlevel 1 (
        echo ✅ 已切换为静态 IP
    ) else (
        echo ❌ 切换失败，请检查适配器名称或 IP 冲突
    )
) else (
    echo 当前为静态 IP，正在切换到 DHCP ...
    netsh interface ip set address "%ADAPTER%" dhcp
    netsh interface ip set dns "%ADAPTER%" dhcp
    if not errorlevel 1 (
        echo ✅ 已切换为 DHCP
        ipconfig /renew "%ADAPTER%" >nul 2>&1
    ) else (
        echo ❌ 切换失败
    )
)

echo.
echo 当前 IP 信息:
ipconfig | findstr /C:"IPv4" /C:"默认网关"
echo.
pause