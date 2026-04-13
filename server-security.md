# 主機安全機制設定

適用於 Ubuntu 24.04 + RunCloud + WordPress + nginx + fail2ban 環境。

---

## 一、SSH Key 連線認證

### 為什麼必須使用 SSH Key

| | 密碼登入 | SSH Key |
|--|---------|---------|
| 暴力破解風險 | 高（密碼可猜測） | 幾乎為零（金鑰無法猜測） |
| 自動化攻擊 | 每天有大量機器人嘗試 | 無金鑰即被拒絕，不進入驗證流程 |
| 外洩風險 | 密碼可能被記錄或外洩 | 私鑰存在本機，不在網路傳輸 |
| 管理便利性 | 每次輸入密碼 | 設定後免密碼登入 |

SSH Key 的安全原理：伺服器只存放公鑰，私鑰永遠留在本機。登入時雙方用非對稱加密互相驗證，即使公鑰外洩也無法偽造登入。

### 產生 SSH Key

```bash
ssh-keygen -t ed25519 -C "備註說明"
```

產生兩個檔案：
- `~/.ssh/id_ed25519`（私鑰，絕對不可外流）
- `~/.ssh/id_ed25519.pub`（公鑰，上傳到伺服器）

### 將公鑰部署到伺服器

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```

或手動將公鑰內容貼到伺服器的 `~/.ssh/authorized_keys`。

### 連線指令

```bash
ssh -i ~/.ssh/id_ed25519 -p 22 user@server
```

### 停用密碼登入（強化安全）

確認 SSH Key 能正常登入後，編輯 `/etc/ssh/sshd_config`：

```
PasswordAuthentication no
PubkeyAuthentication yes
```

套用設定：
```bash
sudo systemctl reload sshd
```

> **注意：** 停用密碼登入前，務必確認 SSH Key 登入正常，否則會把自己鎖在主機外。

### 私鑰保管原則
- 私鑰只存在本機，不上傳任何雲端
- 不同主機建議使用不同金鑰對，降低單一金鑰外洩的影響範圍
- 定期檢查 `authorized_keys`，移除不再使用的公鑰

---

## 二、防誤刪機制（trash-cli）

### 安裝
```bash
sudo apt-get install -y trash-cli
```

### 設定 alias（讓 rm 自動走垃圾桶）
```bash
echo "alias rm='trash'" >> ~/.bashrc
source ~/.bashrc
```

### 使用方式

| 指令 | 說明 |
|------|------|
| `rm 檔案` | 移到垃圾桶（可還原） |
| `sudo trash 檔案` | root 檔案移到垃圾桶 |
| `trash-list` | 查看垃圾桶內容 |
| `trash-restore` | 還原檔案 |
| `sudo trash-restore` | 還原 root 擁有的檔案 |
| `trash-empty` | 永久清空垃圾桶 |

### 注意事項
- `alias rm='trash'` 只保護一般使用者，`sudo rm` 不受保護
- 需要 sudo 刪除時，一律改用 `sudo trash 檔案路徑`

---


---

## 三、版本控制（Git）

### 目的

每次修改 WordPress 網站的客製化程式碼後，透過 git 保存變更紀錄，確保任何修改都可追蹤與還原，避免誤改難以復原。

### 追蹤哪些檔案

WordPress 核心、第三方外掛、佈景主題不需要納入版控（它們有自己的更新機制）。
只追蹤**自己開發或客製化的程式碼**：

- 子主題目錄（`functions.php`、`style.css`、自訂模板等）
- 自行開發的外掛

### Monorepo 結構建議

將同一個網站的客製化程式碼集中在單一 repo 管理，用資料夾區分類型：

```
repo-root/
├── themes/
│   └── my-child-theme/
└── plugins/
    ├── my-plugin-a/
    └── my-plugin-b/
```

### 設定 .gitignore（只追蹤指定目錄）

在 `wp-content/` 初始化 git repo，用 `.gitignore` 限制追蹤範圍：

```gitignore
# 忽略所有
*
!.gitignore

# 開放子主題
!themes/
themes/*
!themes/my-child-theme/
!themes/my-child-theme/**

# 開放自製外掛
!plugins/
plugins/*
!plugins/my-plugin-a/
!plugins/my-plugin-a/**
```

### 初始化與推送

```bash
cd /path/to/wp-content
git init
git branch -m main
git remote add origin https://github.com/your-account/your-repo.git
git add -A
git commit -m "initial commit"
git push -u origin main
```

### 修改檔案後提交

```bash
git add -A
git commit -m "描述這次改了什麼"
git push
```

### 還原檔案

```bash
# 查看歷史
git log --oneline

# 還原特定檔案到上一版
git checkout HEAD~1 -- themes/my-child-theme/functions.php

# 查看某個 commit 的變更內容
git show <commit-hash>
```

### 注意事項
- 私人 repo 不要放入任何帳號密碼、API Key、資料庫連線資訊
- `.gitignore` 設定前先確認 `git status` 追蹤的檔案是否符合預期
- 建議每次完成一個功能或修復就 commit 一次，不要累積太多再一起 commit
