---
title: NVM 簡介
date: 2025-09-07 14:29:27
tags:
- Web
---

在網頁開發中， Node.js 已成為不可或缺的工具。然而，專案之間常常需要不同版本的 Node.js ，例如舊專案需要 `14.x` ，新專案則可能使用 `20.x` 。此時，使用 NVM (Node Version Manager) 來管理多版本 Node.js 就變得非常重要。

### 什麼是 NVM ？

NVM 是一個命令列工具，可在主機上安裝多個版本的 Node.js ，解決不同專案間快速切換 Node.js 版本，同時避免全域安裝 Node.js 帶來的相容性問題。

### 於 Windows 下安裝 NVM

由於 Windows 沒有官方的 NVM ，因此特別介紹使用 nvm-windows 來進行安裝。

1. 下載 [nvm-windows 發行檔](https://github.com/coreybutler/nvm-windows/releases)（建議安裝 `1.1.12` 版本較穩定）。

2. 執行安裝程式並設定安裝目錄。

  > 若曾經安裝過 Node.js ，需要先完整移除
  {% asset_img 0_uninstall_node.js.png %}

  2.1 NVM 安裝路徑
  {% asset_img 1_nvm_1.1.12_folder.png %}

  2.2 Node.js 設定路徑
  {% asset_img 2_nvm_1.1.12_node.js_folder.png %}

3. 安裝完成後，開啟 PowerShell 檢查版本：
  `nvm --version`

### NVM 常用指令

- `nvm install <version>` 安裝指定版本的 Node.js，例如 `nvm install 18.20.2`
- `nvm install latest` 安裝最新的 LTS（長期支援）版本
- `nvm uninstall <version>` 移除某個版本的 Node.js
- `nvm use <version>` 切換至指定版本，例如 `nvm use 18`
- `nvm current` 查看目前使用的 Node.js 版本
- `nvm list` 或 `nvm ls` 列出已安裝的 Node.js 版本

### 使用情境

使用管理員身份開啟 PowerShell

{% asset_img 3_0_PowerShell.png %}

1. **安裝指定版本的 Node.js**
  1.1 以 `nvm list` 確認目前已安裝的版本
  {% asset_img 3_1_nvm_list.png %}
  1.2 安裝 Node.js `18.20.2`
  {% asset_img 3_2_nvm_install_18.20.2.png %}
  1.3 以 `nvm list` 確認已安裝該版本的 Node.js
  {% asset_img 3_3_nvm_list.png %}
  1.4 以 `nvm use` 切換至該 Node.js `18.20.2` 版本
  {% asset_img 3_4_nvm_use_18.20.2.png %}
  1.5 以 `node -v` 或 `nvm current` 檢查 Node.js 版本
  {% asset_img 3_5_node.js_version.png %}

2. **安裝最新 LTS 版本的 Node.js 並切換至該版本**
  2.1 以 `nvm install latest` 安裝最新 LTS 版本
  {% asset_img 4_1_nvm_install_latest.png %}
  2.2 以 `nvm list` 確認已安裝該版本的 Node.js （`*` 代表目前使用的版本）
  {% asset_img 4_2_nvm_list.png %}
  2.3 以 `nvm use` 切換至該 Node.js `24` 主版本
  {% asset_img 4_3_nvm_use_24.png %}
  2.4 以 `nvm list` 及 `nvm current` 確認已切換至指定版本
  {% asset_img 4_4_nvm_list&current.png %}

  > 由檔案結構可以發現，使用 `nvm install <version>` 安裝的 Node.js 版本都會存在 NVM 資料夾中，並且設定軟連結（symbolic link）到 Node.js 的執行路徑，藉由＂捷徑＂快速切換 Node.js 版本。
  {% asset_img 4_5_node.js_symlink.png %}
