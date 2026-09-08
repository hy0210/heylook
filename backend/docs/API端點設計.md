# API 端點設計

回主文件：[後端規劃.md](./後端規劃.md)　對應資料實體：[資料實體設計.md](./資料實體設計.md)

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
POST   /api/auth/signup                    註冊，成功即自動登入（見下方詳細說明）
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
GET    /api/media                          媒體庫列表（公開，可篩選 type/format，支援 sort=latest 依建立時間倒序）
GET    /api/media/:mediaId                 媒體詳情（公開，含心得列表、編輯紀錄）
GET    /api/media/:mediaId/history         查看編輯歷史（公開）

POST   /api/media                          建立媒體資料＋第一篇心得（需登入，一次請求同時寫入 media 與 review 兩筆資料）
PUT    /api/media/:mediaId                 編輯媒體資料（需登入，任何會員皆可，寫入 last_edited_by/at + history）
```

**首頁「最新建立作品」區塊**直接呼叫 `GET /api/media?sort=latest&limit=5`，不需要額外的聚合端點。

**`POST /api/media` 的請求格式**（media 欄位與 review 欄位混合在同一個 multipart/form-data 請求中）：
```
fields: title, type, format, synopsis, creator_name, release_year,
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

**合計約 24 個端點**，聚焦「帳號系統」、「共編媒體資料」、「owner-scoped 心得」、「輕量社交互動（愛心＋通知）」四套核心邏輯。

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

fields: title, type, format, synopsis, creator_name, release_year
file:   cover_image
```

後端可用 `multer` 處理上傳，圖片存放於本地 `uploads/` 目錄（開發階段）或串接雲端物件儲存（如 Cloudinary / S3，視你想展示的技能深度決定）。頭像上傳（`PUT /api/users/profile`）沿用同一套 multer 設定。

---

## 【帳號系統：註冊／登入／登出設計】

不使用第三方登入（無 Google／Facebook OAuth），全部帳密由平台自行管理。註冊不做 email 驗證，填寫資料即建立帳號並自動登入（二階段 email 驗證列入【下一階段規劃】）。

### 流程一：註冊

```
POST /api/auth/signup
  body: { email, password, nickname }
  
  後端動作：
  1. 檢查 email 是否已註冊過（users 表）、nickname 是否已被使用
  2. 建立 users 資料（password_hash 以 bcrypt 產生）
  3. 簽發 JWT，回傳 { token, user }（註冊成功即自動登入，不需再手動登入一次）
```

### 流程二：登入 / 登出

```
POST /api/auth/login
  body: { email, password }
  
  後端動作：
  1. 查 users 表比對 email + password_hash（bcrypt 驗證）
  2. 簽發 JWT，回傳 { token, user }

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

專案結構、能力展示重點、下一階段規劃、開發順序建議請見主文件：[後端規劃.md](./後端規劃.md)
