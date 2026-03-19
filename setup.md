# 🖥️ NEXUS — Corporate Remote Desktop Management System
## Complete Full-Stack Development Prompt

---

> **Behave as**: Senior Full-Stack Engineer + Systems Architect with 10+ years of experience building enterprise-grade internal tooling, real-time web applications, and network management systems. You are opinionated, pragmatic, and security-conscious. You write production-ready code with proper error handling, logging, and documentation.

---

## 📌 Project Overview

Build **NEXUS** — a corporate remote desktop management web application for a 100-PC office network. The system allows:
- Any authorized user to remotely access another PC **with the target user's permission**
- Admins to access any PC **directly without permission**
- Real-time session management with the ability to disconnect at any time
- A live dashboard showing the status of all 100 endpoints

---

## 🏗️ Tech Stack

### Frontend
- **Framework**: React 18 + Vite
- **Styling**: Tailwind CSS + custom CSS variables
- **Real-time**: Socket.io-client
- **State**: Zustand (global) + React Query (server state)
- **Remote View**: noVNC (WebSocket-based VNC client in-browser)
- **Routing**: React Router v6

### Backend
- **Runtime**: Node.js + Express.js
- **Real-time**: Socket.io
- **Auth**: JWT (access + refresh tokens) + bcrypt
- **Database**: PostgreSQL (via Prisma ORM)
- **Cache / PubSub**: Redis
- **VNC Proxy**: websockify (Node.js bridge between browser WebSocket and VNC TCP)

### On-Device Agent (installed on each PC)
- **Language**: Python 3.10+
- **VNC Server**: LibVNCServer or TigerVNC (platform-specific)
- **Agent**: Lightweight Python daemon that:
  - Registers with central server on boot
  - Listens for access requests and shows a system-level permission popup
  - Reports live status (online/busy/offline) via heartbeat
  - Starts/stops VNC on demand
  - Kills VNC session when admin revokes access

---

## 🗃️ Database Schema (PostgreSQL + Prisma)

```prisma
model User {
  id          String    @id @default(uuid())
  name        String
  email       String    @unique
  password    String
  role        Role      @default(USER)
  pcId        String?   @unique
  pc          PC?       @relation(fields: [pcId], references: [id])
  sessions    Session[] @relation("requester")
  createdAt   DateTime  @default(now())
}

model PC {
  id          String    @id @default(uuid())
  pcNumber    Int       @unique   // 1 to 100
  label       String              // "PC-001"
  ipAddress   String    @unique
  status      PCStatus  @default(OFFLINE)
  user        User?
  sessions    Session[]
  lastSeen    DateTime?
  createdAt   DateTime  @default(now())
}

model Session {
  id            String        @id @default(uuid())
  pc            PC            @relation(fields: [pcId], references: [id])
  pcId          String
  requester     User          @relation("requester", fields: [requesterId], references: [id])
  requesterId   String
  accessType    AccessType    @default(VIEW_ONLY)
  status        SessionStatus @default(PENDING)
  vncPort       Int?
  startedAt     DateTime?
  endedAt       DateTime?
  createdAt     DateTime      @default(now())
}

model AuditLog {
  id          String    @id @default(uuid())
  action      String    // "SESSION_STARTED", "SESSION_ENDED", "REQUEST_DENIED", etc.
  actorId     String
  targetPcId  String?
  metadata    Json?
  createdAt   DateTime  @default(now())
}

enum Role          { USER ADMIN }
enum PCStatus      { ONLINE BUSY OFFLINE IN_SESSION }
enum AccessType    { VIEW_ONLY VIEW_AND_CONTROL FILE_TRANSFER }
enum SessionStatus { PENDING APPROVED DENIED ACTIVE ENDED CANCELLED }
```

---

## 🔌 API Endpoints (REST + Socket.io)

### Authentication
```
POST   /api/auth/login              — Login, returns JWT
POST   /api/auth/refresh            — Refresh access token
POST   /api/auth/logout             — Invalidate refresh token
GET    /api/auth/me                 — Get current user info
```

### PC Management
```
GET    /api/pcs                     — List all PCs with status
GET    /api/pcs/:id                 — Get specific PC details
PATCH  /api/pcs/:id/status          — Agent updates PC status (agent auth only)
```

### Sessions
```
POST   /api/sessions                — Create session request
GET    /api/sessions                — List sessions (admin: all, user: own)
GET    /api/sessions/active         — Active sessions only
DELETE /api/sessions/:id            — Disconnect session
PATCH  /api/sessions/:id/respond    — Target user approves/denies request
```

### Audit
```
GET    /api/audit                   — Admin only: full audit log
```

### Socket.io Events (bidirectional real-time)
```
CLIENT → SERVER:
  "pc:heartbeat"      { pcId, status }          — Agent ping every 15s
  "session:request"   { pcId, accessType }       — User requests access
  "session:respond"   { sessionId, approved }    — Target user responds
  "session:end"       { sessionId }              — Disconnect

SERVER → CLIENT:
  "pc:status_update"  { pcId, status }           — Broadcast PC status change
  "session:incoming"  { sessionId, from, type }  — Notify target user of request
  "session:approved"  { sessionId, vncPort }     — Tell requester: connect to VNC
  "session:denied"    { sessionId }              — Tell requester: denied
  "session:revoked"   { sessionId, reason }      — Force-end a session
  "admin:stats"       { online, busy, offline, sessions } — Dashboard stats
```

---

## 🔐 Security Requirements

### Authentication & Authorization
- JWT access tokens expire in **15 minutes**, refresh tokens in **7 days**
- Refresh tokens stored in **HttpOnly cookies** (not localStorage)
- Every API route protected via `authMiddleware`
- Role-based guards: `requireAdmin` middleware for admin-only routes
- Agent devices authenticate via a **shared secret + HMAC signature** (not user JWT)

### Session Security
- VNC port is **randomly assigned** per session (range 5900–5999)
- VNC is **only accessible via the server's websockify proxy** — never exposed directly to the LAN
- VNC connection uses a **one-time token** that expires in 60 seconds if unused
- All VNC traffic is tunneled through **TLS WebSocket**
- When a session ends (any reason), the VNC server is immediately killed on the agent

### Permission Flow (non-admin users)
```
1. User A (PC-042) clicks PC-015 → selects access type → clicks "Send Request"
2. Server creates Session with status=PENDING
3. Server emits "session:incoming" to PC-015's socket
4. PC-015 agent shows OS-level popup: "PC-042 wants to access your computer. [Allow] [Deny]"
5a. User approves → agent emits "session:respond" {approved:true}
    → Server starts VNC on PC-015, assigns port, creates one-time VNC token
    → Server emits "session:approved" to User A with VNC token
    → User A's browser opens noVNC iframe using the token
5b. User denies → server emits "session:denied" to User A
6.  At any point, PC-015 user can click "Revoke Access" on system tray icon
    → Agent emits "session:end" → server kills VNC → emits "session:revoked" to User A
```

### Admin Override Flow
```
1. Admin clicks any PC → clicks "Connect Directly"
2. Server verifies admin role → skips permission step
3. Directly starts VNC on target PC via agent command
4. AuditLog entry created: "ADMIN_OVERRIDE_ACCESS"
5. Admin still appears in sessions panel (visible to other admins)
```

---

## 🖥️ Frontend Pages & Components

### `/login` — Auth Page
- Clean login form (email + password)
- Shows role after login (Admin / User)

### `/dashboard` (main page)
- **TopBar**: Logo, search, live stats pills (Online/Busy/Offline count), admin badge, user avatar
- **Sidebar**: Navigation (PC Grid, Active Sessions, Permissions, Users, Inventory, Audit Log, Settings)
- **Main Content — PC Grid**: 
  - 100 PC cards in responsive grid (default 10 columns)
  - Filter chips: All / Online / Busy / Offline / In Session / My Access Only
  - Search by PC number or username
  - Each card shows: status dot, PC monitor icon, PC number, assigned user, status tag
  - Click → opens Connect Modal
- **Right Panel**: 
  - Sessions tab (active sessions with duration timer + disconnect button)
  - Requests tab (pending permission requests with approve/deny)
  - Info tab (details of last clicked PC)
- **Live Ticker** (bottom bar): Live sessions count, last scan time, network subnet, clock

### Connect Modal
- PC info summary (number, user, IP, status)
- Permission notice (for non-admin)
- Access type selector: View Only / View + Control / File Transfer
- "Send Request" CTA

### Waiting Modal
- Spinner with message "Waiting for approval from [User]"
- Cancel button

### `/session/:id` — Full Screen Remote View
- noVNC canvas fills entire screen
- Floating control bar (top): PC name, duration timer, disconnect button, access type indicator
- Keyboard shortcut: Escape → disconnect confirmation dialog

### `/admin/users` — User Management (Admin only)
- Table of all users: name, email, assigned PC, role, last active
- Assign PC to user, change role, reset password

### `/admin/audit` — Audit Log (Admin only)
- Full log of all events: sessions, permission requests, admin overrides, logins
- Filterable by date, action type, user, PC

---

## 🤖 On-Device Python Agent

```python
# Structure
agent/
├── main.py           # Entry point, manages all modules
├── vnc_manager.py    # Start/stop VNC server
├── socket_client.py  # Connects to central Socket.io server
├── notification.py   # Shows OS-level permission popup (tkinter/plyer)
├── tray_icon.py      # System tray: "Active session — Revoke access"
├── config.py         # Reads from agent_config.json
└── agent_config.json # { "serverUrl", "agentSecret", "pcId" }
```

**Agent responsibilities**:
1. On startup: Connect to server WebSocket, authenticate with `agentSecret + pcId`
2. Every 15s: Emit heartbeat with current status (ONLINE/BUSY based on CPU/input activity)
3. On "session:request_to_agent" event: Show popup → emit response
4. On "session:start" event: Start VNC on assigned port, emit ready confirmation
5. On "session:end" or "session:revoke": Kill VNC process, update status

---

## 📁 Project Structure

```
nexus/
├── frontend/                   # React + Vite app
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Login.jsx
│   │   │   ├── Dashboard.jsx
│   │   │   ├── SessionView.jsx
│   │   │   ├── admin/
│   │   │   │   ├── Users.jsx
│   │   │   │   └── AuditLog.jsx
│   │   ├── components/
│   │   │   ├── layout/
│   │   │   │   ├── TopBar.jsx
│   │   │   │   ├── Sidebar.jsx
│   │   │   │   └── RightPanel.jsx
│   │   │   ├── pc/
│   │   │   │   ├── PCGrid.jsx
│   │   │   │   ├── PCCard.jsx
│   │   │   │   └── ConnectModal.jsx
│   │   │   ├── session/
│   │   │   │   ├── SessionCard.jsx
│   │   │   │   ├── WaitingModal.jsx
│   │   │   │   └── SessionViewer.jsx (noVNC wrapper)
│   │   │   └── ui/             # Toast, Modal, Badge, Button, etc.
│   │   ├── store/
│   │   │   ├── useAuthStore.js
│   │   │   ├── usePCStore.js
│   │   │   └── useSessionStore.js
│   │   ├── hooks/
│   │   │   ├── useSocket.js
│   │   │   └── useSessionTimer.js
│   │   └── lib/
│   │       ├── api.js          # Axios instance with interceptors
│   │       └── socket.js       # Socket.io singleton
│   └── index.html
│
├── backend/                    # Node.js + Express
│   ├── src/
│   │   ├── index.js            # Server entry, Socket.io init
│   │   ├── routes/
│   │   │   ├── auth.routes.js
│   │   │   ├── pc.routes.js
│   │   │   ├── session.routes.js
│   │   │   └── audit.routes.js
│   │   ├── controllers/
│   │   ├── middleware/
│   │   │   ├── auth.middleware.js
│   │   │   └── admin.middleware.js
│   │   ├── services/
│   │   │   ├── vnc.service.js  # Manages websockify processes
│   │   │   └── session.service.js
│   │   ├── socket/
│   │   │   ├── handlers/
│   │   │   │   ├── pc.handlers.js
│   │   │   │   └── session.handlers.js
│   │   │   └── index.js
│   │   └── prisma/
│   │       └── schema.prisma
│   └── package.json
│
├── agent/                      # Python daemon for each PC
│   ├── main.py
│   ├── vnc_manager.py
│   ├── socket_client.py
│   ├── notification.py
│   ├── tray_icon.py
│   ├── config.py
│   ├── requirements.txt
│   └── agent_config.json.example
│
└── docker-compose.yml          # PostgreSQL + Redis + Backend + Frontend
```

---

## 🚦 Implementation Order (Phase-wise)

### Phase 1 — Foundation (Week 1–2)
1. Set up monorepo, Docker Compose (Postgres + Redis)
2. Backend: Auth system (register, login, JWT refresh)
3. Backend: PC and User CRUD with Prisma
4. Frontend: Login page, route guards, Zustand auth store
5. Frontend: Dashboard shell (TopBar, Sidebar, Right Panel layout)

### Phase 2 — Real-time Core (Week 3–4)
1. Socket.io server setup with auth middleware
2. PC heartbeat system (agent → server → broadcast)
3. PC Grid frontend with live status updates
4. Status filter, search, PC cards

### Phase 3 — Session & Permission System (Week 5–6)
1. Session request flow (create, notify via Socket.io)
2. Connect Modal + Waiting Modal UI
3. Agent: permission popup (OS-level notification)
4. Approve/Deny flow → Session activated
5. Right panel: Sessions tab + Requests tab

### Phase 4 — VNC Integration (Week 7–8)
1. Agent: VNC server start/stop (TigerVNC)
2. Backend: websockify process manager
3. Frontend: noVNC viewer in `/session/:id`
4. One-time VNC token auth
5. Disconnect flow (all paths: user, target, admin)

### Phase 5 — Admin Features & Polish (Week 9–10)
1. Admin override (direct access, no permission needed)
2. Audit log (all events logged, admin view)
3. User management page (assign PCs, roles)
4. Agent installer script (cross-platform: Windows + Linux)
5. Performance: Redis caching for PC status, rate limiting

---

## ✅ Non-Functional Requirements

| Requirement       | Target                                    |
|-------------------|-------------------------------------------|
| Concurrency       | Handle 100 PCs + 50 concurrent sessions  |
| Latency           | Status updates < 500ms                   |
| Session start     | < 3 seconds from approval to VNC connect |
| Uptime            | 99.5% (internal corporate tool)           |
| Auth security     | OWASP Top 10 compliance                  |
| Audit trail       | All access events logged with timestamps |
| VNC encryption    | TLS WebSocket tunnel mandatory           |

---

## 🧪 Testing Plan
- **Unit**: Jest (backend services, utils)
- **Integration**: Supertest (API endpoints)
- **E2E**: Playwright (login → connect flow → disconnect)
- **Socket**: Custom test harness simulating agent heartbeats

---

*NEXUS v1.0 — Internal Corporate Remote Control Platform*
*Build with: React · Node.js · Socket.io · PostgreSQL · Python · noVNC*