# Email System

A production-grade **email system** built using **Node.js**, featuring complete SMTP infrastructure, custom DNS resolution, security protocols, and real-time notifications. This project demonstrates deep understanding of email protocols, distributed systems architecture, and asynchronous processing.

---

## Features

### Core Email Functionality

- **Multi-domain support** - Send and receive emails across own domain and external domains
- **Optimized delivery** - Emails to self processed directly by backend; outbound worker stores emails directly in recipient mailboxes for local domain users, delivers to recipents SMTP server for external domains
- **Email threading** - Automatic conversation detection for replies
- **Attachment handling** - Upload and receive file attachments
- **Bounce management** - Automatic bounce emails for undeliverable messages
- **Delivery reliability** - 3 retry attempt mechanism for temporary failures

### User Experience

- **Real-time notifications** - Instant new emails sent to frontend via Server-Sent Events (SSE)
- **Search** - Quickly find emails across your mailbox
- **Email management** - Star, read/unread, delete, trash, and restore
- **Smart composition** - Auto-suggestions for frequently contacted recipients
- **Reply & Forward** - Full conversation threading support
- **Cursor-based pagination** - Efficient loading for large mailboxes
- **Modern UI** - Responsive design with dark mode support

---

## Architecture

The system follows a **distributed, event-driven architecture** with asynchronous communication for fault tolerance and scalability.

```
                               ┌─────────────────────┐
                               │       Nginx         │
                               │ • Serves Frontend   │
                               │ • Proxies API calls │
                               │ • Rate Limiting     │
                               └──────────┬──────────┘
                                          │
              Store            ┌──────────▼──────────┐
         ┌─────────────────────┤  Backend (Fastify)  |
         │                     │ • Business Logic    │
         │                     │ • Server Sent Events│
         │                     └──────┬───────▲──────┘
         │                            │       │
         │                   Enqueue  │       │ Subscribe
         │                  outbound  |       │ to events
         │                            │       |
  ┌──────▼─────────────┐       ┌──────▼───────────┐
  │      MongoDB       │       │    Redis + BullMQ│
  │ • User mailboxes   │  ┌───►│ • Job Queues     │
  │ • Email metadata   │  |    │ • Pub/Sub Events │──────┐
  │ • Thread           │  |    └──────┬───────▲───┘      │
  │ • Attachments      │  |           │   Publish Event  │
  └──────▲───────▲─────┘  |    ┌──────▼─────┐ │ ┌────────▼────────┐
         │       │        |    │  Inbound   │ │ │    Outbound     │
         │       │        |    │  Queue     │ │ │    Queue        │
         │       │        |    └──────┬─────┘ │ └────────┬────────┘
         │       │        |           │       │          │
         │       │        |    ┌──────▼─────┐_│ ┌────────▼────────────┐
         │       │ Store  |    │   Inbound  │ └─┤  Outbound Worker    │
         │       └────────|────┤   Worker   │   │ • Internal Delivery |
         │                |    │ • Process  │   │ • MX resolution     │
         |                |    | • Store    │   │ • Delivery          │
         |                |    │ • Notify   │   | • Retry             │
         |   Store        |    └────────────┘   └────┬────┬──┬────────┘
         └────────────────|──────────────────────────┘    │  | Deliver
                          |                               │  |    ┌───────────────┐
                 Enqueue  |    ┌────────────┐  Resolve MX |  │    |    Remote     │
                 inbound  |    | SMTP Server│             |  └───►| Mail Transfer │
                          |    │ • Port 25  │             │       │  Agent(MTA)   │
                          └────│ • Validate │Inbound Mail │       └─┬────┬────────┘
                               │ • Greylist |◄────────────|─────────┘    │
                               └──────┬─────┘             │              │
                                      │                   │              │
                               ┌──────▼───────────────────▼──────────────▼─┐
                               │           Custom DNS Server               │
                               │ • MX records / SPF / DKIM / DMARC         │
                               │ • Resolves all type of DNS request        │
                               └───────────────────────────────────────────┘

```

### Data Flow

**Incoming Email (External Domain):**

1. Remote MTA → Custom SMTP Server (Port 25)
2. SMTP checks if the IP is greylisted, validates email security (SPF, DKIM, DMARC via DNS Server)
3. SMTP adds email to **Inbound Queue** (BullMQ)
4. Inbound Worker picks job: processes attachments, detects threads, saves to MongoDB
5. Inbound Worker publishes new mail arrival event via **Redis Pub/Sub**
6. Backend receives event and pushes **SSE notification** to recipient's browser

**Outgoing Email (External Domain):**

1. Frontend sends email data to Backend via REST API
2. Backend immediately stores in sender's mailbox (MongoDB)
3. Backend adds job to **Outbound Queue** and responds to frontend (non-blocking)
4. Outbound Worker picks job from queue
5. Worker resolves recipient's MX record via DNS Server
6. Worker sends email via Nodemailer to recipent's SMTP server.
7. On failure: retries up to 3 times, generates bounce email if permanently failed
8. Worker publishes delivery failure event via Redis Pub/Sub
9. Backend receives event and sends delivery failure mail to frontend via SSE

**Email within your domain(Different Users):**

1. Frontend sends email data to Backend
2. Backend stores in sender's mailbox (MongoDB)
3. Backend adds the job to **Outbound Queue**
4. Outbound Worker picks job and saves to recipient's mailbox.
5. Generates bounce mail if some recipient's doesn't exist.
6. Worker publishes event via Redis Pub/Sub
7. Backend receives event and pushes new mails to users using SSE

**Email to Self (Same User):**

1. Frontend sends email data to Backend
2. Backend directly stores in user's mailbox (MongoDB) - no queue needed
3. Backend immediately pushes new mail to frontend via SSE

---

## Tech Stack

| Component            | Technology                                         |
| -------------------- | -------------------------------------------------- |
| **Frontend**         | React                                              |
| **Backend**          | Node.js, Fastify                                   |
| **SMTP Server**      | Node.js                                            |
| **Workers**          | Node.js (Inbound & Outbound)                       |
| **Queue System**     | BullMQ + Redis                                     |
| **Database**         | MongoDB                                            |
| **Pub/Sub**          | Redis                                              |
| **Web Server**       | Nginx (serves frontend + reverse proxy to backend) |
| **DNS Resolution**   | Custom Node.js based DNS Server                    |
| **Containerization** | Docker + Docker Compose                            |

---

## Security Features

- **Email Authentication** - SPF, DKIM, and DMARC verification
- **Greylisting** - Temporary rejection senders with multiple failed Email Authentication to prevent spam
- **Rate Limiting** - Multiple layers at Nginx, Backend and SMTP levels
- **Private Backend** - Backend isolated on private network, accessible only via Nginx
- **Input Validation** - Comprehensive validation of email headers and content

---

## Setup & Installation

### Prerequisites

- Docker & Docker Compose
- Custom DNS Server (https://github.com/n3erajann/dns-server) - **Only required for external domain emails**. Emails between users within your domain work without DNS server.

### Installation

1. **Clone the repository**

   ```bash
   git clone https://github.com/n3erajann/email-system.git
   cd email-system
   ```

2. **Configure environment**

```bash
   cp .env.example .env
   # Edit .env with your configuration

   # Required: Configure backend and frontend
   cd backend && cp .env.example .env

   cd ../frontend && cp .env.example .env

   # Optional: Other components use shared .env by default
   # In development, you may need to configure individual component .env files
```

3. **Start all services**

   ```bash
   docker compose up -d
   ```

4. **Verify DNS server (Optional - for external domains only)**
   - You can skip this step for emails that belong to your domain
   - For sending/receiving external domain emails:
     - Ensure your custom DNS server is accessible
     - Verify MX record resolution is working

5. **Access the application**
   - Frontend: `http://localhost` (or configured domain)
   - SMTP Server: Port 25
   - Backend API: Proxied through Nginx

---

## Project Structure

```
email-system/
├── backend/                  # Fastify API server
│   ├── src/
│   ├── .env.example
│   ├── Dockerfile
│   └── server.js             # Entry point
│
├── core/                     # Shared utilities (used by backend, smtp-server, workers)
│
├── frontend/                 # React application
│   ├── public/
│   ├── src/
│   │   └── main.jsx         # Entry point
│   ├── nginx.conf
│   ├── .env.example
│   ├── Dockerfile
│   └── index.html           # HTML entry point
│
├── inbound-worker/           # Incoming email processor
│   ├── attachments/
│   ├── handlers/
│   ├── threading/
│   ├── .env.example
│   ├── Dockerfile
│   └── index.js              # Entry point
│
├── outbound-worker/          # Outgoing email processor
│   ├── assembly/
│   ├── storage/
|   ├── transport/
│   ├── .env.example
│   ├── Dockerfile
│   └── index.js             # Entry point
│
├── smtp-server/              # Custom SMTP server
│   ├── config/
│   ├── .env.example
│   ├── Dockerfile
│   └── server.js            # Entry point
│
├── compose.yaml             # Service orchestration
└── README.md
```

---

## Key Technical Achievements

- **Custom SMTP Implementation** - Full-featured SMTP server built using Node.js
- **Custom DNS Server Integration** - Full control over domain resolution, MX records, and email security verification
- **Internal vs External Handling** - Emails to domain owned by the system are processed directly; external domains sent via nodemailer.
- **Event-Driven Architecture** - Redis Pub/Sub for real-time inter-service communication
- **Asynchronous Processing** - BullMQ job queues with retry logic and failure handling
- **Real-Time Updates** - Server-Sent Events (SSE) pushes new emails to frontend in real-time
- **Email Threading** - Automatic conversation detection and grouping
- **Production-Ready Architecture** - Fault-tolerant, scalable, and containerized

---

## Demo Environment

For testing and demonstration purposes, a multi-server setup was created using Docker Macvlan networking, allowing two complete mail server replicas with different domain names to run on the same port (25). This setup is not included in the repository but was used to validate:

- Internal domain handling
- External domain delivery
- Cross-domain communication
- Bounce and retry mechanisms
- Security protocol validation

---

## Author

**Nirajan Paudel**

- LinkedIn: www.linkedin.com/in/nirajan-paudel-a9b052265
- GitHub: https://github.com/n3erajann

---
