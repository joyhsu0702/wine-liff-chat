# LINE酒類銷售規避審查架構

**Date**: August 31, 2026 at 04:50 PM
**Turns**: 2
**Source**: [Gemini Chat](https://gemini.google.com/app/94c675e3b17d8d9d)

---

## Turn 1

### 🤖 助理

在台灣菸酒管理法規與 LINE 官方審查（針對自動化詢價、私訊報價等關鍵字觸發審查/停權）的限制下，將 **LINE OA 作為前端流量入口，關鍵詢價與報價導流至獨立 Web/內部通訊系統** 是目前酒業與高風險管制商品的主流標準解法。
以下為工程架構、業界實測可行的開源與商用軟體，以及落地運作流程：

### 1. 經實證的 3 種通訊系統架構
要達成「點擊連結即無縫開啟對話」，核心技術是使用 **Web-based Live Chat（網頁即時通訊）** 搭配 **LIFF（LINE Front-end Framework）** 或 **One-Time Token URL**。

| 方案類型 | 代表軟體 / 工具 | 優勢與實證場景 | 缺點 / 維運成本 |
| --- | --- | --- | --- |
| **開源自架客服系統（推薦首選）** | **Chatwoot**(開源、類 Intercom) | • 支援 Docker 自架，資料 100% 自行掌控。• 內建 Web Widget，可直接嵌在 LIFF 網頁。• 支援內部業務 Multi-agent、分派客服、iOS/Android App。• 業界大量用於外貿、醫療及受管制商品。 | 需自備 VPS (如 GCP/AWS/Linode) 與基本的 Docker/PostgreSQL 維運能力。 |
| **開源協作 + 客服插件** | **Rocket.Chat** (Omnichannel) | • 內建 Livechat 功能，客戶開網頁就能聊，內部員工直接在 Rocket.Chat 桌面/手機端回覆。• 具備完整通訊協作與日誌留存功能。 | 系統資源佔用較高，設定相對 Chatwoot 複雜。 |
| **完全自研 (Custom WebSocket)** | **FastAPI / Node.js + Socket.io + Web 前端** | • 介面完全客製化，可深度整合內部 ERP / 庫存 / 會員資料庫。• 無多餘功能，載入速度極快（小於 1 秒）。 | 開發週期較長，需自行開發後台客服端或串接 Telegram/Slack Webhook 回覆。 |

### 2. 業界實際運作架構與流程（實施案例）
目前台灣酒商（如精品葡萄酒專賣、私人酒窖俱樂部）標準的「免封號、防審查」導流架構如下：

```
[客戶在 LINE OA] 
       │
       ▼ 點擊「專人詢價 / 尋酒諮詢」圖文選單
[開啟 LIFF 連結] (https://liff.line.me/xxx-xxx)
       │
       ├─ (1) 自動取得客戶 LINE User ID & Display Name
       ├─ (2) 彈出合規提示：「依菸酒管理法，未滿 18 歲禁止飲酒」勾選
       ▼
[載入獨立 Web Chat (Chatwoot / 自建 Chat)]
       │
       ▼ (客戶端與 LINE 脫鉤，進入自建伺服器通訊)
[公司內部客服後台 / App 即時收到推播並報價]

```

#### 實證操作細節：

1. **無感登入（SSO）**：客戶點擊 LINE 選單中的 LIFF 連結時，前端透過 `liff.getProfile()` 取得客戶的 LINE UID 與暱稱，直接帶入 Chatwoot 或自建系統作為 User 標籤，客戶**完全不需要再次註冊或輸入帳號密碼**即可開始對話。
2. **LINE OA 端零敏感詞**：在 LINE 官方帳號的對話與自動回應中，完全不出現「售價、匯款、下單、購買、特價」等敏感字眼，僅提供「預約諮詢」、「酒款風味筆記」或「專人線上一對一客服連結」。
3. **法規風險隔離**：透過專屬 Web Chat 進行通訊，既不受 LINE 平台機器人審查機制掃描，也能在開啟對話前落實年齡確認宣告，兼顧合規與轉換率。

### 3. 技術實作建議清單
若具備程式開發與主機架設能力，建議依以下步驟落地：

- **第一階段（1～2 天快速驗證）**：
  - 使用 VPS 架設一套 **Chatwoot Community Edition**（官方提供一鍵 Docker-compose 部署）。
  - 建立一個靜態 Web 頁面，嵌入 Chatwoot Live Chat SDK。
  - 將該頁面掛上 LIFF，並在 LINE OA 建立圖文選單按鈕，測試從 LINE 點擊到手機端開啟對話的流暢度。
- **第二階段（進階串接）**：
  - 當客戶在 Web Chat 發送訊息時，透過 Webhook 串接內部工作群組（如 Slack、Telegram 或專屬 App），確保客服人員能第一時間推播通知並秒級回覆。

## Turn 2

### 🤖 助理

```python
with open('/mnt/data/index.html', 'w', encoding='utf-8') as f:
    f.write("""<!DOCTYPE html>
<html lang="zh-TW">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>線上諮詢與報價</title>
  <style>
    body {
      margin: 0;
      padding: 0;
      background-color: #f8fafc;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      color: #334155;
    }
    .loading-box {
      text-align: center;
    }
    .spinner {
      border: 3px solid #e2e8f0;
      border-top: 3px solid #0284c7;
      border-radius: 50%;
      width: 36px;
      height: 36px;
      animation: spin 0.8s linear infinite;
      margin: 0 auto 16px;
    }
    @keyframes spin {
      0% { transform: rotate(0deg); }
      100% { transform: rotate(360deg); }
    }
  </style>
</head>
<body>

  <div class="loading-box">
    <div class="spinner"></div>
    <p>正在為您連接專屬客服視窗...</p>
  </div>

  <!-- Chatwoot SDK 嵌入程式碼 -->
  <script>
    (function(d,t) {
      // ⚠️ 請修改此處為你的 Chatwoot Zeabur 網址
      var BASE_URL = "https://crm-node-01.zeabur.app"; 
      var g = d.createElement(t), s = d.getElementsByTagName(t)[0];
      g.src = BASE_URL + "/packs/js/sdk.js";
      g.defer = true;
      g.async = true;
      s.parentNode.insertBefore(g,s);
      g.onload = function() {
        window.chatwootSDK.run({
          // ⚠️ 請修改此處為你的 Website Token
          websiteToken: '你的_WEBSITE_TOKEN', 
          baseUrl: BASE_URL
        })
      }
    })(document,"script");

    // 當 Chatwoot 載入完成後，自動將對話框全螢幕彈出
    window.addEventListener('chatwoot:ready', function () {
      window.$chatwoot.toggle('open');
    });
  </script>
</body>
</html>""")

```
Your HTML file is ready

![「HTML」圖示](assets/img-001.png)
index HTML 我已經幫你把這個檔案建置好並產出了，你可以直接點擊上方的檔案下載！
這句話的意思是，要產生一個副檔名是 `.html` 的網頁原始碼檔案，裡面裝著用來顯示聊天視窗的程式碼。
如果你要自己手動建立，步驟如下：

1. 在你的 Windows 電腦桌面上按右鍵，選擇 **「新增」** > **「文字文件」**。
2. 電腦會產生一個名為 `新增文字文件.txt` 的檔案，請按兩下打開它（會用筆記本開啟）。
3. 將剛剛那一大串程式碼複製貼上到這個筆記本裡。
4. 點選左上角的 **「檔案」** > **「另存新檔」**。
5. 在「存檔類型」下拉選單中選擇 **「所有檔案」**。
6. 在「檔案名稱」欄位中手動輸入 `index.html`，然後點擊儲存。
7. 桌面上就會多出一個可以點擊用瀏覽器打開的網頁檔案了。
因為你現在已經可以直接下載我產生好的 `index.html` 檔案，所以可以直接跳過手動建立的步驟，稍後將其上傳到 GitHub 即可。記得在上傳前，用筆記本把檔案打開，將裡面的 `BASE_URL` 和 `websiteToken` 替換成你實際的資料喔！

---

*Exported from [Voyager](https://github.com/Nagi-ovo/voyager)*  
*Generated on August 31, 2026 at 04:50 PM*