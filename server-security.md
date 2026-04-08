# 主機安全機制設定

適用於 Ubuntu 24.04 + RunCloud + WordPress + nginx + fail2ban 環境。

---

## 一、防誤刪機制（trash-cli）

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
- webapps 目錄（`/home/mooga/webapps/`）由 mooga 使用者擁有，alias 有效
- log 與設定檔（`/etc/`、`/home/mooga/logs/nginx/`）由 root 擁有，需用 `sudo trash`

---

## 二、WordPress 排程準時發佈（系統 cron）

WordPress 預設靠訪客觸發 wp-cron，無訪客時排程會延遲。

### 停用訪客觸發
在 `wp-config.php` 加入：
```php
define('DISABLE_WP_CRON', true);
```

### 改用系統 cron（每分鐘執行）
```bash
crontab -e
```
加入：
```
* * * * * cd /home/mooga/webapps/{網站名稱} && wp cron event run --due-now --quiet 2>/dev/null
```

### 效果
- 排程發佈最多延遲 1 分鐘，不依賴訪客
- 減少前端請求負擔

---

## 三、fail2ban 防護

### 現有 jail 一覽

| Jail | 觸發條件 | 封鎖時間 |
|------|---------|---------|
| `nginx-ddos` | 3秒 ≥ 300 次動態請求 | 24 小時 |
| `nginx-xmlrpc` | 60秒 ≥ 5 次 POST xmlrpc.php | 7 天 |
| `nginx-wp-login` | 60秒 ≥ 5 次 wp-login.php | 7 天 |
| `nginx-author-scan` | 60秒 ≥ 3 次 `?author=` | 7 天 |
| `nginx-404-scan` | 60秒 ≥ 20 次 404 | 24 小時 |

### 常用指令
```bash
# 查看封鎖狀態
sudo fail2ban-client status nginx-ddos

# 手動解封
sudo fail2ban-client set nginx-ddos unbanip <IP>

# 手動封鎖
sudo fail2ban-client set nginx-ddos banip <IP>
```

### 白名單設定
編輯 `/etc/fail2ban/jail.local`：
```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 <你的固定IP> 172.104.117.27
```

### 注意事項
- 所有 nginx jail 必須加 `backend = polling` 才會讀取 log 檔
- nginx-ddos filter 已設定排除靜態資源（CSS/JS/圖片），只計算動態請求
- 修改設定後執行 `sudo systemctl restart fail2ban`
