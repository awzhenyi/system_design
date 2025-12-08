# Core Entities for Notification System

## 1. Client
- **Description:** Organization or user that sends notifications.
- **Attributes:**
  - Client ID
  - Name
  - API Key/Secret
  - Contact Info

## 2. User
- **Description:** Recipient of notifications.
- **Attributes:**
  - User ID
  - Name
  - Email
  - Phone Number
  - Notification Preferences (Email/SMS/Alert)

## 3. Notification
- **Description:** A message to be delivered to one or more users.
- **Attributes:**
  - Notification ID
  - Client ID (sender)
  - Template ID
  - Channel (Email/SMS/Alert)
  - Content (personalized)
  - Status (Pending/Sent/Failed)
  - Priority
  - Scheduled Time
  - Created At

## 4. Template
- **Description:** Predefined message format for notifications.
- **Attributes:**
  - Template ID
  - Client ID (owner)
  - Channel (Email/SMS/Alert)
  - Subject (for Email)
  - Body
  - Placeholders/Variables

## 5. Delivery Log
- **Description:** Record of notification delivery attempts.
- **Attributes:**
  - Log ID
  - Notification ID
  - User ID (recipient)
  - Channel
  - Status (Delivered/Failed/Pending)
  - Timestamp
  - Error Message (if any)
