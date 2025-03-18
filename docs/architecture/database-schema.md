# 📌 予約システム データベース設計（PostgreSQL + NoSQL + Redis）

本予約システムのデータ管理は **PostgreSQL（RDBMS）をメイン** に、  
**MongoDB（NoSQL）と Redis（インメモリDB）を併用** し、スケーラブルかつリアルタイム対応可能な設計を採用する。

---

## **1. データベース構成**
| データ | 保存場所 | 理由 |
|--------|---------|------|
| **ユーザー・予約情報** | PostgreSQL | 正規化・一貫性を保つ |
| **予約履歴・会話ログ** | MongoDB | 柔軟な構造でのログ保存 |
| **リアルタイム予約枠** | Redis | 高速なデータアクセス |

---

## **2. テーブル設計（PostgreSQL）**
### **2.1. `users`（ユーザー情報）**
ユーザー情報を管理する。

| カラム名 | 型 | 制約 | 説明 |
|----------|-----|------|------|
| `id` | UUID | PRIMARY KEY | ユーザーID |
| `name` | VARCHAR(255) | NOT NULL | ユーザー名 |
| `email` | VARCHAR(255) | UNIQUE | メールアドレス |
| `phone` | VARCHAR(20) | UNIQUE | 電話番号 |
| `line_id` | VARCHAR(50) | UNIQUE | LINEアカウントID |
| `created_at` | TIMESTAMP | DEFAULT now() | 登録日時 |
| `updated_at` | TIMESTAMP | DEFAULT now() | 更新日時 |

---

### **2.2. `stores`（店舗情報）**
店舗情報を管理する。

| カラム名 | 型 | 制約 | 説明 |
|----------|-----|------|------|
| `id` | UUID | PRIMARY KEY | 店舗ID |
| `name` | VARCHAR(255) | NOT NULL | 店舗名 |
| `address` | TEXT | NOT NULL | 住所 |
| `phone` | VARCHAR(20) | NOT NULL | 電話番号 |
| `business_hours` | JSONB | NOT NULL | 営業時間設定 |
| `created_at` | TIMESTAMP | DEFAULT now() | 登録日時 |

---

### **2.3. `staff`（スタッフ情報）**
スタッフ情報を管理する。

| カラム名 | 型 | 制約 | 説明 |
|----------|-----|------|------|
| `id` | UUID | PRIMARY KEY | スタッフID |
| `store_id` | UUID | FOREIGN KEY | 所属店舗 |
| `name` | VARCHAR(255) | NOT NULL | スタッフ名 |
| `role` | VARCHAR(50) | NOT NULL | 役割（医師・美容師など） |
| `created_at` | TIMESTAMP | DEFAULT now() | 登録日時 |

---

### **2.4. `reservations`（予約情報）**
予約情報を管理する。

| カラム名 | 型 | 制約 | 説明 |
|----------|-----|------|------|
| `id` | UUID | PRIMARY KEY | 予約ID |
| `user_id` | UUID | FOREIGN KEY | 予約者 |
| `store_id` | UUID | FOREIGN KEY | 店舗 |
| `staff_id` | UUID | FOREIGN KEY | スタッフ |
| `menu_id` | UUID | FOREIGN KEY | メニュー |
| `start_time` | TIMESTAMP | NOT NULL | 予約開始時間 |
| `end_time` | TIMESTAMP | NOT NULL | 予約終了時間 |
| `status` | ENUM('confirmed', 'pending', 'canceled') | DEFAULT 'pending' | 予約状態 |
| `notes` | TEXT | NULL | ユーザーメモ |

---

### **2.5. `menus`（メニュー情報）**
メニュー情報を管理する。

| カラム名 | 型 | 制約 | 説明 |
|----------|-----|------|------|
| `id` | UUID | PRIMARY KEY | メニューID |
| `store_id` | UUID | FOREIGN KEY | 店舗ID |
| `name` | VARCHAR(255) | NOT NULL | メニュー名 |
| `price` | DECIMAL(10,2) | NOT NULL | 価格 |
| `duration` | INT | NOT NULL | 所要時間（分） |

---

### **2.6. `payments`（決済情報）**
決済情報を管理する。

| カラム名 | 型 | 制約 | 説明 |
|----------|-----|------|------|
| `id` | UUID | PRIMARY KEY | 支払いID |
| `reservation_id` | UUID | FOREIGN KEY | 予約ID |
| `user_id` | UUID | FOREIGN KEY | 支払い者 |
| `amount` | DECIMAL(10,2) | NOT NULL | 支払金額 |
| `status` | ENUM('pending', 'completed', 'failed') | DEFAULT 'pending' | 支払い状態 |
| `created_at` | TIMESTAMP | DEFAULT now() | 支払い日時 |

---

## **3. NoSQL（MongoDB: ログ管理）**


### **3.1. `reservation_history`（予約履歴）**
```json
{
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "history": [
    {
      "reservation_id": "987f6543-b21c-45d7-a6f1-987654321000",
      "store": "XYZクリニック",
      "menu": "ホワイトニング",
      "date": "2025-03-16T14:00:00Z",
      "status": "completed"
    }
  ]
}
```


### **3.2. `chat_logs`（対話履歴ログ）**
```json
{
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "channel": "LINE",
  "logs": [
    {
      "timestamp": "2025-03-16T14:00:00Z",
      "role": "user",
      "message": "明日空いてる？"
    },
    {
      "timestamp": "2025-03-16T14:00:01Z",
      "role": "system",
      "message": "明日は15:00〜17:00が空いています。予約しますか？"
    }
  ]
}
```

## **4. インメモリDB（Redis: キャッシュ管理）**


### **4.1. 予約枠キャッシュ**
```bash
SET available_slots:store_123 "['2025-03-16T14:00:00Z', '2025-03-16T15:00:00Z']"
```

### **4.2. キャンセル待ちリスト**
```bash
LPUSH waitlist:store_123:user_456 "2025-03-16T14:00:00Z"
```

