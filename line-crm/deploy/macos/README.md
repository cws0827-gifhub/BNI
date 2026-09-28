# Mac mini ＋ Synology 部署指南

```
客人 LINE → LINE 平台 → Cloudflare Tunnel → Mac mini :8765 /callback
                                              │  （程式＋資料庫都在 Mac mini）
                    諮詢師在診所內網開管理頁 ─┘
                                              │ 每天 03:15 備份
                                              ▼
                                   Synology /backup/line-crm（保留 30 天）
                                              │ Hyper Backup
                                              ▼
                                   異地：C2 雲端 或 外接 USB 硬碟
```

**分工原則**：Mac mini 負責執行程式，Synology 只放備份。
⚠️ 不要把資料庫直接放在 NAS 共用資料夾、讓程式透過網路讀寫：SQLite 經由 SMB 存取很容易損毀。

---

## 1. Synology 準備（DSM 控制台）

1. **控制台 → 共用資料夾 → 新增**：名稱 `backup`（已有備份用資料夾就沿用），在裡面建立 `line-crm` 資料夾。
2. **控制台 → 使用者帳號**：建一個專用帳號，例如 `macmini-backup`，只給 `backup` 的讀寫權限。
3. **控制台 → 檔案服務 → SMB**：確認已啟用。
4. **強烈建議**：用 **Hyper Backup** 把 `backup/line-crm` 再備份一份到 Synology C2 或外接硬碟。
   NAS 本身壞掉或遇到勒索病毒時，這一份才救得回來。
5. 選配：**Snapshot Replication** 對 `backup` 開每日快照，可防止勒索病毒加密備份檔。

## 2. Mac mini 掛載 NAS 共用資料夾（只需做一次）

1. Finder → 前往 → 連接伺服器（⌘K）→ 輸入 `smb://DiskStation.local/backup`
   （`DiskStation` 換成你的 NAS 名稱，或直接輸入 IP，例如 `smb://192.168.1.10/backup`）
2. 用 `macmini-backup` 帳號登入，**勾選「在我的鑰匙圈中記住此密碼」**。
3. 系統設定 → 一般 → 登入項目：把掛載後的 `backup` 拖進去，開機就會自動掛載。
4. 萬一沒有掛上，備份腳本也會用 `.env` 裡的 `NAS_SMB_URL` 自動嘗試掛載。

## 3. Mac mini 系統設定

你的 Mac mini 已經在跑排程，下面幾項應該都設好了，確認一下即可：
- **系統設定 → 能源**：關閉「顯示器關閉時自動睡眠」，並開啟「停電後自動啟動」。
- **系統設定 → 使用者與群組 → 自動登入**：要開啟。本程式屬於「使用者層級」的排程，要登入後才會執行，和你現有的排程一樣。
- 確認 8765 埠沒被佔用：`lsof -i :8765` 沒有輸出就代表可以用。
- 備份時間預設在凌晨 **03:15**。若和現有排程撞時段，改 `install.sh` 裡的 `Hour`／`Minute` 後重跑即可。

## 4. 安裝程式

```bash
cd ~          # 或你想放的位置
git clone https://github.com/cws0827-gifhub/BNI.git
cd BNI/line-crm
cp .env.example .env
open -e .env  # 填入 LINE 金鑰、管理頁密碼、NAS_BACKUP_DIR
bash deploy/macos/install.sh
```

`.env` 裡的 `NAS_BACKUP_DIR` 要填 **Finder 掛載後的實際路徑**，通常是 `/Volumes/backup/line-crm`。

完成後：
- 管理頁：`http://Mac名稱.local:8765`（診所內任何一台電腦或手機，連同一個 Wi-Fi 就能開）
- 先手動跑一次備份，確認 NAS 上出現檔案：`bash deploy/macos/backup.sh`

## 5. 讓 LINE 連得進來：Cloudflare Tunnel（建議）

不用在路由器開 port，自動提供 HTTPS，而且**只開放 `/callback`，管理頁不會出現在網路上**。
網域使用 **beauty-keys.com**，DNS 需要交給 Cloudflare 管理（免費方案就夠用），做法見 [`../cloudflare-relay/README.md`](../cloudflare-relay/README.md) 第 1 步。

```bash
brew install cloudflared
cloudflared tunnel login                       # 瀏覽器選 beauty-keys.com
cloudflared tunnel create line-crm             # 記下 Tunnel ID
cloudflared tunnel route dns line-crm line.beauty-keys.com
cp deploy/macos/cloudflared-config.example.yml ~/.cloudflared/config.yml
open -e ~/.cloudflared/config.yml              # 填入 Tunnel ID、使用者名稱
sudo cloudflared service install               # 開機自動啟動
```

⚠️ **LINE 的 Webhook 目前接在領健，不要直接改成 `https://line.beauty-keys.com/callback`**，否則領健會收不到訊息。
要讓兩邊都收到，請看 [`../cloudflare-relay/README.md`](../cloudflare-relay/README.md)。

## 6. 在診所外看管理頁（選配）

在 Mac mini 和手機上都安裝 **Tailscale**（Synology 套件中心也有），就能在外面安全地打開 `http://Mac名稱:8765`，
不需要把管理頁公開到網路上。

---

## 日常維護與疑難排解

| 狀況 | 怎麼做 |
|---|---|
| 看程式紀錄 | `tail -f ~/Library/Logs/line-crm/com.meizhiyao.line-crm.log` |
| 看備份紀錄 | `tail ~/Library/Logs/line-crm/com.meizhiyao.line-crm-backup.log` |
| 更新程式 | `git pull && bash deploy/macos/install.sh` |
| 重新啟動 | `launchctl kickstart -k gui/$(id -u)/com.meizhiyao.line-crm` |
| 備份出現 `Operation not permitted` | 系統設定 → 隱私權與安全性 → 完整磁碟取用權限，加入 `/bin/bash` |
| 備份出現「找不到 NAS 資料夾」 | NAS 沒開機，或共用資料夾沒掛載，回到第 2 步 |

**還原資料**（Mac mini 壞掉、換新機時）：

```bash
launchctl bootout gui/$(id -u)/com.meizhiyao.line-crm
gunzip -c /Volumes/backup/line-crm/line_crm-日期.db.gz > line_crm.db
bash deploy/macos/install.sh
```
