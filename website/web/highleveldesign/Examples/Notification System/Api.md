# Notification System API

## Authentication
All endpoints require an API key in the header:
```
Authorization: Bearer <API_KEY>
```

---

## 1. Send Notification
**POST** `/api/notifications/send`

**Request Body:**
```json
{
  "template_id": "string",
  "channel": "email|sms|alert",
  "recipients": ["user_id1", "user_id2"],
  "personalization": {
    "user_id1": { "name": "Alice" },
    "user_id2": { "name": "Bob" }
  },
  "schedule_time": "2024-06-01T10:00:00Z", // optional
  "priority": "high|normal|low" // optional
}
```

**Response:**
```json
{
  "notification_id": "string",
  "status": "pending|sent|failed"
}
```

---

## 2. Get Notification Status
**GET** `/api/notifications/{notification_id}`

**Response:**
```json
{
  "notification_id": "string",
  "status": "pending|sent|failed",
  "delivery": [
    {
      "user_id": "string",
      "channel": "email|sms|alert",
      "status": "delivered|failed|pending",
      "error_message": "string|null"
    }
  ]
}
```

---

## 3. Create Template
**POST** `/api/templates`

**Request Body:**
```json
{
  "channel": "email|sms|alert",
  "subject": "string", // for email
  "body": "string",
  "placeholders": ["name", "code"]
}
```

**Response:**
```json
{
  "template_id": "string"
}
```

---

## 4. List Templates
**GET** `/api/templates`

**Response:**
```json
[
  {
    "template_id": "string",
    "channel": "email|sms|alert",
    "subject": "string",
    "body": "string",
    "placeholders": ["name", "code"]
  }
]
```

---

## 5. Delete Template
**DELETE** `/api/templates/{template_id}`

**Response:**
```json
{
  "success": true
}
```