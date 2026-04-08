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
