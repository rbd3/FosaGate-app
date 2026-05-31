# FosaGate Backend Implementation Guide

> **Purpose:** Step-by-step spec for building the FosaGate backend server. An agent implementing this should follow each section in order and not deviate from the patterns described here.

---

## Stack

| Tool | Version | Why |
|------|---------|-----|
| Node.js | 20+ LTS | Runtime |
| Express | 5.x | HTTP framework |
| TypeScript | 5.x | Type safety |
| viem | 2.x | Ethereum interaction, SIWE verification, EIP-1271 support |
| express-session | latest | Server-side session management |
| firebase-admin | latest | Firebase SDK for Firestore Database and Notifications |
| connect-session-firestore | latest | Firestore-backed session store for production |
| helmet | latest | Security headers |
| cors | latest | CORS policy |
| zod | latest | Request validation |
| dotenv | latest | Environment config |
| tsx | latest | Dev runner (no build step needed during dev) |

---

## Project Location

Create the backend as a sibling to the frontend:

```
FosaGate AI/
├── FosaGate-app/          # Existing Vite frontend
└── FosaGate-server/       # New backend (this guide)
```

Do NOT nest the server inside `FosaGate-app/`.

---

## Directory Structure

```
FosaGate-server/
├── src/
│   ├── index.ts                  # Express app entry point
│   ├── config.ts                 # Environment variables + validation
│   ├── routes/
│   │   ├── auth.siwe.ts          # SIWE nonce + verify
│   │   ├── auth.session.ts       # Session check + logout
│   │   ├── wallets.ts            # Link/unlink wallets
│   │   ├── agents.ts             # Agent CRUD + delegation + revoke
│   │   └── notifications.ts      # Notifications fetch + read status
│   ├── middleware/
│   │   ├── session.ts            # express-session config with Firestore
│   │   ├── csrf.ts               # CSRF token middleware
│   │   ├── requireAuth.ts        # Require authenticated session
│   │   └── requireWallet.ts      # Require wallet-linked session
│   ├── services/
│   │   ├── siwe.ts               # SIWE message creation + verification
│   │   ├── nonce.ts              # Nonce generation + storage + validation
│   │   ├── user.ts               # User account CRUD (Firestore)
│   │   ├── delegation.ts         # Delegation policy storage + validation (Firestore)
│   │   └── notification.ts       # Notification CRUD + trigger (Firestore)
│   ├── db/
│   │   └── firebase.ts           # Firebase Admin SDK initialization & Firestore client
│   └── types/
│       └── index.ts              # Shared TypeScript types
├── package.json
├── tsconfig.json
└── .env.example
```

---

## Phase 1: Project Init

### package.json

```json
{
  "name": "fosagate-server",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "start": "node --import tsx src/index.ts",
    "typecheck": "tsc --noEmit"
  }
}
```

### Dependencies to install

```bash
npm install express viem express-session helmet cors zod dotenv firebase-admin connect-session-firestore
npm install -D typescript tsx @types/express @types/express-session @types/cors
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "resolveJsonModule": true
  },
  "include": ["src"]
}
```

### .env.example

```env
PORT=3001
NODE_ENV=development
SESSION_SECRET=replace-with-64-char-random-hex
FRONTEND_ORIGIN=http://localhost:5173
ARBITRUM_RPC_URL=https://arb1.arbitrum.io/rpc
# Firebase Credentials (JSON string or path to file. Alternatively, let ADC handle it in GCP)
FIREBASE_PROJECT_ID=fosagate-ai
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxxxx@fosagate-ai.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQC..."
```

---

## Phase 2: App Entry Point

### `src/config.ts`

Load and validate all env vars at startup using zod. Export a typed `config` object. If any required var is missing, crash immediately with a clear error.

Required vars:
- `PORT` (number, default 3001)
- `NODE_ENV` (enum: development | production)
- `SESSION_SECRET` (string, min 32 chars)
- `FRONTEND_ORIGIN` (string URL)
- `ARBITRUM_RPC_URL` (string)
- `FIREBASE_PROJECT_ID` (string)
- `FIREBASE_CLIENT_EMAIL` (string)
- `FIREBASE_PRIVATE_KEY` (string)

### `src/index.ts`

Wire up Express in this exact order:

```typescript
// 1. Load config (crash early if invalid)
// 2. Create Express app
// 3. Trust proxy (for secure cookies behind reverse proxy)
//    app.set('trust proxy', 1)
// 4. Apply helmet()
// 5. Apply cors({ origin: config.FRONTEND_ORIGIN, credentials: true })
// 6. Apply express.json({ limit: '100kb' })
// 7. Apply session middleware (see middleware/session.ts)
// 8. Apply CSRF middleware on state-changing routes
// 9. Mount routes:
//    /auth/siwe      → auth.siwe router
//    /auth           → auth.session router
//    /wallets        → wallets router
//    /agents         → agents router
//    /notifications  → notifications router
// 10. Global error handler
// 11. Listen on config.PORT
```

Log `Server running on port ${PORT}` on successful start. Nothing else.

---

## Phase 3: Session Middleware

### `src/middleware/session.ts`

Export a configured `express-session` middleware.

Session config:

```typescript
{
  secret: config.SESSION_SECRET,
  name: 'fosagate.sid',           // Custom cookie name (not default 'connect.sid')
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,               // JavaScript cannot read it
    secure: config.NODE_ENV === 'production',
    sameSite: 'strict',           // Strict for maximum CSRF protection
    maxAge: 24 * 60 * 60 * 1000,  // 24 hours absolute lifetime
    domain: undefined,            // Let browser infer
    path: '/',
  },
  store: redisStore || memoryStore  // Redis in prod, MemoryStore in dev
}
```

**Dev mode:** Use the default MemoryStore with a console warning: `"⚠ Using in-memory sessions. Do not use in production."`.

**Production:** Use `connect-redis` with `ioredis`.

### `src/middleware/requireAuth.ts`

Check `req.session.userId`. If missing, respond `401 { error: 'Not authenticated' }`. Otherwise call `next()`.

### `src/middleware/requireWallet.ts`

Check `req.session.walletAddress`. If missing, respond `403 { error: 'Wallet not connected' }`. Otherwise call `next()`.

### `src/middleware/csrf.ts`

For **Phase 1 (buildathon)**: Use a simple double-submit cookie pattern.
- On session creation, generate a random CSRF token and store it in the session.
- Expose `GET /auth/session` which returns the CSRF token in the response body.
- On every POST/PUT/DELETE request, check the `x-csrf-token` header matches `req.session.csrfToken`.
- If mismatch, respond `403 { error: 'Invalid CSRF token' }`.

---

## Phase 4: SIWE Authentication

### `src/services/nonce.ts`

```typescript
// In-memory Map<string, { createdAt: number }> for dev.
// Redis SET with TTL for production.

generateNonce(): string
  // Return crypto.randomUUID() — 36 chars, cryptographically random.
  // Store in nonce map with createdAt = Date.now().

consumeNonce(nonce: string): boolean
  // Check if nonce exists and is < 5 minutes old.
  // If valid: delete it (single-use) and return true.
  // If expired or missing: return false.
```

Run a cleanup interval every 60 seconds to purge expired nonces (dev mode only; Redis TTL handles this automatically).

### `src/services/siwe.ts`

```typescript
import { createPublicClient, http } from 'viem'
import { arbitrum } from 'viem/chains'
import { parseSiweMessage, validateSiweMessage } from 'viem/siwe'

verifySiweMessage(message: string, signature: string, expectedNonce: string, expectedDomain: string): Promise<{ address: string, chainId: number }>

  // 1. Parse the SIWE message using viem's parseSiweMessage.
  // 2. Validate:
  //    - parsed.domain === expectedDomain
  //    - parsed.nonce === expectedNonce
  //    - parsed.expirationTime is in the future (or not set)
  //    - parsed.notBefore is in the past (or not set)
  //    - parsed.chainId is an allowed chain (42161 for Arbitrum One, 421614 for Sepolia)
  // 3. Verify signature using viem's validateSiweMessage.
  //    viem handles both EOA (ecrecover) and EIP-1271 (smart contract) verification automatically
  //    when you pass a publicClient.
  // 4. Return { address, chainId }.
  // 5. On any failure, throw a descriptive error.
```

### `src/routes/auth.siwe.ts`

Two endpoints:

#### `GET /auth/siwe/nonce`

```
Response 200: { nonce: string }
```

Generate a nonce via `nonceService.generateNonce()` and return it.

#### `POST /auth/siwe/verify`

```
Request body (validate with zod):
{
  message: string,    // Raw SIWE message string
  signature: string   // Hex signature from wallet
}

Validation:
  - message: non-empty string
  - signature: starts with '0x', length 132 (65 bytes hex)
```

Handler logic:

```
1. Extract nonce from the parsed SIWE message.
2. Consume the nonce via nonceService.consumeNonce(). If invalid → 401.
3. Validate the Origin header matches the SIWE domain. If mismatch → 403.
4. Verify the SIWE message via siweService.verifySiweMessage(). If fails → 401.
5. Look up or create a user account by wallet address via userService.
6. Set session:
   req.session.userId = user.id
   req.session.walletAddress = address
   req.session.loginMethod = 'siwe'
   req.session.csrfToken = crypto.randomUUID()
7. Respond 200: { user: { id, walletAddress }, csrfToken: req.session.csrfToken }
```

---

## Phase 5: Session Routes

### `src/routes/auth.session.ts`

#### `GET /auth/session`

No auth required.

```
If req.session.userId exists:
  Response 200: {
    authenticated: true,
    user: { id, walletAddress, loginMethod },
    csrfToken: req.session.csrfToken
  }
Else:
  Response 200: { authenticated: false }
```

Always return 200. Let the frontend decide what to do.

#### `POST /auth/logout`

Requires auth (use `requireAuth` middleware).

```
1. Destroy the session: req.session.destroy()
2. Clear the cookie: res.clearCookie('fosagate.sid')
3. Respond 200: { ok: true }
```

---

## Phase 6: Wallet Management

### `src/routes/wallets.ts`

All routes require `requireAuth`.

#### `POST /wallets/link`

Links an additional wallet to an existing account. Used when a user logged in via OIDC wants to connect a wallet.

```
Request body:
{
  message: string,     // SIWE message
  signature: string    // Wallet signature
}

Logic:
1. Verify the SIWE message (same as login flow).
2. Check the wallet isn't already linked to another account.
3. Add the wallet address to the user's linked wallets.
4. Update session: req.session.walletAddress = address
5. Respond 200: { linked: true, address }
```

#### `DELETE /wallets/:address`

Unlinks a wallet from the account.

```
Logic:
1. Validate :address is a valid Ethereum address.
2. Ensure user has at least one other login method (wallet or OIDC) remaining.
   Never let a user remove their last auth method.
3. Remove the wallet from user's linked wallets.
4. If the removed wallet was the current session wallet, clear req.session.walletAddress.
5. Respond 200: { unlinked: true }
```

---

## Phase 7: Agent Management

### `src/routes/agents.ts`

All routes require `requireAuth` AND `requireWallet`.

#### `POST /agents`

Create a new agent profile.

```
Request body:
{
  name: string,           // Human-readable agent name
  agentAddress: string    // Ethereum address the agent will sign from
}

Logic:
1. Validate agentAddress is a valid Ethereum address.
2. Validate agentAddress is not the user's own wallet.
3. Create agent record linked to user.
4. Respond 201: { agent: { id, name, agentAddress, status: 'pending_delegation' } }
```

#### `POST /agents/:id/delegations`

Store a signed EIP-712 delegation for an agent.

```
Request body:
{
  delegation: {
    allowedContracts: string[],    // Target contract addresses
    allowedSelectors: string[],    // Function selector bytes4
    maxValuePerTx: string,         // Wei string
    dailySpendLimit: string,       // Wei string
    expiry: number,                // Unix timestamp
    maxRiskScore: number           // 0-1000
  },
  signature: string                // User's EIP-712 signature over the delegation
}

Logic:
1. Validate the agent belongs to the requesting user.
2. Validate the EIP-712 signature against the delegation struct using viem's verifyTypedData.
3. Validate expiry is in the future and not more than 30 days out.
4. Store the delegation with status 'active'.
5. Respond 201: { delegation: { id, ...params, status: 'active' } }
```

#### `POST /agents/:id/revoke`

Revoke a specific agent's delegation.

```
Logic:
1. Validate the agent belongs to the requesting user.
2. Mark all active delegations for this agent as 'revoked'.
3. Respond 200: { revoked: true }
```

#### `POST /agents/revoke-all`

Emergency: revoke ALL agent delegations for the user.

```
Logic:
1. Mark all active delegations for all user's agents as 'revoked'.
2. Respond 200: { revoked: true, count: N }
```

#### `GET /agents/:id/delegations`

View active delegations.

```
Logic:
1. Validate the agent belongs to the requesting user.
2. Return all delegations (active and revoked) for the agent.
3. Respond 200: { delegations: [...] }
```

---

## Phase 8: Firebase Data Layer

Instead of in-memory maps or PostgreSQL, FosaGate utilizes **Google Cloud Firestore** (via the `firebase-admin` SDK) for persistent state. This also powers real-time user notifications.

### `src/db/firebase.ts`

Initialize the Firebase Admin SDK using variables defined in `config.ts`:

```typescript
import admin from 'firebase-admin';
import { config } from '../config.js';

// Parse private key newlines correctly from env
const privateKey = config.FIREBASE_PRIVATE_KEY.replace(/\\n/g, '\n');

if (!admin.apps.length) {
  admin.initializeApp({
    credential: admin.credential.cert({
      projectId: config.FIREBASE_PROJECT_ID,
      clientEmail: config.FIREBASE_CLIENT_EMAIL,
      privateKey: privateKey,
    }),
  });
}

export const db = admin.firestore();
export const auth = admin.auth();
```

---

## Phase 9: Database Schemas & Services (Firestore)

Create the corresponding services to interact with Firestore collections.

### Firestore Collections

#### `users`
- Document ID: `userId` (string)
- Fields:
  - `wallets`: `string[]` (array of checksummed Ethereum addresses)
  - `oidcSubject`: `string | null`
  - `oidcProvider`: `string | null`
  - `createdAt`: `admin.firestore.Timestamp`

#### `agents`
- Document ID: `agentId` (string)
- Fields:
  - `userId`: `string` (owner user ID)
  - `name`: `string`
  - `agentAddress`: `string` (checksummed agent signing address)
  - `status`: `'pending_delegation' | 'active' | 'suspended'`
  - `createdAt`: `admin.firestore.Timestamp`

#### `delegations`
- Document ID: `delegationId` (string)
- Fields:
  - `agentId`: `string`
  - `userId`: `string`
  - `allowedContracts`: `string[]`
  - `allowedSelectors`: `string[]`
  - `maxValuePerTx`: `string` (bigint serialized to string)
  - `dailySpendLimit`: `string` (bigint serialized to string)
  - `expiry`: `number` (Unix timestamp)
  - `maxRiskScore`: `number`
  - `signature`: `string`
  - `status`: `'active' | 'revoked' | 'expired'`
  - `createdAt`: `admin.firestore.Timestamp`

### `src/services/user.ts`

```typescript
import { db } from '../db/firebase.js';
import { getAddress } from 'viem';

export async function findOrCreateByWallet(address: string) {
  const checksummed = getAddress(address);
  const usersRef = db.collection('users');
  const snapshot = await usersRef.where('wallets', 'array-contains', checksummed).limit(1).get();

  if (!snapshot.empty) {
    const doc = snapshot.docs[0];
    return { id: doc.id, ...doc.data() };
  }

  // Create new user
  const newUserRef = usersRef.doc();
  const userData = {
    wallets: [checksummed],
    oidcSubject: null,
    oidcProvider: null,
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
  };
  await newUserRef.set(userData);
  return { id: newUserRef.id, ...userData };
}

export async function findById(id: string) {
  const doc = await db.collection('users').doc(id).get();
  return doc.exists ? { id: doc.id, ...doc.data() } : null;
}

export async function linkWallet(userId: string, address: string) {
  const checksummed = getAddress(address);
  
  // 1. Verify not already linked
  const collision = await db.collection('users').where('wallets', 'array-contains', checksummed).limit(1).get();
  if (!collision.empty) {
    throw new Error('Wallet already linked to another account');
  }

  // 2. Perform array union update
  await db.collection('users').doc(userId).update({
    wallets: admin.firestore.FieldValue.arrayUnion(checksummed)
  });
}

export async function unlinkWallet(userId: string, address: string) {
  const checksummed = getAddress(address);
  const user = await findById(userId);
  if (!user) throw new Error('User not found');
  
  if (user.wallets.length <= 1 && !user.oidcSubject) {
    throw new Error('Cannot unlink last login method');
  }

  await db.collection('users').doc(userId).update({
    wallets: admin.firestore.FieldValue.arrayRemove(checksummed)
  });
}
```

---

## Phase 10: Real-Time Notifications

Firestore is perfectly designed for real-time applications because frontends can establish **listeners** directly on Firestore documents. The backend server writes notifications to Firestore, and the client receives them instantly.

1. The Core Notification Matrix (What to send)
🔴 Critical Alerts (Immediate action required)
Agent Transaction Blocked:
Message: "Agent Alpha attempted execution on suspicious contract 0x... and was blocked."
Why: Indicates potential agent compromise, MEV vulnerability, or an exploit attempt.
Delegation Revoked (Emergency):
Message: "All active delegations for Agent Beta have been revoked by [Wallet Address]."
Why: Confirmation of a critical security action.
Agent Suspension:
Message: "Agent Gamma has been suspended by FosaGate due to repeated policy violations."
Why: The protocol itself blocked the agent due to high risk.
🟡 Warnings (Requires attention soon)
Spend Limit Approaching:
Message: "Agent Alpha has consumed 80% of its daily spend limit ($3,200 / $4,000)."
Why: Prevents the agent from failing mid-execution later in the day.
Delegation Expiring Soon:
Message: "Delegation for Agent Beta expires in 24 hours. Click here to renew."
Why: Prompts the user to sign a new EIP-712 message before the agent stops working.
Low Gas Balance:
Message: "Agent Gamma address (0x...) has less than 0.005 ETH left for gas."
Why: The agent needs gas to submit transactions on-chain.

### `notifications` Collection Schema

- Document ID: `notificationId` (string)
- Fields:
  - `userId`: `string` (recipient)
  - `type`: `'agent_rejected' | 'limit_warning' | 'delegation_revoked' | 'delegation_created' | 'agent_suspended' | 'security_event'`
  - `title`: `string`
  - `message`: `string`
  - `severity`: `'critical' | 'warning' | 'info'`
  - `read`: `boolean` (default: `false`)
  - `metadata`: `object` (optional context like `agentAddress`, `txHash`, or `riskScore`)
  - `createdAt`: `admin.firestore.Timestamp`

### `src/services/notification.ts`

Use this service to create user alerts. The database handles distributing them to the client instantly.

```typescript
import { db } from '../db/firebase.js';
import admin from 'firebase-admin';

export interface NotificationParams {
  userId: string;
  type: string;
  title: string;
  message: string;
  severity: 'critical' | 'warning' | 'info';
  metadata?: Record<string, any>;
}

export async function triggerNotification(params: NotificationParams) {
  const notificationRef = db.collection('notifications').doc();
  const notificationData = {
    ...params,
    read: false,
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
  };
  await notificationRef.set(notificationData);
  return { id: notificationRef.id, ...notificationData };
}

export async function getUserNotifications(userId: string, limitVal = 20) {
  const snapshot = await db.collection('notifications')
    .where('userId', '==', userId)
    .orderBy('createdAt', 'desc')
    .limit(limitVal)
    .get();

  return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}

export async function markAsRead(notificationId: string, userId: string) {
  const ref = db.collection('notifications').doc(notificationId);
  const doc = await ref.get();
  if (!doc.exists) throw new Error('Notification not found');
  if (doc.data()?.userId !== userId) throw new Error('Unauthorized');
  
  await ref.update({ read: true });
}

export async function markAllAsRead(userId: string) {
  const batch = db.batch();
  const unreadSnapshot = await db.collection('notifications')
    .where('userId', '==', userId)
    .where('read', '==', false)
    .get();

  unreadSnapshot.docs.forEach(doc => {
    batch.update(doc.ref, { read: true });
  });

  await batch.commit();
}
```

### `src/routes/notifications.ts`

All routes require `requireAuth`.

```typescript
import { Router } from 'express';
import { requireAuth } from '../middleware/requireAuth.js';
import * as notificationService from '../services/notification.js';
import { asyncHandler } from '../utils.js';

const router = Router();

// GET /notifications - Fetch user's notification history
router.get('/', requireAuth, asyncHandler(async (req, res) => {
  const list = await notificationService.getUserNotifications(req.session.userId!);
  res.json({ notifications: list });
}));

// POST /notifications/:id/read - Mark notification as read
router.post('/:id/read', requireAuth, asyncHandler(async (req, res) => {
  await notificationService.markAsRead(req.params.id, req.session.userId!);
  res.json({ success: true });
}));

// POST /notifications/read-all - Mark all as read
router.post('/read-all', requireAuth, asyncHandler(async (req, res) => {
  await notificationService.markAllAsRead(req.session.userId!);
  res.json({ success: true });
}));

export default router;
```

---

## EIP-712 Delegation Type

Use this typed data structure for agent delegations. Both frontend and backend must use the same schema.

```typescript
const DELEGATION_TYPES = {
  AgentDelegation: [
    { name: 'agentAddress', type: 'address' },
    { name: 'allowedContracts', type: 'address[]' },
    { name: 'allowedSelectors', type: 'bytes4[]' },
    { name: 'maxValuePerTx', type: 'uint256' },
    { name: 'dailySpendLimit', type: 'uint256' },
    { name: 'expiry', type: 'uint256' },
    { name: 'maxRiskScore', type: 'uint256' },
    { name: 'nonce', type: 'uint256' },
  ],
} as const

const DELEGATION_DOMAIN = {
  name: 'FosaGate',
  version: '1',
  chainId: 42161,                    // Arbitrum One
  verifyingContract: '0x...',        // FosaGateRouter address (set after deploy)
  salt: '0x...',                     // Unique salt
} as const
```

---

## Error Handling

### Global error handler

```typescript
app.use((err, req, res, next) => {
  // 1. Log error (never log stack traces in production response).
  // 2. If err is a ZodError → 400 { error: 'Validation error', details: formatted issues }
  // 3. If err has a statusCode property → use it.
  // 4. Otherwise → 500 { error: 'Internal server error' }
})
```

### Route-level errors

Wrap all async route handlers with a helper that catches rejections:

```typescript
const asyncHandler = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next)
```

Every route handler must use `asyncHandler`.

---

## Security Checklist

These are non-negotiable. Verify each one before shipping.

- [ ] `helmet()` applied before any routes
- [ ] CORS restricted to `FRONTEND_ORIGIN` only
- [ ] Session cookie: `httpOnly: true`, `secure: true` (prod), `sameSite: 'strict'`
- [ ] Custom cookie name (not `connect.sid`)
- [ ] CSRF token validated on all POST/PUT/DELETE
- [ ] SIWE nonces: single-use, 5-min TTL, server-generated
- [ ] Origin header validated against SIWE domain
- [ ] No tokens in `localStorage` — sessions only
- [ ] Request body size limited (100kb)
- [ ] `trust proxy` set for production reverse proxy
- [ ] All user input validated with zod before use
- [ ] Wallet addresses checksummed with `viem/getAddress` before storage/comparison

---

## API Summary

```
GET    /auth/siwe/nonce          → { nonce }
POST   /auth/siwe/verify         → { user, csrfToken }
GET    /auth/session             → { authenticated, user?, csrfToken? }
POST   /auth/logout              → { ok }

POST   /wallets/link             → { linked, address }
DELETE /wallets/:address         → { unlinked }

POST   /agents                   → { agent }
POST   /agents/:id/delegations   → { delegation }
GET    /agents/:id/delegations   → { delegations[] }
POST   /agents/:id/revoke        → { revoked }
POST   /agents/revoke-all        → { revoked, count }

GET    /notifications            → { notifications[] }
POST   /notifications/:id/read   → { success }
POST   /notifications/read-all   → { success }
```

---

## Implementation Order

Follow this order exactly. Do not skip ahead.

```
Phase 1: npm init, install deps, tsconfig, .env with Firebase secrets
Phase 2: config.ts + index.ts (bare Express with helmet + cors + json)
Phase 3: session middleware (configured with connect-session-firestore or memory in dev)
Phase 4: Firebase client initialization (db/firebase.ts)
Phase 5: User database models & CRUD services (services/user.ts)
Phase 6: SIWE nonce + verify routes & siwe services using Firestore
Phase 7: session check + logout routes
Phase 8: wallet link/unlink routes
Phase 9: agent + delegation routes & services using Firestore
Phase 10: Notifications services (services/notification.ts) + routing (routes/notifications.ts)
```

After each phase, verify the server starts and the new endpoints respond correctly. Do not proceed to the next phase if the current one has errors.

---

## Testing During Dev

Use `curl` or the frontend to verify each endpoint. Example test sequence:

```bash
# 1. Get nonce
curl http://localhost:3001/auth/siwe/nonce

# 2. Check session (should be unauthenticated)
curl -c cookies.txt http://localhost:3001/auth/session

# 3. After frontend signs SIWE message, verify
curl -X POST http://localhost:3001/auth/siwe/verify \
  -H "Content-Type: application/json" \
  -b cookies.txt -c cookies.txt \
  -d '{"message":"...","signature":"0x..."}'

# 4. Check session (should be authenticated now)
curl -b cookies.txt http://localhost:3001/auth/session
```

---

## What This Backend Does NOT Handle (Yet)

These are deferred to later phases. Do not build them now.

- OIDC / password login (deferred — see AUTHENTICATION_GUIDE.md)
- BFF proxy for OIDC tokens (deferred — needed only when OIDC is added)
- Rate limiting (add after core auth works)
- On-chain contract interaction (FosaGateRouter calls are made by the frontend + agent, not this server)

---

## References

- Express 5: https://expressjs.com/
- viem SIWE: https://viem.sh/docs/siwe/utilities/parseSiweMessage
- viem EIP-712: https://viem.sh/docs/actions/public/verifyTypedData
- express-session: https://github.com/expressjs/session
- connect-redis: https://github.com/tj/connect-redis
- AUTHENTICATION_GUIDE.md: Sibling file in FosaGate-app/
- ARCHITECTURE.md: Parent directory
