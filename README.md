# 🤖 EZ-Chatbot: Multi-Tenant AI Agent & Conversational Booking Platform

[![NestJS](https://img.shields.io/badge/NestJS-v11.0-E0234E?style=flat&logo=nestjs&logoColor=white)](https://nestjs.com/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![LangGraph](https://img.shields.io/badge/LangGraph-StateGraph-1C3C3C?style=flat&logo=langchain&logoColor=white)](https://langchain-ai.github.io/langgraphjs/)
[![Model Context Protocol](https://img.shields.io/badge/MCP-Standard-4B32C3?style=flat)](https://modelcontextprotocol.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-GiST_Constraints-336791?style=flat&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Socket.io](https://img.shields.io/badge/Socket.io-WebSockets-010101?style=flat&logo=socketdotio&logoColor=white)](https://socket.io/)
[![Next.js](https://img.shields.io/badge/Next.js-App_Router-black?style=flat&logo=next.js&logoColor=white)](https://nextjs.org/)
[![React](https://img.shields.io/badge/React-18.x-61DAFB?style=flat&logo=react&logoColor=black)](https://react.dev/)

> **EZ-Chatbot** is an enterprise-grade, multi-tenant conversational AI platform designed to transform customer support and appointment scheduling for service businesses. It bridges cutting-edge **Agentic AI orchestration (LangGraph + Model Context Protocol)** with **distributed backend engineering** to deliver real-time, race-condition-free booking, automated RAG knowledge retrieval, and seamless live human agent escalation.

---

## 📑 Table of Contents
1. [Executive Overview](#-executive-overview)
2. [System Architecture](#-system-architecture)
3. [Core Engineering Highlights](#-core-engineering-highlights)
   - [1. Agentic AI & MCP Tool Orchestration](#1-agentic-ai--mcp-tool-orchestration)
   - [2. Race-Condition-Free Booking Engine](#2-race-condition-free-booking-engine)
   - [3. High-Throughput Real-Time WebSockets](#3-high-throughput-real-time-websockets)
   - [4. Hardened Security & Multi-Tenant Isolation](#4-hardened-security--multi-tenant-isolation)
   - [5. Embeddable Widget & Real-Time Analytics](#5-embeddable-widget--real-time-analytics)
4. [Monorepo Structure](#-monorepo-structure)
5. [Technology Stack](#-technology-stack)
6. [Key User Journeys](#-key-user-journeys)
7. [Getting Started & Local Setup](#-getting-started--local-setup)
8. [Author & Lead Engineer](#-author--lead-engineer)

---

## 🌟 Executive Overview

Modern customer support faces severe trade-offs: either expensive SaaS platforms with unpredictable per-seat pricing, or complex, disconnected open-source bots lacking domain-specific execution. 

**EZ-Chatbot** solves this by providing:
* **True Multi-Tenancy:** 100% strict data isolation supporting 1,000+ business tenants on a single shared infrastructure.
* **Agentic Tool Execution (MCP):** Decoupled, tenant-aware tool connectivity via Anthropic's **Model Context Protocol (MCP)** for calendar inspection, slot booking, and knowledge search.
* **Autonomous Conversational Booking:** Complete end-to-end appointment scheduling, telephone verification, cancellation, and mid-dialogue recovery in a web chat interface.
* **Bulletproof Database Concurrency:** Elimination of double-booking race conditions through mathematical database exclusion constraints.
* **Hybrid Human Handoff:** Real-time presence detection and live agent takeover via bi-directional WebSocket rooms.

---

## 🏛️ System Architecture

The platform is designed around a micro-services / modular monorepo architecture:

```mermaid
flowchart TB
    subgraph ClientLayer ["Client Layer"]
        Visitor["🌐 Website Visitor"]
        AdminUser["👔 Business Admin / Agent"]
        Widget["💬 Embeddable Chat Widget\n(React + Zustand + Socket.io)"]
        Dashboard["📊 Admin Dashboard\n(Next.js + Refine + AntD)"]
    end

    subgraph GatewayLayer ["Gateway & Ingestion Layer"]
        ChatGW["⚡ Chat WebSocket Gateway\n(Socket.io / Presence Tracking)"]
        NotifGW["🔔 Notification Gateway\n(Role-scoped Event Broadcasting)"]
        RestAPI["🛡️ REST API Controllers\n(NestJS + Swagger + RTR Auth)"]
    end

    subgraph CoreEngine ["Core Backend Engine (chatbot-api)"]
        LangGraphAgent["🧠 LangGraph Agent StateGraph\n(Flow Classifier, Intent, Abort Nodes)"]
        BookingEngine["📅 Booking Core Service\n(Slot Calculation, Hold Expiry)"]
        AuthService["🔐 Auth & Session Service\n(RTR + 2-min Grace Period)"]
    end

    subgraph MCPLayer ["AI Tool Layer (Model Context Protocol)"]
        MCPServer["🔌 MCP Tool Server (@modelcontextprotocol/sdk)"]
        Tools["🛠️ Tools:\n• searchKnowledge\n• getAvailableSlots\n• holdSlot / createBooking\n• cancelBooking\n• getUserDetails"]
    end

    subgraph DataStorage ["Data & Cache Storage"]
        Postgres[("🐘 PostgreSQL\nTypeORM + btree_gist\nExclusion Constraints")]
        RedisStore[("⚡ Redis\nBullMQ Queues +\nPub/Sub Cache")]
    end

    Visitor --> Widget
    AdminUser --> Dashboard
    Widget <-->|WebSockets (Chat)| ChatGW
    Dashboard <-->|WebSockets (Live Events)| NotifGW
    Dashboard -->|REST / NextAuth| RestAPI
    Widget -->|REST / Visitor Auth| RestAPI

    ChatGW --> LangGraphAgent
    NotifGW --> RedisStore
    RestAPI --> AuthService
    RestAPI --> BookingEngine

    LangGraphAgent <-->|MCP Client Adapter| MCPServer
    MCPServer --> Tools
    Tools <-->|Internal Secure REST| RestAPI

    BookingEngine <--> Postgres
    AuthService <--> Postgres
    LangGraphAgent <--> RedisStore
```

---

## ⚙️ Core Engineering Highlights

### 1. Agentic AI & MCP Tool Orchestration
* **StateGraph State Machine:** Implemented using `@langchain/langgraph` with a sliding 30-message history reducer (`AgentState`). Nodes include **Intent Classifier**, **Fast-Path Booking Router**, **Knowledge Search Status Handler**, and **Abort / Flow Abandonment Handler**.
* **Context-Aware Flow Abandonment:** When a user is in the middle of a structured multi-step booking or cancellation flow and switches context (e.g., asks *"Where is your address?"* or says *"Never mind, keep my booking"*), the engine cleanly aborts the sub-dialogue without leaking ghost states.
* **Decoupled MCP Architecture:** Instead of hard-coding API calls inside the LLM prompt, tool calling is externalized via the **Model Context Protocol (`@modelcontextprotocol/sdk`)**. Every MCP tool call is guarded with strict tenant scoping (`siteId`) and secret tokens (`McpSecretGuard`).

### 2. Race-Condition-Free Booking Engine
* **PostgreSQL GiST Exclusion Constraints:** High-traffic booking platforms suffer from concurrency race conditions where two simultaneous requests can reserve the same staff/slot. 
  Instead of fragile application-level mutexes, the engine enforces a database-level exclusion constraint using PostgreSQL's `btree_gist` extension:
  ```sql
  ALTER TABLE "bookings" ADD CONSTRAINT "no_overlap" 
  EXCLUDE USING gist (
      "siteId" WITH =,
      COALESCE("resourceId"::text, '___NULL___') WITH =,
      tstzrange("startAt", "endAt") WITH &&
  ) WHERE (status = 'confirmed');
  ```
  This guarantees mathematical zero-overlap at the storage engine level even under distributed load.
* **Dynamic Slot Availability:** Slot availability algorithm supports both assigned staff and a *"No Preference"* resource assignment mode with automated buffer intervals and business timezone conversions (`date-fns-tz`).

### 3. High-Throughput Real-Time WebSockets
* **Role-Scoped Socket Rooms:** Handled via `@nestjs/websockets` and Socket.io. Clients automatically join isolated rooms:
  - `site-settings-{siteId}`: Instant branding and behavior updates across all visitor widgets.
  - `admin-notifications-{siteId}`: Broadcasts customer escalation requests to on-duty staff.
  - `conversation-{conversationId}`: Direct bi-directional conversation pipeline between visitor and AI/human agent.
* **Agent Presence & Escalation:** Heartbeat-driven presence tracking detects when live support agents are active to enable conversational handoff.

### 4. Hardened Security & Multi-Tenant Isolation
* **Refresh Token Rotation (RTR) with Grace Period:** Prevents token replay attacks without punishing legitimate users experiencing network latency or concurrent browser requests. The backend maintains a **2-minute grace period buffer** where the previous refresh token remains valid for in-flight requests before hard invalidation.
* **Multi-Tenant Isolation:** Every DB query, cache key, vector embedding search, and MCP invocation is strictly partitioned by `siteId`. Super Admin, Site Admin, and Support Agent roles are enforced via custom NestJS Guards and RBAC policies.

### 5. Embeddable Widget & Real-Time Analytics
* **Modular Micro-Widget:** Built with React, TypeScript, and Zustand. Decoupled from legacy monolithic structures into clean hooks (`useSocketChat`, `useBookingActions`), customizable CSS theme variables, and multi-language support (i18n).
* **Binary Feedback System (Thumbs Up / Down):** Visitors can rate AI answers with custom feedback reasons. Feedback is asynchronously aggregated into the admin analytics dashboard with real-time KPI metrics.

---

## 📂 Monorepo Structure

```text
ez-chatbot-project/
├── final-project-ez-chatbot-api/       # 🚀 NestJS Core Backend & Agentic Gateway
│   ├── src/
│   │   ├── auth/                       # JWT Authentication, RTR & 2-Min Grace Period
│   │   ├── booking/                    # Core Booking Engine, PostgreSQL GiST Constraints
│   │   ├── chat/                       # LangGraph StateGraph, Flow Classifier, Socket Gateway
│   │   ├── mcp/                        # Internal MCP Controller & Security Guards
│   │   ├── notifications/              # Real-Time WebSocket Notification Gateway
│   │   ├── knowledge/                  # RAG Vector Search & Document Ingestion
│   │   └── site/                       # Multi-tenant Configuration & Templates
│   └── package.json
│
├── final-project-ez-mcp/               # 🔌 Model Context Protocol Server
│   ├── src/
│   │   ├── tools/                      # MCP Tools: booking, slot holds, knowledge, user details
│   │   ├── schemas/                    # Zod Input / Output validation schemas
│   │   └── server.ts                   # MCP Server Runtime (@modelcontextprotocol/sdk)
│   └── package.json
│
├── final-project-ez-chat-widget/        # 💬 Embeddable Visitor Chat Widget
│   ├── src/
│   │   ├── components/booking/         # InlineSlotPicker, BookingConfirmationCard
│   │   ├── store/                      # Zustand Centralized Chat State
│   │   ├── hooks/                      # Socket.io Communication & Presence Hooks
│   │   └── locales/                    # i18n Translation dictionaries
│   └── package.json
│
├── final-project-ez-admin-dashboard/   # 📊 Next.js Enterprise Admin Dashboard
│   ├── app/                            # Next.js App Router & NextAuth Integration
│   ├── src/
│   │   ├── components/                 # Refine & Ant Design Management Views
│   │   ├── hooks/                      # Real-time WebSocket Status & KB Hooks
│   │   └── lib/api.client.ts           # Authenticated API Client with Auto-Refresh
│   └── package.json
│
├── final-project-ez-chatbot-landing/   # 🌐 Product Landing Page
└── README.md                           # 📖 Project Documentation
```

---

## 🛠️ Technology Stack

| Layer | Technologies & Libraries |
| :--- | :--- |
| **Backend Framework** | **NestJS v11**, Express, Node.js (v20+), RxJS, TypeScript |
| **AI & Agentic Tooling** | **LangChain**, **LangGraph** (`StateGraph`), **Model Context Protocol (MCP)**, OpenAI GPT-4o API |
| **Database & ORM** | **PostgreSQL**, **TypeORM**, `btree_gist` Exclusion Constraints, Migrations |
| **Caching & Queues** | **Redis**, **BullMQ** (Background jobs & email delivery) |
| **Real-Time Communication**| **Socket.io**, `@nestjs/websockets`, Event-driven architectures |
| **Authentication & Security**| Passport JWT, Refresh Token Rotation (RTR), Bcrypt, Helmet, Throttler |
| **Frontend & UI** | **Next.js (App Router)**, **React 18**, **Zustand**, Ant Design, Refine, TailwindCSS |
| **Code Quality & Tooling** | ESLint, Prettier, Husky, Lint-Staged, Jest, Swagger (`@nestjs/swagger`) |

---

## 🔄 Key User Journeys

### 1. The Autonomous Conversational Booking Journey
```text
Visitor: "I'd like to book a haircut tomorrow afternoon."
   │
   ├─► 1. ChatGateway receives message ➔ LangGraph Agent StateGraph invokes.
   ├─► 2. Classifier Node identifies intent: 'book_appointment'.
   ├─► 3. MCP Tool 'getAvailableSlots' is called with date range and service requirements.
   ├─► 4. Agent returns structured response ➔ Widget renders <InlineSlotPicker /> component.
   │
Visitor: Selects 2:30 PM slot with Stylist David & inputs phone number.
   │
   ├─► 5. Fast-path sub-handler reserves slot via 'holdSlot' MCP Tool.
   ├─► 6. Verification completes ➔ BookingEngine persists record to PostgreSQL.
   ├─► 7. 'no_overlap' constraint validates zero conflict ➔ Confirmed!
   └─► 8. WebSockets emit real-time event to Admin Dashboard calendar instantly.
```

---

## 🚀 Getting Started & Local Setup

### Prerequisites
* **Node.js**: `v20.x` or higher
* **npm**: `v10.x` or higher
* **PostgreSQL**: `v15+` (with `btree_gist` extension enabled)
* **Redis**: `v7+`

### 1. Clone & Install Dependencies
```bash
# Clone the showcase repository
git clone https://github.com/JasonNguyenHMinh/ez-chatbot-platform.git
cd ez-chatbot-platform

# Install dependencies for all services
cd final-project-ez-chatbot-api && npm ci && cd ..
cd final-project-ez-mcp && npm ci && cd ..
cd final-project-ez-chat-widget && npm ci && cd ..
cd final-project-ez-admin-dashboard && npm ci && cd ..
```

### 2. Environment Configuration
Copy `.env.example` to `.env` in `final-project-ez-chatbot-api`:
```bash
cp final-project-ez-chatbot-api/.env.example final-project-ez-chatbot-api/.env
```
Key configuration values:
```env
PORT=3000
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=postgres
DATABASE_PASSWORD=your_password
DATABASE_NAME=ez_chatbot_db
REDIS_HOST=localhost
REDIS_PORT=6379
OPENAI_API_KEY=sk-...
JWT_SECRET=super_secret_jwt_key
REFRESH_TOKEN_GRACE_PERIOD_MS=120000
MCP_SERVER_URL=http://localhost:3002
```

### 3. Run Database Migrations
```bash
cd final-project-ez-chatbot-api
npm run migration:run
npm run seed:super-admin
```

### 4. Start Development Servers
Open separate terminals to launch each service:
```bash
# Terminal 1: Backend API
cd final-project-ez-chatbot-api && npm run start:dev

# Terminal 2: MCP Server
cd final-project-ez-mcp && npm run start

# Terminal 3: Chat Widget
cd final-project-ez-chat-widget && npm run dev

# Terminal 4: Admin Dashboard
cd final-project-ez-admin-dashboard && npm run dev
```

* API Documentation (Swagger): `http://localhost:3000/api/docs`
* Admin Dashboard: `http://localhost:3001`
* Widget Testbed: `http://localhost:5173`

---

## 👨‍💻 Author & Engineering Profile

**Nguyen Hoang Minh (Jason Minh)**  
*Backend & AI Systems Engineer*  
* Ho Chi Minh City University of Technology (HCMUT - VNU-HCM)
* **Specialization:** Backend Scalability, Distributed Systems, Network Mechanics, Agentic AI Architectures.
* **GitHub:** [@JasonNguyenHMinh](https://github.com/JasonNguyenHMinh) / [@JasonHoangMinh](https://github.com/JasonHoangMinh)
* **Email:** minhnguyenh250904@gmail.com
* **LinkedIn:** [Nguyễn Hoàng Minh](https://www.linkedin.com/in/nguyễn-hoàng-minh-6003292bb/)

---
<div align="center">
  <sub>Developed with pride as part of the Zodinet Advanced Engineering Program (2026).</sub>
</div>
