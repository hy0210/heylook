# 專案規劃：欸你看（HeyLook）— 朋友間的作品分享平台

## 【專案概述】

「欸你看」是一個讓朋友之間分享「最近看了什麼」的跨媒體平台，涵蓋書籍、電影、多集影劇、漫畫、動畫。

**核心使用情境**：註冊登入後，任何人都可以：
- 新增或編輯媒體資料（維基式共同維護，如同 IMDb／維基百科的協作模式）
- 針對某部作品留下自己的心得與星級評分
- 對別人的心得按愛心，並收到「誰愛了你的心得」的通知

**建立媒體必須同時附上第一篇心得**：新增一筆媒體資料時，強制要求同時填寫心得內容與星級評分，一次 API 呼叫建立兩筆資料。會建立媒體資料的人，必然是因為看過這部作品才想記錄它，讓他順手留下第一手心得，也避免平台上出現「有作品資料但沒人給過意見」的空殼條目。

**帳號系統**：不使用第三方登入（Google／Facebook 等），由平台自行管理帳號密碼；註冊採**二階段 email 驗證**——填寫資料後先寄送驗證碼，輸入正確驗證碼才完成註冊，避免使用假 email 或他人 email 濫用註冊。詳見文末【帳號系統：註冊／登入／登出設計】。

**登入即是完整功能**：所有會員權限一致，沒有角色分級，降低平台的參與門檻。

**產品重心**：新增媒體與發表心得是最高頻的核心行為（詳見文末【下一階段規劃】）。

---

## 【角色設計：簡化為兩層】

| 角色 | 說明 |
|------|------|
| **一般會員** | 註冊登入即可：建立/編輯媒體資料、留心得 |
| **管理員** | 唯一的特殊角色，僅負責下架不當內容（心得、媒體資料），不負責審核升級（因為沒有角色可升） |

**權限判斷只有兩個維度**：
1. 是否已登入（決定能否寫入）
2. 是否為管理員（決定能否下架他人內容）

---

## 【核心設計決策：媒體資料共編 vs 心得私有】

本專案最重要的資料治理邏輯，體現「不同內容類型該有不同的擁有權模型」的思考：

| 內容類型 | 擁有權模型 | 原因 |
|---------|-----------|------|
| **媒體資料**（書/電影/劇/漫畫/動畫的基本資訊） | **共同編輯**（類似維基百科） | 這是「客觀事實」（片名、年份、大綱），任何人發現資料不完整或有誤都該能修正，避免同一部作品被重複建立 |
| **心得** | **僅本人可編輯/刪除**（owner-scoped） | 這是「主觀意見」，代表個人觀點，他人不該能竄改 |

媒體資料雖然開放共編，但**保留完整的編輯歷史脈絡**：每筆媒體資料記錄「最後編輯者」與「最後編輯時間」，讓使用者知道資料是誰、何時更新的，兼顧開放協作與可追溯性。

---

## 【資料實體設計】

```
users
  id, email, password_hash, nickname, avatar_url, bio, role,
  email_verified_at,     ← 完成 email 驗證的時間，NULL 代表尚未驗證
  created_at, updated_at
  
  role: 'member' | 'admin'
  bio: 自我介紹，選填，純文字（不支援 Markdown），有長度上限（例如 200 字）
  ⚠️ nickname 全站唯一（unique constraint），註冊與編輯個人主頁時都需檢查

email_verifications  (註冊時的二階段驗證碼) ⭐
  id, email, password_hash, nickname,   ← 註冊表單的暫存資料，驗證成功才寫入 users
      verification_code,                ← 6 位數驗證碼
      expires_at,                       ← 驗證碼有效期限（例如 10 分鐘）
      created_at
  
  ⚠️ 這張表儲存「尚未完成驗證的註冊申請」，驗證成功後才真正寫入 users 表並刪除此筆暫存資料
  ⚠️ 驗證碼過期或驗證失敗次數過多需重新申請（不做無限重試）

media  (跨五種類型統一管理，任何登入會員皆可建立/編輯)
  id, title, type, genre,
  format,                            ← 收看/閱讀平台
  cover_image_url,                   ← 上傳的封面圖
  synopsis,                          ← 大綱介紹
  creator_name,                      ← 作者/導演名稱
  release_year,
  created_by,                        ← 最初建立者（保留紀錄用途，非權限依據）
  last_edited_by,                    ← 最後編輯者 ⭐
  last_edited_at,                    ← 最後編輯時間 ⭐
  status, created_at, updated_at
  
  type:   'book' | 'movie' | 'series' | 'comic' | 'animation'
  format: 書/漫畫 → 'physical' | 'ebook'
          電影/影集/動畫 → 'streaming' | 'theater'
  status: 'published' | 'removed'    ← 管理員下架機制

media_edit_history  (媒體編輯歷史，記錄每次異動) ⭐
  id, media_id, edited_by, changed_fields (jsonb), edited_at
  
  ⚠️ 這張表讓「最後編輯者」有完整歷史可查，
     不只是覆蓋式記錄，展示稍微進階的資料異動追蹤設計

reviews  (會員針對某媒體發表的心得，僅本人可編輯/刪除)
  id, media_id, user_id, content, rating (1-5),
  status, created_at, updated_at
  
  status: 'published' | 'removed'    ← 管理員下架機制
  ⚠️ 一個會員對同一媒體只能有一篇心得（unique(media_id, user_id)）

review_likes  (心得的愛心紀錄) ⭐
  id, review_id, user_id, created_at
  
  ⚠️ 一個人對同一則心得只能按一次愛心（unique(review_id, user_id)）
  ⚠️ 不能對自己的心得按愛心（寫入時驗證 review.user_id !== req.user.id）

notifications  (愛心通知) ⭐
  id, recipient_id,      ← 通知要給誰看（心得的原作者）
      actor_id,          ← 誰按的愛心
      review_id,         ← 被按愛心的心得
      media_id,          ← 冗餘欄位，省去通知列表時多一次 join 查詢媒體資訊
      created_at
  
  ⚠️ 每次按愛心都新增一筆獨立通知，同一則心得被多人按愛心會產生多筆通知，不合併
  ⚠️ 沒有 is_read 欄位，通知頁單純顯示最近 N 筆列表，不做已讀/未讀狀態
```

**共 7 個實體**：User, EmailVerification, Media, MediaEditHistory, Review, ReviewLike, Notification

---

## 【媒體欄位設計】

| 欄位 | 說明 | 適用類型 |
|------|------|---------|
| `type` | book / movie / series / comic / animation | 全部 |
| `format` | 書/漫畫：`physical`（紙本）／`ebook`（電子版）<br>電影/影集/動畫：`streaming`（串流）／`theater`（院線） | 依類型顯示對應選項 |
| `cover_image_url` | 上傳的封面圖 | 全部 |
| `synopsis` | 大綱介紹 | 全部 |
| `creator_name` | 作者／導演／原作 | 全部 |
| `last_edited_by` / `last_edited_at` | 最後編輯者與時間，前端顯示為「上次更新：小明・3 天前」 | 全部 |

多集影劇（劇集/動畫/漫畫）**不分集管理**，整部作品視為一筆 media 資料。

---

## 【API 端點設計】

### 路由檔案結構
```
routes/
  ├── healthcheck.js
  ├── auth.js              (註冊/登入)
  ├── users.js             (個人資料)
  ├── media.js             (媒體 CRUD，公開查詢 + 登入即可寫入)
  ├── reviews.js           (心得 CRUD，owner-scoped，含愛心)
  ├── notifications.js     (愛心通知)
  └── admin.js             (管理員：下架內容)
```

### 1. `auth.js` / `users.js`
```
POST   /api/auth/signup                    提交註冊資料，發送驗證碼（見下方詳細說明）
POST   /api/auth/signup/verify             輸入驗證碼，完成註冊並自動登入
POST   /api/auth/signup/resend             驗證碼過期或未收到，要求重新發送
POST   /api/auth/login                     登入
POST   /api/auth/logout                    登出（需登入）

GET    /api/users/profile                  查看自己的個資（需登入）
PUT    /api/users/profile                  編輯個人主頁：暱稱、頭像、自我介紹（需登入）
GET    /api/users/check-nickname           即時檢查暱稱是否已被使用（公開，供註冊與編輯個人主頁的前端即時驗證）
PUT    /api/users/password                 修改密碼（需登入）

GET    /api/users/:userId                  查看他人公開主頁（公開，含貢獻總覽）
GET    /api/users/:userId/media-edits      查看他編輯過的媒體資料列表（公開）
GET    /api/users/:userId/reviews          查看他發表過的所有心得（公開）
```

**小型社交平台的核心設計**：`GET /api/users/:userId` 是個人主頁的骨幹端點，彙整這位成員在平台上的所有活動軌跡——不只是「他是誰」，而是「他在這裡做過什麼」。這對應到一個小圈子分享平台的本質：**認識朋友的品味，比認識朋友的自我介紹更重要**。

**個人主頁編輯**：`PUT /api/users/profile` 以 `multipart/form-data` 接受 `nickname`、`bio`（選填）與 `avatar` 檔案，後端驗證暱稱唯一性（更新前排除自己目前的暱稱，允許「不改暱稱、只改頭像或自我介紹」的情況通過檢查）；頭像圖片同樣經由 `multer` 處理，流程與媒體封面圖上傳一致。`bio` 為選填欄位，未填寫時為空字串或 NULL，前端個人主頁不顯示該區塊。

### 2. `media.js`
```
GET    /api/media                          媒體庫列表（公開，可篩選 type/genre/format，支援 sort=latest 依建立時間倒序）
GET    /api/media/:mediaId                 媒體詳情（公開，含心得列表、編輯紀錄）
GET    /api/media/:mediaId/history         查看編輯歷史（公開）

POST   /api/media                          建立媒體資料＋第一篇心得（需登入，一次請求同時寫入 media 與 review 兩筆資料）
PUT    /api/media/:mediaId                 編輯媒體資料（需登入，任何會員皆可，寫入 last_edited_by/at + history）
```

**首頁「最新建立作品」區塊**直接呼叫 `GET /api/media?sort=latest&limit=5`，不需要額外的聚合端點。

**`POST /api/media` 的請求格式**（media 欄位與 review 欄位混合在同一個 multipart/form-data 請求中）：
```
fields: title, type, genre, format, synopsis, creator_name, release_year,
        review_content, review_rating   ← 心得內容與星級，皆為必填
file:   cover_image
```
後端在同一個資料庫 transaction 內依序寫入 `media` 與 `review`，任一步驟失敗則整筆回滾，避免出現「有作品資料但沒有心得」的中間狀態。

### 3. `reviews.js`
```
GET    /api/reviews/latest                 全站最新心得列表（公開，依發表時間倒序，供首頁使用）
GET    /api/media/:mediaId/reviews         查看某媒體的所有心得（公開，含每則心得的愛心數）

POST   /api/media/:mediaId/reviews         發表心得＋評分（需登入；若該媒體剛由自己建立，此端點不會再被呼叫，因為心得已隨 POST /api/media 一併建立）
PUT    /api/reviews/:reviewId              編輯自己的心得（需登入，owner-scoped）
DELETE /api/reviews/:reviewId              刪除自己的心得（需登入，owner-scoped）

POST   /api/reviews/:reviewId/like         對一則心得按愛心（需登入，不可對自己的心得按讚，寫入 review_likes 並產生一筆 notification）
DELETE /api/reviews/:reviewId/like         取消愛心（需登入，owner-scoped：只能取消自己按過的愛心）
GET    /api/reviews/:reviewId/likes        查看這則心得被哪些人按過愛心（公開，回傳按讚者的暱稱列表）
```

**首頁「最新發表心得」區塊**呼叫 `GET /api/reviews/latest?limit=5`，與媒體列表各自獨立查詢，不做跨資料表的合併排序——兩個區塊各自對應一支簡單的列表查詢即可。

**愛心設計說明**：
- 按愛心與取消愛心是兩個獨立端點（`POST`／`DELETE`），對應前端「愛心圖示的開／關切換」互動
- 按愛心的當下，後端在同一個 transaction 內寫入 `review_likes` 並新增一筆 `notifications`，通知心得的原作者
- 自己不能對自己的心得按愛心：`POST` 端點驗證 `review.user_id !== req.user.id`，否則回傳 403
- `GET /api/reviews/:reviewId/likes` 是「點進愛心數看到是誰按過的」這個互動的資料來源，公開端點無需登入即可查看

### 4. `notifications.js`
```
GET    /api/notifications                  查看我收到的通知列表（需登入，依時間倒序，僅限自己的通知）
```

**設計說明**：這支端點是「主頁右上角通知」的資料來源。回傳格式包含足夠的資訊讓前端直接渲染「Alice 愛了你在《鬼滅之刃》的心得」並點擊後導向 `/media/:mediaId#review-:reviewId`：
```json
[
  {
    "id": "...",
    "actor": { "id": "...", "nickname": "Alice" },
    "review_id": "...",
    "media": { "id": "...", "title": "鬼滅之刃" },
    "created_at": "..."
  },
  ...
]
```
沒有已讀/未讀狀態，單純顯示最近 N 筆通知；同一則心得被多人按愛心會各自產生一筆通知，不合併顯示。

### 5. `admin.js`（僅管理員）
```
DELETE /api/admin/media/:mediaId                    下架媒體資料
DELETE /api/admin/reviews/:reviewId                 下架心得
```

**合計約 26 個端點**，聚焦「帳號系統（含二階段驗證）」、「共編媒體資料」、「owner-scoped 心得」、「輕量社交互動（愛心＋通知）」四套核心邏輯。

---

## 【中介層設計】

```
middleware/
  ├── auth.js       驗證 JWT，掛載 req.user
  └── admin.js      驗證 req.user.role 是否為 'admin'
```

```javascript
// admin.js
export default function admin(req, res, next) {
  if (req.user.role !== 'admin') {
    return next(new AppError(403, '此操作需要管理員權限'));
  }
  next();
}
```

**Owner-scoped 驗證**（僅套用在心得，媒體資料不套用）：
```javascript
// 範例：編輯心得
if (review.user_id !== req.user.id) {
  return next(new AppError(403, '你沒有權限編輯這則心得'));
}

// 範例：編輯媒體資料（任何登入會員皆可，僅記錄編輯者）
media.last_edited_by = req.user.id;
media.last_edited_at = new Date();
await MediaEditHistoryRepo.save({ media_id, edited_by: req.user.id, changed_fields });

// 範例：對心得按愛心（不可對自己的心得按讚）
if (review.user_id === req.user.id) {
  return next(new AppError(403, '不能對自己的心得按愛心'));
}
await ReviewLikeRepo.save({ review_id, user_id: req.user.id });
await NotificationRepo.save({
  recipient_id: review.user_id,
  actor_id: req.user.id,
  review_id,
  media_id: review.media_id,
});
```

---

## 【圖片上傳設計】

建立媒體資料時可上傳封面圖：
```
POST /api/media
Content-Type: multipart/form-data

fields: title, type, genre, format, synopsis, creator_name, release_year
file:   cover_image
```

後端可用 `multer` 處理上傳，圖片存放於本地 `uploads/` 目錄（開發階段）或串接雲端物件儲存（如 Cloudinary / S3，視你想展示的技能深度決定）。頭像上傳（`PUT /api/users/profile`）沿用同一套 multer 設定。

---

## 【帳號系統：註冊／登入／登出設計】

不使用第三方登入（無 Google／Facebook OAuth），全部帳密由平台自行管理，採**二階段 email 驗證**——驗證碼進行式（非信件連結型），流程分兩步：

### 流程一：註冊

```
Step 1: POST /api/auth/signup
  body: { email, password, nickname }
  
  後端動作：
  1. 檢查 email 是否已註冊過（users 表）、nickname 是否已被使用
  2. 產生 6 位數隨機驗證碼，寫入 email_verifications（暫存，非正式帳號）
  3. 寄送驗證碼到該 email（開發環境可先印在 console 或回傳於 API response 方便測試）
  4. 回傳：{ message: "驗證碼已寄出，請於 10 分鐘內完成驗證" }

Step 2: POST /api/auth/signup/verify
  body: { email, verification_code }
  
  後端動作：
  1. 查 email_verifications，比對驗證碼是否正確、是否過期
  2. 正確 → 將暫存資料正式寫入 users 表（email_verified_at 設為現在時間），刪除 email_verifications 該筆暫存
  3. 簽發 JWT，回傳 { token, user }（驗證成功即自動登入，不需再手動登入一次）
  4. 錯誤 → 回傳 400，前端顯示「驗證碼錯誤或已過期」

（若使用者沒收到或驗證碼過期）
POST /api/auth/signup/resend
  body: { email }
  → 重新產生驗證碼並寄送，舊驗證碼失效
```

驗證碼進行式的前端是同一頁面的兩步驟表單、體驗連貫；開發階段可以先不整合真正的寄信服務（用 console.log 印出驗證碼），先把整條驗證邏輯做完整，之後再串接 Nodemailer 等寄信套件。

### 流程二：登入 / 登出

```
POST /api/auth/login
  body: { email, password }
  
  後端動作：
  1. 查 users 表比對 email + password_hash（bcrypt 驗證）
  2. 若 email_verified_at 為 NULL（理論上不該發生，因為未驗證不會有 users 紀錄，此檢查是防禦性寫法）→ 拒絕登入
  3. 簽發 JWT，回傳 { token, user }

POST /api/auth/logout
  需登入（帶 JWT）
  
  後端動作：JWT 是無狀態的，後端不需要維護 session，登出主要是前端行為
  （清除本地儲存的 token），此端點存在是為了語意完整與未來可能的黑名單機制擴充，
  MVP 階段可以只回傳 200 OK，不做額外處理
```

### 暱稱唯一性檢查

```
GET /api/users/check-nickname?nickname=xxx
  回傳：{ available: true | false }
```
用於：
1. 註冊表單填寫暱稱時即時檢查（debounce 觸發）
2. 編輯個人主頁修改暱稱時即時檢查（排除自己目前使用的暱稱）

---

## 【專案結構】

```
heylook/
├── backend/
│   ├── entities/
│   │   ├── User.js
│   │   ├── EmailVerification.js
│   │   ├── Media.js
│   │   ├── MediaEditHistory.js
│   │   ├── Review.js
│   │   ├── ReviewLike.js
│   │   └── Notification.js
│   ├── routes/
│   │   ├── healthcheck.js
│   │   ├── auth.js
│   │   ├── users.js
│   │   ├── media.js
│   │   ├── reviews.js
│   │   ├── notifications.js
│   │   └── admin.js
│   ├── controllers/
│   │   （對應各路由）
│   ├── services/
│   │   └── mailer.js         (寄送驗證碼信件，開發階段可先印在 console)
│   ├── middleware/
│   │   ├── auth.js
│   │   └── admin.js
│   ├── uploads/              (圖片上傳暫存，含頭像與媒體封面圖)
│   ├── db/
│   │   └── data-source.js
│   ├── app.js
│   ├── server.js
│   ├── Dockerfile
│   └── package.json
├── frontend/
│   └── src/pages/
│       ├── public/           (媒體庫、心得閱讀)
│       ├── auth/             (註冊/驗證碼/登入)
│       ├── member/           (建立/編輯媒體、發表心得、通知、個人主頁編輯)
│       └── admin/            (下架內容)
├── test/
│   ├── auth.test.js
│   ├── media.test.js
│   ├── reviews.test.js
│   ├── notifications.test.js
│   ├── admin.test.js
│   └── smoke.test.js
├── docker-compose.yml
└── package.json
```

---

## 【核心後端能力展示重點】

1. **差異化擁有權模型**：同一個系統內，媒體資料共編、心得 owner-scoped，展示「依內容性質設計不同治理規則」的判斷力，而非套用單一權限模板
2. **編輯歷史追蹤**：`media_edit_history` 記錄每次異動與異動欄位（JSONB），比單純的 `updated_at` 更進階
3. **軟性下架機制**：media/reviews 皆有 status 欄位，管理員下架非硬刪除，保留資料完整性
4. **檔案上傳處理**：multer 圖片上傳，展示處理 multipart/form-data 的能力
5. **唯一性約束設計**：一個會員對同一媒體只能發表一篇心得（unique constraint）
6. **多型別欄位邏輯**：`format` 欄位依 `type` 不同而有不同合法值（前後端皆需驗證）
7. **輕量管理員機制**：不做複雜的角色審核流程，僅聚焦在「下架不當內容」這個最小必要的治理功能
8. **跨表 transaction 設計**：`POST /api/media` 在同一個 transaction 內建立 media 與 review 兩筆資料，展示「一個使用者動作對應多筆資料寫入」時如何確保資料一致性
9. **社交互動與通知系統**：愛心功能（唯一性約束、自我限制驗證）與通知系統（寫入時機、資料反正規化設計），展示從單純 CRUD 延伸到社交型資料模型的能力
10. **反正規化設計取捨**：`notifications.media_id` 是刻意反正規化的欄位（可從 review 反查得到），用來換取通知列表查詢時少一次 join，體現對讀取效能與資料正規化之間取捨的判斷
11. **二階段驗證流程設計**：註冊資料先暫存於 `email_verifications`、驗證成功才正式寫入 `users`，避免未驗證的髒資料污染正式帳號表，展示「暫存 → 確認 → 正式寫入」這類多步驟流程的資料模型設計能力
12. **暱稱唯一性的雙重使用場景**：`GET /api/users/check-nickname` 同時服務註冊與編輯個人主頁兩種情境，且編輯情境需要「排除自己目前的暱稱」這個額外判斷，是唯一性驗證中容易被忽略的邊界情況

---

## 【下一階段規劃（暫緩，先不實作）】

- **邀請制小圈圈**（類似 FB 社團）：以「圈子」為單位聚集朋友，僅受邀者能加入與瀏覽圈內內容，需要 `circles`（圈子基本資料）、`circle_invitations`（邀請連結／邀請碼，含使用次數與有效期限）、`circle_members`（成員名單與角色，如圈主／一般成員）等表；媒體、心得、通知等既有功能屆時可能需要加上「所屬圈子」的範疇限制，是對現有資料模型影響最大的一項擴充，值得獨立評估
- **策展主題**（Curated List）：會員可將喜歡的作品整理成主題清單推薦給朋友（如「2024 科幻片單」），owner-scoped，僅本人可編輯/刪除；需要 `curated_lists`、`curated_list_items` 兩張表與對應的 CRUD 端點，首頁屆時可再加回第三個區塊
- **個人待看清單**（Watchlist）：記錄自己想看/已看的作品，含 pending/done 狀態
- **擴充媒體來源類型**：新增 podcast、YouTube 影片／頻道等內容形式，納入 `media.type` 的可選值（例如 `podcast`、`youtube_video`）；YouTube／podcast 內容可能需要額外欄位（如節目集數、原始連結、頻道名稱），屆時視實際需求擴充 `media` 表或另立子表
- 可能的社群延伸：追蹤朋友、留言互動

---

## 【開發順序建議】

```
1. 帳號系統（auth + users，含二階段 email 驗證）→ 驗證：註冊流程兩步驟正確、驗證碼過期/錯誤處理正確、登入/登出正常、暱稱唯一性檢查正確
2. 個人主頁編輯（users/profile，含頭像上傳）→ 驗證：暱稱重複時擋下、允許不改暱稱只改頭像、multer 正確處理頭像檔案
3. 媒體＋首篇心得建立（media，一次 transaction）→ 驗證：兩筆資料同時成功寫入、任一失敗則整筆回滾、共編邏輯正確、last_edited_by/at 正確更新、sort=latest 排序正確
4. 媒體編輯歷史（media_edit_history）→ 驗證：每次 PUT 都正確寫入歷史紀錄
5. 心得系統（reviews，含 /reviews/latest）→ 驗證：唯一性約束、owner-scoped、星級評分範圍檢查、最新列表排序正確
6. 愛心與通知（review_likes、notifications）→ 驗證：不可對自己心得按讚、唯一性約束、按讚時正確產生通知、通知列表資料正確
7. 使用者公開主頁（users/:id）→ 驗證：貢獻總覽數字與兩個分頁籤（心得、編輯紀錄）資料正確
8. 管理員下架功能（admin）→ 驗證：僅 admin 可下架，其他人 403
9. 圖片上傳整合 → 驗證：multer 正確處理檔案並儲存路徑
10. 容器化 → 驗證：docker compose 完整啟動
```
