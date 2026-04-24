# Avenyx Setup Guide

## Prerequisites

- Node.js 18+
- Docker & Docker Compose
- PostgreSQL (included in Docker)
- Java 17+ (for Android build)

---

## Quick Start

### 1. Clone the Project

```bash
git clone <repository-url>
cd avenyx
```

### 2. Configure Environment

Copy the example env file and edit:

```bash
cp api/.env api/.env
```

Edit `api/.env`:
```env
DATABASE_URL="postgresql://postgres:postgres@db:5432/avenyx"
JWT_SECRET="your-secure-random-secret-key"
RESEND_API_KEY="re_your_resend_api_key"
FRONTEND_URL="https://avenyx.qzz.io"
ADMIN_URL="https://admin.avenyx.qzz.io"
APK_STORAGE_PATH="./storage/apk"
```

### 3. Start Docker Services

```bash
cd docker
docker-compose up -d
```

This starts:
- `api` on port 3000
- `db` (PostgreSQL) on port 5432
- `frontend` on port 3001
- `admin` on port 3002

### 4. Verify Services

```bash
curl http://localhost:3000/health
```

---

## Initial Setup

### 1. Create Database Tables

```bash
cd ../api
docker-compose exec api npx prisma db push
```

Or from host (if database is accessible):
```bash
cd api
npx prisma db push
```

### 2. Create Admin User

```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@avenyx.com","password":"admin123","name":"Admin"}'
```

### 3. Upgrade to Admin Role

Connect to PostgreSQL and update:

```bash
docker-compose exec db psql -U postgres -d avenyx -c "UPDATE \"User\" SET role = 'ADMIN' WHERE email = 'admin@avenyx.com';"
```

---

## Marzban Node Setup

### Prerequisites

Each Marzban node must have:
- Marzban panel installed
- API enabled with an API key
- VLESS inbound configured

### Add Node via Admin Panel

1. Go to http://localhost:3002/login
2. Login with admin credentials
3. Navigate to Nodes
4. Click "Add Node"
5. Fill in details:
   - **Name**: e.g., "India Node 1"
   - **Location**: India or Germany
   - **API URL**: e.g., https://marzban.india.example.com
   - **API Key**: Your Marzban API key
   - **Inbound ID**: The VLESS inbound ID (usually 1)
   - **Capacity**: Max users (e.g., 100)

---

## Plan Setup

### Create Plans via Admin Panel

1. Go to http://localhost:3002/plans
2. Create plans:

| Plan | Price | Duration | Devices |
|------|-------|----------|--------|
| Basic | $5 | 30 days | 1 |
| Standard | $12 | 90 days | 3 |
| Premium | $25 | 180 days | 5 |

---

## User Flow

### Register User

1. Go to http://localhost:3001/register
2. Create account
3. Login at http://localhost:3001/login

### Create Subscription

1. Go to Dashboard
2. Select a plan
3. Click "Create Subscription"
4. Note the amount

### Make Payment

Contact admin with payment details. Admin approves in http://localhost:3002/payments

### Connect VPN

1. Select location (India/Germany)
2. Click Connect button

---

## APK Distribution

### Build Android App

```bash
cd android
./gradlew assembleRelease
```

APK will be at: `app/build/outputs/apk/release/app-release.apk`

### Upload APK

```bash
cp app/build/outputs/apk/release/app-release.apk ../storage/apk/latest.apk
```

Or mount as volume in docker-compose.yml:
```yaml
volumes:
  - ./apk:/storage/apk
```

### Download via API

```
GET http://localhost:3000/apk/download
```

---

## Production Deployment

### 1. DNS Setup

Configure DNS:
- A record:avenyx.qzz.io -> server IP
- A record: admin.avenyx.qzz.io -> server IP
- A record: api.avenyx.qzz.io -> server IP

### 2. Nginx Configuration

```nginx
server {
    server_nameavenyx.qzz.io;
    location / {
        proxy_pass http://localhost:3001;
    }
}

server {
    server_name admin.avenyx.qzz.io;
    location / {
        proxy_pass http://localhost:3002;
    }
    location /api {
        proxy_pass http://localhost:3000;
    }
}

server {
    server_name api.avenyx.qzz.io;
    location / {
        proxy_pass http://localhost:3000;
    }
}
```

### 3. SSL (Let's Encrypt)

```bash
certbot --nginx -d avenyx.qzz.io -d admin.avenyx.qzz.io -d api.avenyx.qzz.io
```

### 4. Update Environment

Update `.env` with production URLs and secrets.

---

## Troubleshooting

### Check Logs

```bash
docker-compose logs -f api
docker-compose logs -f db
```

### Restart Services

```bash
docker-compose restart
```

### Reset Database

```bash
docker-compose exec db psql -U postgres -d avenyx -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
cd api && npx prisma db push
```

### Common Issues

**Database connection failed:**
- Check DATABASE_URL format
- Ensure PostgreSQL container is running

**API returns 401:**
- Token expired or invalid
- Login again to get new token

**VPN connection failed:**
- Check Marzban node is accessible
- Verify API key is correct
- Check inbound ID exists

---

## API Reference

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| /auth/register | POST | No | Register user |
| /auth/login | POST | No | Login user |
| /auth/me | GET | JWT | Get current user |
| /plans | GET | No | List active plans |
| /plans | POST | ADMIN | Create plan |
| /subscriptions/me | GET | JWT | My subscription |
| /payments | GET | ADMIN | List payments |
| /payments/:id/approve | POST | ADMIN | Approve payment |
| /nodes | GET | No | List active nodes |
| /nodes | POST | ADMIN | Create node |
| /vpn/config | GET | JWT | Get VPN config |
| /vpn/connect | POST | JWT | Create VPN user |
| /apk/latest | GET | No | Get APK info |
| /apk/download | GET | No | Download APK |
| /admin/users | GET | ADMIN | List users |

---

## File Structure

```
avenyx/
├── api/                 # Backend API
│   ├── src/
│   │   └── server.js   # Main server
│   ├── prisma/
│   │   └── schema.prisma
│   ├── Dockerfile
│   └── package.json
├── frontend/            # Customer frontend
│   ├── src/
│   │   └── app/
│   │       ├── page.js          # Landing
│   │       ├── login/
│   │       ├── register/
│   │       └── dashboard/
│   ├── Dockerfile
│   └── package.json
├── admin/              # Admin frontend
│   ├── src/
│   │   └── app/
│   │       ├── login/
│   │       ├── users/
│   │       ├── payments/
│   │       ├── nodes/
│   │       └── plans/
│   ├── Dockerfile
│   └── package.json
├── android/            # Android app
│   └── app/
├── docker/
│   └── docker-compose.yml
├── storage/
│   └── apk/
├── SPEC.md
└── README.md
```