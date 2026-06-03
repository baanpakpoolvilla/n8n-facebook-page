# Facebook Page × n8n — Auto Post & Chatbot Setup

ทักษะสำหรับเชื่อมต่อ Facebook Page กับ n8n บน VPS ทั้งระบบโพสต์อัตโนมัติและ Chat Bot ตอบอัตโนมัติ

---

## ส่วนที่ 1: สิ่งที่ต้องเตรียม

### Facebook
- **Page**: เป็น Admin ของ Page ที่ต้องการเชื่อม
- **Meta Developer App**: สร้างที่ `developers.facebook.com` → เลือก app หรือสร้างใหม่
- **FB Page Token**: Long-lived token (หมดอายุ ~60 วัน) หรือ Never-expiring token
- **App Permissions**: `pages_manage_posts`, `pages_read_engagement`, `pages_messaging`

### n8n
- n8n เวอร์ชัน 2.x บน Docker (VPS หรือ Cloud)
- HTTPS webhook URL (SSL จำเป็นสำหรับ Facebook)

---

## ส่วนที่ 2: ขอ Facebook Page Token

### วิธีที่ 1 — Graph API Explorer (เร็วสุด)
1. `developers.facebook.com` → Tools → Graph API Explorer
2. เลือก App → User or Page → เลือก Page ที่ต้องการ
3. เพิ่ม Permission: `pages_manage_posts`, `pages_messaging`, `pages_read_engagement`
4. กด **Generate Access Token** → คัดลอก token
5. ทดสอบ:
```bash
curl "https://graph.facebook.com/v21.0/me?access_token=TOKEN_HERE"
```

### วิธีที่ 2 — Long-lived Token (60 วัน)
```bash
# Step 1: Exchange short-lived for long-lived user token
curl "https://graph.facebook.com/v1.0/oauth/access_token?grant_type=fb_exchange_token&client_id=APP_ID&client_secret=APP_SECRET&fb_exchange_token=SHORT_TOKEN"

# Step 2: Get Page Token จาก long-lived user token
curl "https://graph.facebook.com/v21.0/me/accounts?access_token=LONG_LIVED_USER_TOKEN"
```

### เช็คว่า Token ยังใช้ได้
```bash
curl "https://graph.facebook.com/debug_token?input_token=TOKEN&access_token=APP_ID|APP_SECRET"
```

---

## ส่วนที่ 3: ตั้งค่า docker-compose.yml

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    environment:
      # Core
      - N8N_HOST=your-subdomain.yourdomain.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://your-subdomain.yourdomain.com/
      - GENERIC_TIMEZONE=Asia/Bangkok

      # Facebook
      - FB_PAGE_ID=123456789012345
      - FB_PAGE_TOKEN=EAAxxxxx...
      - FB_VERIFY_TOKEN=your_custom_verify_token

      # OpenAI (ถ้าใช้ GPT)
      - OPENAI_API_KEY=sk-proj-...

      # n8n v2 — สำคัญมาก: อนุญาต $env ใน nodes
      - N8N_BLOCK_ENV_ACCESS_IN_NODE=false

      # อนุญาต built-in modules ใน Code nodes
      - NODE_FUNCTION_ALLOW_BUILTIN=fs,path
```

---

## ส่วนที่ 4: Fix สำหรับ n8n v2 Task Runner (สำคัญ!)

### ปัญหา
n8n v2 ใช้ Task Runner แยก process สำหรับ execute Code nodes และ expressions  
Task Runner ได้รับ env vars จำกัด — ไม่รวม `OPENAI_API_KEY`, `FB_PAGE_TOKEN`, `N8N_BLOCK_ENV_ACCESS_IN_NODE`  
ผล: `$env.OPENAI_API_KEY` ใน nodes จะ throw **"access to env vars denied"**

### วิธีแก้ — แก้ whitelist ใน Task Runner
```bash
# 1. Copy ไฟล์ออกจาก container
docker cp n8n-n8n-1:/usr/local/lib/node_modules/n8n/dist/task-runners/task-runner-process-js.js /tmp/task-runner-process-js.js

# 2. เพิ่ม env vars ใน getProcessEnvVars() — ต่อจาก N8N_RUNNERS_INSECURE_MODE
sed -i 's|N8N_RUNNERS_INSECURE_MODE: process.env.N8N_RUNNERS_INSECURE_MODE,|N8N_RUNNERS_INSECURE_MODE: process.env.N8N_RUNNERS_INSECURE_MODE,\n            N8N_BLOCK_ENV_ACCESS_IN_NODE: process.env.N8N_BLOCK_ENV_ACCESS_IN_NODE,\n            OPENAI_API_KEY: process.env.OPENAI_API_KEY,\n            FB_PAGE_TOKEN: process.env.FB_PAGE_TOKEN,\n            FB_PAGE_ID: process.env.FB_PAGE_ID,|' /tmp/task-runner-process-js.js

# 3. Copy กลับเข้า container และ restart
docker cp /tmp/task-runner-process-js.js n8n-n8n-1:/usr/local/lib/node_modules/n8n/dist/task-runners/task-runner-process-js.js
docker compose restart n8n
```

> ⚠️ Fix นี้อยู่ใน container layer — จะหายถ้า `docker compose up -d` ใหม่ (recreate container)  
> ควรทำ persistent ด้วย Dockerfile custom image หรือ entrypoint script

### Persistent Fix (แนะนำ)
เพิ่ม entrypoint script ใน docker-compose:
```yaml
volumes:
  - ./fix-task-runner.sh:/docker-entrypoint-init.d/fix-task-runner.sh
```

`fix-task-runner.sh`:
```bash
#!/bin/sh
sed -i 's|N8N_RUNNERS_INSECURE_MODE: process.env.N8N_RUNNERS_INSECURE_MODE,|N8N_RUNNERS_INSECURE_MODE: process.env.N8N_RUNNERS_INSECURE_MODE,\n            N8N_BLOCK_ENV_ACCESS_IN_NODE: process.env.N8N_BLOCK_ENV_ACCESS_IN_NODE,\n            OPENAI_API_KEY: process.env.OPENAI_API_KEY,\n            FB_PAGE_TOKEN: process.env.FB_PAGE_TOKEN,|' \
  /usr/local/lib/node_modules/n8n/dist/task-runners/task-runner-process-js.js
```

---

## ส่วนที่ 5: Workflow — Auto Post ไปยัง Facebook

### โครงสร้าง Nodes
```
Cron (ทุก 09:00) → Set Topic → GPT Write Post → Generate Image → Post to Facebook → Log
```

### Facebook Photos API (โพสต์รูป + ข้อความ)
HTTP Request Node:
```
Method: POST
URL: https://graph.facebook.com/v21.0/{{ $env.FB_PAGE_ID }}/photos
Auth: Form Data
Body:
  - caption: {{ $json.postText }}
  - source: [binary image file]
  - access_token: {{ $env.FB_PAGE_TOKEN }}
```

### ใช้ gpt-image-2 สร้างรูป
```
URL: https://api.openai.com/v1/images/generations
Header: Authorization: Bearer {{ $env.OPENAI_API_KEY }}
Body: { "model": "gpt-image-2", "prompt": "...", "size": "1536x1024", "output_format": "jpeg" }
```
> ผลลัพธ์เป็น base64 → ต้องแปลงเป็น binary ก่อน upload

---

## ส่วนที่ 6: Workflow — Chat Bot ตอบ Messenger

### โครงสร้าง Nodes
```
POST Webhook → Parse Message → IF has text → GPT Reply → Save History → Send Reply → Log
```

### ขั้นตอน Setup Messenger Webhook

**1. สร้าง Workflow ยืนยัน Webhook (GET)**
```
Webhook Node:
  - HTTP Method: GET
  - Path: poolvilla-chat (หรือ path ที่ต้องการ)
  - Response: Code 200, Body: {{ $json.query['hub.challenge'] }}
```

**2. สร้าง Workflow รับข้อความ (POST)**
```
Webhook Node:
  - HTTP Method: POST
  - Path: poolvilla-chat (path เดียวกัน — n8n แยก workflow ได้)
  - Response Mode: Immediately (onReceived)
  - Response Data: No Data (ส่ง 200 ทันที ป้องกัน timeout)
```

**3. ผูก Webhook ใน Meta Developer Console**
1. `developers.facebook.com` → App → Messenger → Settings
2. Webhooks → Add Callback URL:
   ```
   Callback URL: https://your-n8n.domain.com/webhook/poolvilla-chat
   Verify Token: your_custom_verify_token
   ```
3. กด **Verify and Save**
4. Subscribe Fields: ✅ `messages` ✅ `messaging_postbacks`
5. Page Subscriptions → เลือก Page → Subscribe

### Parse Message Node (Code)
```javascript
const body = $json.body || $json;
const messaging = body.entry?.[0]?.messaging?.[0];
if (!messaging) return [{ json: { hasMessage: false } }];

const senderId = messaging.sender?.id;
const text = messaging.message?.text || messaging.postback?.title;
if (!text || messaging.message?.is_echo) return [{ json: { hasMessage: false } }];

// Load conversation history
const sd = $getWorkflowStaticData('global');
const history = (sd.conversations?.[senderId] || []).slice(-10);

return [{ json: { hasMessage: true, senderId, messageText: text, gptMessages: [
  { role: 'system', content: 'System prompt ของคุณ' },
  ...history,
  { role: 'user', content: text }
] }}];
```

### Send Reply Node (HTTP Request)
```
Method: POST
URL: https://graph.facebook.com/v21.0/me/messages?access_token={{ $env.FB_PAGE_TOKEN }}
Body (JSON):
{
  "recipient": { "id": "{{ $json.senderId }}" },
  "message": { "text": "{{ $json.replyText }}" },
  "messaging_type": "RESPONSE"
}
```

### จำประวัติแชท (Save History Code Node)
```javascript
const sd = $getWorkflowStaticData('global');
if (!sd.conversations) sd.conversations = {};
const history = sd.conversations[$json.senderId] || [];
history.push({ role: 'user', content: $json.messageText });
history.push({ role: 'assistant', content: $json.replyText });
if (history.length > 20) history.splice(0, history.length - 20);
sd.conversations[$json.senderId] = history;
```

---

## ส่วนที่ 7: PATCH Workflow ผ่าน n8n API

```bash
# Login
curl -X POST https://n8n.domain.com/rest/login \
  -H 'Content-Type: application/json' \
  -d '{"emailOrLdapLoginId":"email@example.com","password":"password"}' \
  -c /tmp/n8n_cookies.txt

# Get versionId
VERSION=$(curl -s -b /tmp/n8n_cookies.txt https://n8n.domain.com/rest/workflows/WORKFLOW_ID \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["data"]["versionId"])')

# Activate workflow
curl -X POST https://n8n.domain.com/rest/workflows/WORKFLOW_ID/activate \
  -b /tmp/n8n_cookies.txt \
  -H 'Content-Type: application/json' \
  -d "{\"versionId\":\"$VERSION\"}"

# PATCH workflow (ต้องส่ง full workflow JSON)
curl -X PATCH https://n8n.domain.com/rest/workflows/WORKFLOW_ID \
  -b /tmp/n8n_cookies.txt \
  -H 'Content-Type: application/json' \
  --data-binary @workflow.json
```

> ⚠️ PATCH ต้องส่ง complete workflow JSON — ไม่ใช่ partial update  
> ⚠️ อย่าใช้ print() ใน Python ปนกับ JSON output → ทำให้ JSON malformed → 500 error

---

## ส่วนที่ 8: อัปเดต Facebook Token (ทุก ~60 วัน)

```bash
# SSH เข้า VPS
ssh root@YOUR_VPS_IP

# แก้ token ใน docker-compose
sed -i 's/FB_PAGE_TOKEN=.*/FB_PAGE_TOKEN=NEW_TOKEN_HERE/' /docker/n8n/docker-compose.yml

# Recreate container (ไม่ใช่ restart — ต้อง up -d เพื่อ reload env)
cd /docker/n8n && docker compose up -d --force-recreate n8n

# อย่าลืม re-apply Task Runner fix หลัง recreate
```

---

## Troubleshooting

| อาการ | สาเหตุ | วิธีแก้ |
|-------|--------|---------|
| `access to env vars denied` | Task Runner ไม่ได้รับ env vars | Apply Task Runner fix (ส่วนที่ 4) |
| Webhook verify ไม่ผ่าน | Workflow inactive / URL ผิด / token ไม่ตรง | เช็ค workflow active + verify token |
| แชทไม่ตอบ | Webhook ไม่ผูก Page / messaging fields ไม่ได้ subscribe | เช็ค Meta Developer → Messenger → Page subscriptions |
| 401 จาก OpenAI | API key ว่างเปล่า | เช็ค env var + Task Runner fix |
| 190 จาก Facebook | Page token หมดอายุ | อัปเดต FB_PAGE_TOKEN |
| PATCH workflow 500 | JSON malformed (print ปนใน output) | ใช้ `sys.stdout.write()` + redirect stderr |
| Static data หาย | n8n restart ล้าง in-memory state | ปกติ — history จะเริ่มใหม่หลัง restart |
