# 主機安全機制設定

適用於 Ubuntu 24.04 + RunCloud + WordPress + nginx + fail2ban 環境。

---

## 防誤刪機制（trash-cli）

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
