# n8n × Facebook Page Automation

Base template สำหรับเชื่อมต่อ Facebook Page กับ n8n บน self-hosted VPS  
ใช้ได้กับทุก Page — ปรับ env vars แล้วใช้งานได้ทันที

## สิ่งที่ได้

- **Auto Post** — โพสต์รูป + ข้อความลง Facebook Page อัตโนมัติตามตาราง
- **Chat Bot** — ตอบ Messenger อัตโนมัติด้วย GPT-4o จำประวัติแชทต่อคน

## โครงสร้าง

```
n8n-facebook-page/
├── README.md
├── SETUP_GUIDE.md          # คู่มือตั้งค่าทั้งหมด (Facebook Token, Webhook, Fix n8n v2)
├── docker-compose.yml      # Template พร้อมใช้
└── workflows/
    ├── auto-post.json      # Workflow โพสต์อัตโนมัติ
    └── chat-bot.json       # Workflow Chat Bot + Webhook verify
```

## เริ่มต้นเร็ว

1. Copy `docker-compose.yml` → ใส่ค่า env vars
2. `docker compose up -d`
3. Apply [n8n v2 Task Runner Fix](SETUP_GUIDE.md#ส่วนที่-4-fix-สำหรับ-n8n-v2-task-runner-สำคัญ)
4. Import workflows จากโฟลเดอร์ `workflows/`
5. ผูก Facebook Webhook ตาม [SETUP_GUIDE.md](SETUP_GUIDE.md)

## Requirements

- VPS ที่มี Docker + domain + SSL
- Facebook Page (Admin access)
- Meta Developer App
- OpenAI API Key (สำหรับ GPT-4o และ gpt-image-2)
