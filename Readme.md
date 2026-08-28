# Ledger Core

> A production-grade financial transaction API built with Node.js, Express, and MongoDB.

**Ledger Core** is a backend system for handling financial accounts and money transfers using **double-entry bookkeeping**, **MongoDB ACID transactions**, **idempotency protection**, and **JWT-based authentication**.

The project is designed around an important financial-system principle:

> **The ledger is the source of truth.**

Account balances are derived from immutable ledger entries rather than being treated as an independently mutable value.

---

## ✨ Features

* 🔐 **JWT Authentication**

  * User registration and login
  * HTTP-only authentication cookies
  * Bearer token support
  * Logout with token revocation

* 🏦 **Account Management**

  * Create financial accounts
  * Retrieve user accounts
  * Account status management
  * Balance calculation from ledger history

* 📒 **Double-Entry Ledger**

  * Every transfer creates a matching `DEBIT` and `CREDIT`
  * Ledger entries are immutable
  * Complete transaction history
  * Balance derived from ledger entries

* 💸 **Atomic Money Transfers**

  * MongoDB sessions and transactions
  * All-or-nothing transfer execution
  * Automatic rollback when a transaction fails

* 🔑 **Idempotent Transactions**

  * Idempotency keys prevent duplicate transfers
  * Safe retries when clients resend requests

* 🤖 **System Account**

  * Privileged system user for initial fund allocation

* 📧 **Transaction Notifications**

  * Email notifications using Nodemailer and Gmail OAuth2

---

## 🏗️ Architecture

```text
                    ┌──────────────────┐
                    │      Client      │
                    │ Web / Mobile/API │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   Express API    │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        Authentication   Accounts      Transactions
              │              │              │
              └──────────────┼──────────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │     MongoDB      │
                    │                  │
                    │ Users            │
                    │ Accounts         │
                    │ Transactions     │
                    │ Ledger Entries   │
                    │ Token Blacklist  │
                    └──────────────────┘
```

---

## 💰 Double-Entry Ledger Model

The system does not simply subtract money from one account and add it to another.

Every transfer produces two ledger entries:

```text
Sender Account
      │
      │ DEBIT
      ▼
┌───────────────┐
│    Ledger     │
└───────────────┘
      ▲
      │ CREDIT
      │
Recipient Account
```

For example, transferring `$100` from Account A to Account B produces:

```text
Account A   DEBIT    $100
Account B   CREDIT   $100
```

The ledger therefore provides an auditable history of every movement of funds.

---

## 🔄 Transaction Flow

A transfer follows an atomic workflow:

```text
1. Validate request
        ↓
2. Validate idempotency key
        ↓
3. Verify sender and recipient
        ↓
4. Calculate sender balance
        ↓
5. Start MongoDB transaction
        ↓
6. Create transaction record
        ↓
7. Create DEBIT ledger entry
        ↓
8. Create CREDIT ledger entry
        ↓
9. Mark transaction completed
        ↓
10. Commit transaction
        ↓
11. Send notification
```

If any operation fails before the commit:

```text
MongoDB Transaction
        │
        ├── Transaction record
        ├── Debit entry
        └── Credit entry
                 │
                 ▼
              FAILURE
                 │
                 ▼
              ROLLBACK
```

This prevents partially completed transfers.

---

## 🔑 Idempotency

Financial APIs must safely handle retries.

For example, a client sends:

```http
POST /api/transactions
Idempotency-Key: transfer-abc-123
```

If the client times out and sends the same request again, the idempotency mechanism prevents the transfer from being processed twice.

This protects against scenarios such as:

```text
Client
  │
  │ Transfer $100
  ▼
Server
  │
  │ Transaction succeeds
  ▼
Network timeout
  │
  ▼
Client retries
  │
  │ Same Idempotency-Key
  ▼
Server
  │
  └── Duplicate request detected
```

---

## 🧮 Balance Calculation

Balances are derived from ledger activity.

Conceptually:

```text
Balance = Total Credits - Total Debits
```

For example:

```text
Credits:
  + $1,000
  + $500

Debits:
  - $200
  - $100

Balance:
  $1,500 - $300 = $1,200
```

This makes the ledger the authoritative record of financial activity.

---

## 🔐 Authentication

Authentication is implemented using JWTs.

Supported authentication mechanisms include:

```http
Cookie: access_token=<token>
```

or:

```http
Authorization: Bearer <token>
```

Tokens can also be revoked during logout using a blacklist mechanism.

---

## 🛠️ Tech Stack

| Technology   | Purpose                   |
| ------------ | ------------------------- |
| Node.js      | Runtime                   |
| Express 5    | REST API                  |
| MongoDB      | Database                  |
| Mongoose     | MongoDB ODM               |
| JWT          | Authentication            |
| bcryptjs     | Password hashing          |
| Nodemailer   | Email notifications       |
| Gmail OAuth2 | Email provider            |
| dotenv       | Environment configuration |
| Nodemon      | Development server        |

The current package configuration confirms Express, Mongoose, JWT, bcryptjs, Nodemailer, dotenv and related dependencies.

---

## 📁 Project Structure

```text
ledger-core/
│
├── server.js
├── package.json
│
└── src/
    │
    ├── app.js
    │
    ├── config/
    │   └── db.js
    │
    ├── controllers/
    │   ├── auth.controller.js
    │   ├── account.controller.js
    │   └── transaction.controller.js
    │
    ├── middleware/
    │   └── auth.middleware.js
    │
    ├── models/
    │   ├── user.model.js
    │   ├── account.model.js
    │   ├── transaction.model.js
    │   ├── ledger.model.js
    │   └── blackList.model.js
    │
    ├── routes/
    │   ├── auth.routes.js
    │   ├── account.routes.js
    │   └── transaction.routes.js
    │
    └── services/
        └── email.service.js
```

---

## 🚀 Getting Started

### Prerequisites

Make sure you have:

* Node.js 18+
* npm
* MongoDB
* MongoDB replica set or MongoDB Atlas

MongoDB transactions require transaction support through a replica set deployment.

---

### 1. Clone the repository

```bash
git clone https://github.com/ankurdotio/backend-ledger.git

cd backend-ledger
```

### 2. Install dependencies

```bash
npm install
```

### 3. Configure environment variables

Create a `.env` file:

```env
DATABASE_URL=mongodb://localhost:27017/ledger-core

JWT_SECRET=your-super-secret-jwt-key

EMAIL_USER=your-email@gmail.com
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REFRESH_TOKEN=your-google-refresh-token
```

### 4. Start development server

```bash
npm run dev
```

### 5. Start production server

```bash
npm start
```

The API runs on:

```text
http://localhost:3000
```

---

# 📡 API

## Authentication

| Method | Endpoint             | Description             |
| ------ | -------------------- | ----------------------- |
| `POST` | `/api/auth/register` | Register a user         |
| `POST` | `/api/auth/login`    | Authenticate user       |
| `GET`  | `/api/auth/logout`   | Logout and revoke token |

## Accounts

| Method | Endpoint                           | Description         |
| ------ | ---------------------------------- | ------------------- |
| `POST` | `/api/accounts`                    | Create an account   |
| `GET`  | `/api/accounts`                    | List user accounts  |
| `GET`  | `/api/accounts/balance/:accountId` | Get account balance |

## Transactions

| Method | Endpoint                                 | Description        |
| ------ | ---------------------------------------- | ------------------ |
| `POST` | `/api/transactions`                      | Transfer funds     |
| `POST` | `/api/transactions/system/initial-funds` | Seed initial funds |

---

## Example Transfer

```http
POST /api/transactions
Authorization: Bearer <token>
Idempotency-Key: transfer-12345
Content-Type: application/json
```

```json
{
  "senderAccountId": "sender-account-id",
  "receiverAccountId": "receiver-account-id",
  "amount": 100
}
```

The transfer creates:

```text
Transaction
     │
     ├── DEBIT  → Sender
     │
     └── CREDIT → Receiver
```

Both ledger entries are committed atomically.

---

## 🧪 Testing

Add automated tests before considering the system production-ready.

Recommended test coverage:

```text
Authentication
├── Registration
├── Login
├── Logout
└── Invalid credentials

Accounts
├── Account creation
├── Account ownership
└── Balance calculation

Transactions
├── Successful transfer
├── Insufficient balance
├── Invalid account
├── Duplicate idempotency key
├── Concurrent transfers
└── Transaction rollback

Ledger
├── Debit/Credit symmetry
├── Ledger immutability
└── Balance reconciliation
```

---

## 🔎 Financial Integrity

A key design principle of the project is that financial operations should be:

* **Atomic**
* **Consistent**
* **Auditable**
* **Idempotent**
* **Immutable**

A useful invariant is:

```text
For every completed transfer:

Total Debits = Total Credits
```

If this invariant ever fails, the system should be treated as inconsistent and investigated.

---

## 🛡️ Security Considerations

Before using this project for real financial workloads, additional hardening is recommended:

* Request validation with Zod/Joi
* Rate limiting
* CSRF protection for cookie authentication
* Secure cookie configuration
* CORS policy
* Structured logging
* Audit logging
* Secret management
* Input sanitization
* Automated security testing
* Integration tests
* Ledger reconciliation jobs
* Monitoring and alerting

This repository should be considered an engineering/learning implementation rather than a certified banking system.

---

## 🗺️ Roadmap

Potential improvements:

* [ ] OpenAPI / Swagger documentation
* [ ] Automated test suite
* [ ] Request validation
* [ ] Rate limiting
* [ ] Transaction reconciliation
* [ ] Transaction reversal/refund support
* [ ] Multi-currency accounts
* [ ] Pagination for ledger history
* [ ] Audit logs
* [ ] Structured application logging
* [ ] Docker support
* [ ] CI/CD pipeline
* [ ] Health checks
* [ ] Metrics and observability
* [ ] Webhook notifications
* [ ] Refresh-token rotation

---

## 🎯 Why This Project?

This project explores backend engineering problems that appear in real financial systems:

* How do you prevent duplicate payments?
* How do you guarantee atomic money movement?
* How do you model financial transactions?
* How can an account balance be reconstructed?
* How do you maintain an immutable audit trail?
* What happens when a transfer fails halfway through?
* How should APIs behave when clients retry requests?

The goal is to build a strong foundation for understanding **financial systems, distributed systems, database transactions, and backend architecture**.

---

## 📄 License

ISC
