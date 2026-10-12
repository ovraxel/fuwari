---
title: 從零開始：用 Termux 把 Android 手機變成 Linux 終端
published: 2026-10-12
description: 詳細教學如何正確安裝 Termux、換源、常用指令與實用套件，讓你在手機上也能輕鬆使用 Linux 環境。
image: ''
tags: [Termux, Android, Linux, 教學]
category: 教學
draft: false
---

**Termux** 是目前 Android 上最實用的終端模擬器，不需要 root 就能擁有接近完整的 Linux 環境。

這篇文章會從安裝、基本設定到常用工具，帶你一步步把手機變成隨身終端機。

## 一、正確安裝 Termux（重要！）

**請不要從 Google Play 安裝舊版 Termux**，官方已不再維護 Play 商店版本，會導致套件無法更新。

### 推薦安裝方式

1. 前往 [F-Droid 官方頁面](https://f-droid.org/en/packages/com.termux/)
2. 點擊 **Download APK** 下載最新版本
3. 允許「未知來源」後安裝

也可以從 [GitHub Releases](https://github.com/termux/termux-app/releases) 下載。

> 安裝完成後第一次開啟，Termux 會自動進行初始化，請耐心等待完成。

## 二、第一次開啟後必做的事

### 1. 更新套件列表並升級

```bash
pkg update && pkg upgrade -y
```

### 2. 更換國內鏡像源（中國大陸用戶強烈建議）

```bash
termux-change-repo
```

選擇 **Mirrors in Chinese Mainland**（國內鏡像組），之後下載速度會快很多。

### 3. 開啟儲存空間權限

```bash
termux-setup-storage
```

系統會彈出權限請求，請允許。之後就可以透過 `~/storage/` 存取手機的下載、照片等資料夾。

## 三、常用基本指令

| 指令 | 說明 |
|------|------|
| `ls` | 列出目前目錄內容 |
| `cd 資料夾名稱` | 進入資料夾 |
| `cd ..` | 回到上一層 |
| `pwd` | 顯示目前路徑 |
| `mkdir 資料夾` | 建立資料夾 |
| `rm 檔案` | 刪除檔案 |
| `rm -rf 資料夾` | 刪除資料夾（小心使用） |
| `cp 來源 目標` | 複製 |
| `mv 來源 目標` | 移動或重新命名 |
| `clear` | 清空螢幕 |

## 四、推薦安裝的基礎套件

一次安裝常用工具：

```bash
pkg install -y git curl wget nano vim unzip zip tar tree htop neofetch
```

### 各套件用途簡介

- **git**：版本控制
- **curl / wget**：下載檔案
- **nano**：簡單文字編輯器（新手友善）
- **vim**：強大編輯器
- **unzip / zip / tar**：壓縮與解壓縮
- **tree**：樹狀顯示資料夾結構
- **htop**：查看系統資源
- **neofetch**：顯示系統資訊

## 五、安裝開發環境（依需求）

### Python

```bash
pkg install python
pip install --upgrade pip
```

### Node.js

```bash
pkg install nodejs
```

### OpenSSH（可從電腦連線到手機）

```bash
pkg install openssh
sshd
```

預設 SSH 埠為 **8022**。

## 六、實用小技巧

### 1. 讓 Termux 在背景保持運行

到手機設定 → 電池 → 省電策略，把 Termux 設為「不限制」。

### 2. 使用快捷鍵

- `Ctrl + C`：中斷目前指令
- `Ctrl + D`：退出目前 shell
- `Ctrl + L`：清屏（等同 `clear`）
- 音量上鍵 + `Q`：顯示額外鍵盤

### 3. 快速進入下載資料夾

```bash
cd ~/storage/downloads
```

### 4. 一鍵初始化腳本（懶人版）

```bash
pkg update && pkg upgrade -y && \
pkg install -y git curl wget nano vim unzip zip tar tree htop neofetch python nodejs openssh && \
termux-setup-storage
```

## 七、常見問題排除

**問題：pkg 找不到套件**

先執行 `pkg update`，再試一次。

**問題：下載速度很慢**

重新執行 `termux-change-repo` 換其他鏡像。

**問題：無法存取手機檔案**

確認已執行過 `termux-setup-storage`，並在系統設定中給予儲存權限。

## 結語

Termux 讓手機不再只是通訊娛樂工具，也能成為隨身的 Linux 工作站。

從寫腳本、跑 Python、用 Git 管理專案，到架設簡易伺服器，都能在手機上完成。

如果你有其他想深入的主題（例如用 Termux 架設網站、跑 proot-distro 安裝完整 Linux 發行版、或搭配 Termux:API 操作手機功能），歡迎留言告訴我，我再寫後續文章。