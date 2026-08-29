---
title: 在 Windows 用 WSL 打造 Linux 開發環境
date: 2026-08-29 10:36:55
tags:
- WSL
- Linux
categories:
- WSL
- Linux
---

## 為什麼需要 WSL

在 Windows 上想使用 Linux 環境時，常見的選擇有三種：原生安裝雙系統、使用虛擬機（VM），或是使用 WSL。三者的取捨大致如下：

| 方式 | 效能 | 與 Windows 的整合度 | 設定複雜度 |
|------|------|------|------|
| 雙系統 | 最佳（原生執行） | 低（開機需切換系統） | 高（需分割磁區） |
| 虛擬機（VM） | 較差（需模擬硬體） | 低（檔案、剪貼簿需額外設定共享） | 中（需另裝 Hypervisor） |
| WSL | 接近原生 | 高（可直接存取 Windows 檔案系統、共用剪貼簿） | 低（`wsl --install` 即可） |

WSL（Windows Subsystem for Linux）的核心優勢在於：**不需要切換系統或另外模擬硬體，就能在 Windows 桌面環境下直接使用 Linux 指令列與工具鏈**。這對於需要 Linux 專屬開發工具（例如某些套件只提供 Linux 版本）、或想在 Windows 上練習 Linux 指令、架設後端服務的開發者來說，是最輕量的入門方式。

## 使用 WSL 指令

基本上安裝流程算是很單純，不需要手動去調整安裝目錄，因為微軟會自動處理。

* 列出可供下載的版本
  `wsl --list --online`
  <div class="img-left">{% asset_img 1_wsl_list_online.png %}</div>

* 安裝特定版本，以 Ubuntu 22.04 為例
  `wsl --install -d Ubuntu-22.04`
  <div class="img-left">{% asset_img 2_wsl_install_command.png %}</div>

* 查看目前已安裝的版本
  `wsl --list --verbose` 或 `wsl -l -v`
  在執行這個指令後，會輸出一張表格，列出每個發行版（ distribution ）的三個關鍵資訊：
  - `NAME`：Linux 發行版的內部名稱（例如 `Ubuntu`、`Debian`）。
  - `STATE`：發行版目前的狀態，`Running`（執行中）或 `Stopped`（已停止）。
  - `VERSION`：使用的架構版本，`1` 代表 WSL 1，`2` 代表 WSL 2。
  名稱旁若有星號（`*`），代表這是目前輸入 `wsl` 時預設啟動的發行版。
  如果需要更改，可以使用 `wsl --set-default <DistroName>` 來設定預設值，例如
  `wsl --set-default Ubuntu-22.04`
  完成後出現 "The operation completed successfully." 訊息代表設定成功。

  > 若要移除某個版本，可以使用註銷（ unregister ）指令來徹底刪除它（不需要先切換預設版本），例如
  > `wsl --unregister Ubuntu-22.04`
  > 完成後出現 "The operation completed successfully." 訊息代表刪除成功。

<!-- more -->

## 進入並建立新專案

在終端機輸入 `wsl` 即可進入預設版本的 Linux 環境。

{% asset_img 3_wsl.png %}

建立專案資料夾並初始化 Git 儲存庫
```bash
mkdir -p ~/projects/wsl-flask-demo
cd ~/projects/wsl-flask-demo
git init
```

> WSL 的家目錄（`~`，例如 `/home/<username>`）是 Linux 檔案系統，讀寫效能遠優於掛載在 `/mnt/c/` 底下的 Windows 磁碟。因此開發用的專案建議放在 `~` 底下，而不是 Windows 的 `C:\` 路徑。

## 用 Python Flask 建立簡易後端伺服器

1. 在專案資料夾內，先建立虛擬環境並安裝 Flask
  ```bash
  python3 -m venv .venv
  source .venv/bin/activate
  pip install flask
  ```
  > 若執行 `python3 -m venv .venv` 時出現以下訊息：
  > ```
  > The virtual environment was not created successfully because ensurepip is not
  > available.  On Debian/Ubuntu systems, you need to install the python3-venv
  > package using the following command.
  >
  >     apt install python3.10-venv
  >
  > You may need to use sudo with that command.  After installing the python3-venv
  > package, recreate your virtual environment.
  > ```
  > 這是因為 Ubuntu／Debian 的 `python3` 套件預設不包含 `venv` 模組所需的 `ensurepip`，需要另外安裝 `python3-venv`（或訊息中指定的對應版本，例如 `python3.10-venv`）：
  > ```bash
  > sudo apt update
  > sudo apt install -y python3.10-venv
  > ```
  > 安裝完成後，重新執行 `python3 -m venv .venv` 即可正常建立虛擬環境。

2. 建立 `app.py` 檔案
  ```python
  from flask import Flask

  app = Flask(__name__)

  @app.route("/")
  def hello():
    return "Hello from WSL!"

  if __name__ == "__main__":
    # host="0.0.0.0" 讓 Windows 端也能連線，不僅限於 WSL 內部
    app.run(host="0.0.0.0", port=5000)
  ```

3. 啟動伺服器
  ```bash
  python app.py
  ```
  > WSL 2 預設會將容器內的服務自動轉發（port forwarding）到 Windows 的 `localhost`，所以大多數情況下不需要額外設定防火牆或連接埠轉發規則，只要在 Flask 啟動時將 `host` 設為 `0.0.0.0`（而非預設的 `127.0.0.1`），Windows 端就能透過 `localhost` 連進來。若仍連不上，可執行 `wsl hostname -I` 查詢 WSL 的 IP，改用該 IP 連線測試。

## 在 WSL 外以瀏覽器查看畫面

伺服器啟動後，直接在的瀏覽器網址列輸入 `http://localhost:5000` 即可看到 Flask 回傳的 `Hello from WSL!` 畫面。整個開發流程中，程式碼與伺服器都跑在 WSL 的 Linux 環境裡，但瀏覽端完全在 Windows 原生環境操作，不需要額外安裝虛擬機的顯示工具或圖形介面轉發設定。

## 開發工具設定：在 VS Code 中直接連進 WSL

因為 WSL 就在 Windows 本機內，VS Code 提供一個更好用、速度更快、且不需要設定 SSH 金鑰的 `WSL`（舊稱 `Remote - WSL`）延伸模組，可以直接在 WSL 的 Linux 檔案系統內開發，同時享有 Windows 原生的 VS Code 操作介面。

### 方法一：安裝 `WSL` 延伸模組（建議）

1. 在 VS Code 中搜尋並安裝微軟官方的 [`WSL` 延伸模組](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl)。
  {% asset_img 4_wsl_extension.png %}
2. 一鍵連入：
    - 方法 A：點擊 VS Code 左下角的＂開啟遠端視窗＂圖示（`><` 符號），選擇＂連線至 WSL＂，接著＂檔案＂ → ＂開啟資料夾...＂選擇專案路徑。
    - 方法 B：直接在 Ubuntu 終端機內，切換到專案資料夾並輸入以下指令， Windows 的 VS Code 就會自動開啟並直接連入該專案路徑。
      ```bash
      code .
      ```
      > 第一次在這個 WSL 環境中執行 `code .` 時， VS Code 會自動幫您建置「遠端連線環境」的過程：
      > 1. 偵測到尚未安裝 VS Code Server
      > 2. 自動下載 Server 執行檔並解壓縮安裝
      > 3. 檢查系統相容性
      > 4. 確認可以正常運作
      > 
      > 之後只要版本沒變，再次執行 `code .` 就不會重複這個安裝流程，會直接連線進去。

### 方法二：透過 Remote - SSH 連線

若想使用傳統的遠端連線方式，也可以安裝 `Remote - SSH` 延伸模組進行連線：

1. 在 Ubuntu 內安裝 SSH 伺服器

    WSL 預設的 SSH 服務並不完整，需要重新安裝：
    ```bash
    sudo apt update
    sudo apt install -y openssh-server
    ```

2. 修改 SSH 設定檔以允許密碼登入

    編輯設定檔（`sudo nano /etc/ssh/sshd_config`），確認或修改以下幾行：
    ```text
    Port 2222    # 建議改用 2222 埠，避免與 Windows 本身的 22 埠衝突
    PasswordAuthentication yes
    ```

3. 重啟 SSH 服務

    ```bash
    sudo service ssh restart
    ```

    > WSL 預設不使用 systemd 管理服務，因此以 `sudo service ssh restart`（而非 `systemctl`）啟動／重啟 SSH 服務。

4. 在 VS Code 中使用 Remote - SSH 連線

    1. 在 Windows 的 VS Code 安裝 `Remote - SSH` 延伸模組。
    2. 點擊左下角的＂開啟遠端視窗＂圖示（`><` 符號），選擇＂連線到主機...＂ → ＂新增 SSH 主機...＂。
    3. 輸入連線指令（因為在同一台電腦，主機名使用 `localhost`）：
      ```text
      ssh <您的 Ubuntu 帳號>@localhost -p 2222
      ```
      完成後，會跳出通知訊息
      {% asset_img 6_remote_ssh_notification.png %}
      > 若中途跳出＂選取要更新的 SSH 設定檔＂，通常預設清單裡會有 `C:\Users\<你的帳號>\.ssh\config` ，選擇這一個使用者層級的即可。選完後 VS Code 會自動把類似以下內容寫入該檔案：
      > ```
      > Host localhost
      >   HostName localhost
      >   User <您的 Ubuntu 帳號>
      >   Port 2222
      > ```
      > 之後就能直接在＂遠端總管＂清單看到這個主機，點擊即可連線，不用每次都重新輸入 `ssh` 指令。
      > {% asset_img 7_wsl_host.png %}
    4. 連線後，依提示輸入密碼即可連入。

### 兩種方式的比較

| 面向 | WSL 延伸模組 | Remote - SSH |
|------|------|------|
| 設定複雜度 | 低（安裝延伸模組即可，免金鑰與密碼設定） | 高（需安裝並設定 SSH 伺服器、修改設定檔、管理連接埠與密碼） |
| 傳輸效能 | 高（透過本機通道直接溝通，不經過網路堆疊） | 較低（需經過 TCP／SSH 加密通道，多一層網路開銷） |
| 檔案權限 | 直接沿用 WSL 內的 Linux 原生權限 | 同樣沿用 Linux 權限，但需額外管理登入密碼或金鑰檔案的存取權限 |

一般情況下，同一台機器內的 WSL 開發優先建議使用官方的 `WSL` 延伸模組；只有在需要連線到其他遠端 Linux 主機時，才需要用到 Remote - SSH。

## 參考資料
- [Developing in WSL](https://code.visualstudio.com/docs/remote/wsl)
