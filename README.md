# RabbitMQ Banking System

A full-stack banking platform built with a **React** frontend, **Node.js/Express** backend, **PostgreSQL** database, and a **Python microservice** that handles asynchronous messaging via **RabbitMQ**. The system supports core banking operations including user authentication, account management, and event-driven notifications through a message queue architecture.


## Architecture Overview

```
┌────────────────────────────────────────────────────────────┐
│                        CLIENT                              │
│              React App (port 3000)                         │
│   MUI Components │ FullCalendar │ Axios HTTP Requests      │
└──────────────────────────┬─────────────────────────────────┘
                           │ HTTP (proxied to :3001)
┌──────────────────────────▼─────────────────────────────────┐
│                   NODE.JS BACKEND                          │
│              Express Server (port 3001)                    │
│   Auth (JWT/bcrypt) │ REST APIs │ Nodemailer │ Twilio SMS  │
└──────────┬───────────────────────────────┬─────────────────┘
           │ SQL (pg Pool)                 │ AMQP
┌──────────▼──────────┐        ┌───────────▼─────────────────┐
│   PostgreSQL DB     │        │        RabbitMQ             │
│  Users, Accounts,   │        │   Message Queue Broker      │
│  Transactions, etc. │        └───────────┬─────────────────┘
└─────────────────────┘                    │ AMQP (pika)
                               ┌───────────▼─────────────────┐
                               │   Python Microservice       │
                               │   (Dockerized Consumer)     │
                               │  Processes async events,    │
                               │  DB writes, notifications   │
                               └─────────────────────────────┘
```

The system follows a **microservice-oriented** pattern where time-sensitive or side-effect-heavy operations (e.g., transaction processing, notifications) are offloaded from the main API server and delegated to a dedicated Python consumer via RabbitMQ. This keeps the Node.js server responsive and decouples event processing from the HTTP request lifecycle.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, MUI (Material UI / Joy UI), FullCalendar, React Router v6, Axios |
| Backend | Node.js, Express 4, JSON Web Tokens, bcrypt |
| Database | PostgreSQL (via `pg` connection pool) |
| Message Broker | RabbitMQ (AMQP protocol) |
| Messaging Consumer | Python 3, `pika` (RabbitMQ client), `psycopg2` (PostgreSQL client) |
| Notifications | Nodemailer (email), Twilio (SMS) |
| Containerization | Docker |

---

## System Components

### React Frontend

The client is a single-page application bootstrapped with **Create React App**. It communicates with the backend exclusively through a proxy configured to `http://localhost:3001`, meaning all `/api/...` calls in the React code are automatically forwarded to the Express server during development.

Key libraries:
- **MUI (Material UI & Joy UI)** — component library for all UI elements (tables, forms, modals, icons)
- **`@mui/x-data-grid`** — for rich, sortable/filterable data tables (e.g., transaction history, account lists)
- **`@fullcalendar/react`** — calendar UI for scheduling features (e.g., payment due dates, appointments)
- **`react-auth-kit`** — lightweight authentication state management (stores JWT in memory/cookies, provides `useAuthUser` hooks)
- **`react-router-dom v6`** — client-side routing with protected routes
- **`axios`** — HTTP client for all API calls, with interceptors for attaching auth tokens

### Node.js Backend

The Express server acts as the primary API layer between the frontend and all downstream services.

Responsibilities:
- **Authentication**: Issues and validates JWTs. Passwords are hashed with `bcrypt`/`bcryptjs` before storage.
- **REST API**: Handles all CRUD operations for users, accounts, and transactions by querying PostgreSQL directly via a `pg` connection pool (`db.js`).
- **Message Publishing**: When certain banking events occur (e.g., a new transaction is submitted), the server publishes a message to a RabbitMQ queue using `amqplib`. This offloads slow or asynchronous work from the HTTP response cycle.
- **Notifications**: Uses `nodemailer` to send transactional emails (e.g., transaction confirmations, alerts) and `twilio` for SMS-based two-factor authentication or alerts.

### Python Messaging Service

This is a **long-running Dockerized process** that subscribes to RabbitMQ queues using the `pika` library. It acts as an asynchronous worker/consumer.

Responsibilities:
- Listens continuously for messages published by the Node.js backend.
- Processes events such as transaction completions, fraud flags, or account updates.
- Writes processed results directly to PostgreSQL using `psycopg2`.
- Sends downstream notifications or triggers further business logic without blocking the main API.

The service is containerized independently, allowing it to be scaled, restarted, or deployed separately from the rest of the stack.

### PostgreSQL Database

The central data store for all persistent application data. The Node.js backend connects to it through a `pg.Pool` instance configured via environment variables. SSL is enforced (`rejectUnauthorized: false`) to support hosted database providers (e.g., AWS RDS, Supabase, Neon).

The Python service also connects directly to PostgreSQL via `psycopg2` to perform write operations as part of processing consumed messages.

### RabbitMQ Message Broker

RabbitMQ is the asynchronous backbone of the system. It decouples the Node.js API from the Python processing service using the **AMQP protocol**.

Flow:
1. A banking event occurs (e.g., E-transfer initiated).
2. Node.js publishes a serialized message to a named RabbitMQ queue via `amqplib`.
3. RabbitMQ durably holds the message until a consumer is ready.
4. The Python service picks up the message via `pika`, processes it, and acknowledges it.

This pattern ensures **no event is lost** even if the Python service is temporarily unavailable — RabbitMQ will hold the messages until it reconnects.

---

## Project Structure

```
/
├── Dockerfile                  # Docker config for Python messaging service
├── requirements.txt            # Python dependencies (pika, psycopg2)
├── PythonServer.py             # Python RabbitMQ consumer entrypoint
│
├── package.json                # Node.js backend dependencies & scripts
├── db.js                       # PostgreSQL connection pool (pg)
├── Server/
│   └── index.js                # Express app entrypoint
│
└── src/                        # React frontend (Create React App)
    ├── package.json            # Frontend dependencies & scripts
    ├── public/
    └── src/
        ├── components/         # Reusable UI components
        ├── pages/              # Route-level page components
        └── App.js              # Root app with router & auth provider
```

---



## Authentication & Security

- **JWT (JSON Web Tokens)**: Issued on login and validated on every protected API request via middleware.
- **bcrypt**: All user passwords are hashed using bcrypt before being stored in the database. Plain-text passwords are never persisted.
- **SSL on DB connections**: The PostgreSQL pool is configured with SSL enabled to support encrypted connections to cloud-hosted databases.
- **React Auth Kit**: Manages the auth state on the frontend, protecting routes and attaching tokens to outgoing requests.

---

## Messaging Architecture

The RabbitMQ integration uses a **producer/consumer** pattern:

```
Node.js (Producer)
    │
    │  amqplib.connect(RABBITMQ_URL)
    │  channel.sendToQueue('queue_name', Buffer.from(JSON.stringify(payload)))
    ▼
RabbitMQ Queue
    │
    │  pika.BlockingConnection(...)
    │  channel.basic_consume(queue='queue_name', on_message_callback=handler)
    ▼
Python Consumer
    │
    │  Process message
    │  psycopg2 → write to PostgreSQL
    │  Acknowledge message (channel.basic_ack)
    ▼
Done
```

Messages are JSON-serialized payloads describing banking events. The Python consumer deserializes them, performs the required database operations or business logic, then sends an acknowledgment back to RabbitMQ to confirm processing is complete.

---

## API Overview

All API routes are served from `http://localhost:3001` and prefixed appropriately. The React app proxies all requests through `http://localhost:3000` during development.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/register` | Register a new user |
| `POST` | `/api/auth/login` | Authenticate and receive JWT |
| `GET` | `/api/accounts` | Get all accounts for authenticated user |
| `POST` | `/api/accounts` | Create a new bank account |
| `GET` | `/api/transactions` | Fetch transaction history |
| `POST` | `/api/transactions` | Initiate a new transaction (publishes to RabbitMQ) |

> Note: All protected routes require an `Authorization: Bearer <token>` header.

---

## Available Scripts

### Backend (Node.js)

| Command | Description |
|---|---|
| `npm start` | Starts the Express server (`Server/index.js`) |
| `npm test` | Placeholder — no tests configured |

### Frontend (React)

| Command | Description |
|---|---|
| `npm start` | Runs the app in development mode at `localhost:3000` |
| `npm run build` | Builds the app for production into the `build/` folder |
| `npm test` | Launches the test runner in interactive watch mode |
| `npm run eject` | Ejects from Create React App (irreversible) |

### Python Consumer

| Command | Description |
|---|---|
| `python PythonServer.py` | Starts the RabbitMQ consumer process |
| `docker build -t banking-python-consumer .` | Builds the Docker image |
| `docker run --env-file .env banking-python-consumer` | Runs the consumer in a container |

