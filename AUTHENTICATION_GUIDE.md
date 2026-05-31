# FosaGate App Authentication Guide

## Goal

FosaGate should support two user-friendly login choices:

- **Wallet login first:** best default for a Web3 app on Arbitrum.
- **Email/password login later:** handled by a modern OAuth 2.0/OIDC identity provider using OAuth 2.1-style security practices.

Important distinction: **OpenID Connect authenticates users**. OAuth authorizes access. For password login, use OIDC on top of OAuth.

## Recommended Model

Use one internal FosaGate account with multiple linked identities:

```text
FosaGate user account
  - wallet identities: one or more Ethereum/Arbitrum addresses
  - oidc identity: provider + subject id
  - agents: authorized agent addresses/session keys
```

This lets a user start with a wallet or with password, then link the other method later.

## Login Option 1: Wallet

Use **Sign-In with Ethereum (SIWE / ERC-4361)**.

Flow:

1. Frontend discovers wallets using **EIP-6963** (not `window.ethereum`) and asks backend for a nonce.
2. User connects wallet and signs a SIWE message.
3. Backend verifies domain, chain id, nonce, expiration, and signature.
   - For EOA wallets: verify with `ecrecover`.
   - For smart contract wallets (Safe, Kernel, etc.): verify with **EIP-1271** `isValidSignature` on the contract.
4. Backend also validates the HTTP `Origin` header matches the SIWE message `domain`.
5. Backend creates an HTTP-only secure session cookie.
6. If wallet is new, create a FosaGate user account.
7. If user is already logged in with password, link this wallet to the existing account.

SIWE implementation rules:

- Use an audited SIWE library (`viem/siwe` or the `siwe` npm package). Do not roll your own parser.
- Store nonces server-side with a short TTL (5 minutes). Each nonce must be single-use.
- Reject expired or replayed nonces.
- Support EIP-1271 signature verification so smart contract wallet users are not locked out.

Use wallet login for:

- Creating or managing agents.
- Registering agent addresses.
- Authorizing agent session keys.
- Signing high-trust actions.

## Login Option 2: Password via OAuth/OIDC (Don't do yet)

Do not build raw password auth directly in the React app. Use an OIDC provider such as Auth0, Clerk, WorkOS, Keycloak, Cognito, or another standards-compliant provider.

Use:

- Authorization Code Flow with PKCE (`S256` challenge method only; never `plain`).
- OIDC ID token for user identity.
- Short-lived access tokens.
- Refresh token rotation if refresh tokens are needed.
- Exact redirect URI matching.
- Unique `state` parameter on every authorization request to prevent CSRF on the redirect.
- Validate the `nonce` claim in the ID token to prevent token replay.
- No Implicit Flow.
- No Resource Owner Password Credentials flow.

### Backend for Frontend (BFF) pattern

Because FosaGate is a Vite SPA, **never let OAuth/OIDC tokens reach the browser**. Use a thin BFF server that acts as the confidential OAuth client:

1. The BFF handles the full OIDC flow (redirects, code exchange, token storage).
2. Access and refresh tokens live only in the BFF's server-side session store.
3. The BFF issues an HTTP-only, `Secure`, `SameSite=Strict` session cookie to the browser.
4. When the frontend needs to call a protected API, the request goes through the BFF, which attaches the access token from its server-side store.
5. The browser never sees, stores, or transmits any OAuth token.

This eliminates the entire class of token-theft-via-XSS attacks.

Flow:

1. User clicks "Continue with email".
2. Frontend redirects to the BFF login endpoint, which redirects to the OIDC provider.
3. Provider handles email/password, MFA, recovery, and breach protections.
4. Provider redirects back to the BFF callback endpoint with an authorization code.
5. BFF exchanges the code using PKCE (`S256`) and its `client_secret`.
6. BFF stores tokens in a server-side session and sets an HTTP-only secure session cookie.
7. If the user has no wallet linked, show "Connect wallet" before allowing agent creation or on-chain actions.

Password login should be used for:

- Dashboard access.
- Viewing history and analytics.
- Managing profile settings.

Password login alone should not be enough for:

- Registering an agent.
- Changing spending limits.
- Signing transaction intents.
- Executing on-chain actions.

Require wallet confirmation for those actions.

## Agent Authorization

Agents should not authenticate with the user's password session.

Use wallet-based delegation:

1. User logs in.
2. User connects wallet.
3. User creates an agent profile.
4. User signs an EIP-712 delegation for the agent address/session key.
5. Backend stores the delegation policy.
6. Agent signs each transaction intent.
7. FosaGate Evaluator verifies agent signature, nonce, deadline, and policy.
8. Evaluator signs a short-lived attestation.
9. Agent submits transaction plus attestation to `FosaGateRouter`.
10. `FosaGateRouter` enforces delegation policy on-chain before executing (see below).

Delegation policy should include:

- Allowed target contracts.
- Allowed function selectors.
- Max value per transaction.
- Daily or weekly spend limit.
- Expiration time.
- Max risk score.
- Revocation status.

### On-chain policy enforcement

Do not rely solely on the off-chain Evaluator to enforce delegation limits. If the Evaluator is compromised or bypassed, an agent could exceed its permissions.

`FosaGateRouter` (or a dedicated guard contract) must independently verify:

- The delegation has not expired (`block.timestamp <= delegation.expiry`).
- The delegation has not been revoked (check on-chain revocation mapping).
- The target contract and function selector are in the allowed set.
- The transaction value does not exceed the per-transaction cap.
- The cumulative spend does not exceed the daily/weekly on-chain spend limit.
- The agent signature is valid for the intent payload.

This creates defense in depth: the Evaluator provides fast off-chain screening and attestation, while the contract provides a hard on-chain safety net that cannot be bypassed.

The on-chain guard should expose:

```solidity
function executeWithDelegation(
    bytes calldata agentSignedIntent,
    bytes calldata evaluatorAttestation
) external;

function revokeAll() external;           // owner kills all active delegations
function revokeDelegation(bytes32 id) external;

function spentToday(address agent) external view returns (uint256);
```

### Session key lifecycle

- Session keys should be short-lived (7 days maximum).
- Renewal requires an explicit user wallet signature.
- Provide an emergency `revokeAll()` function (on-chain) and a `POST /agents/revoke-all` endpoint (off-chain) so the user can kill all active delegations instantly.
- Consider **EIP-7702** for EOA-native delegation support, allowing EOA users to adopt session-key features without deploying a new contract.

## Session And Token Rules

Frontend:

- Store app session in an HTTP-only, `Secure`, `SameSite=Lax` or `Strict` cookie.
- Do not store access tokens in `localStorage`.
- Use CSRF protection for cookie-authenticated state-changing requests.

Backend:

- Validate every SIWE nonce once only.
- Validate OIDC issuer, audience, expiry, nonce, and signature.
- Keep access tokens short-lived.
- Rotate refresh tokens.
- Log login method, linked wallet, agent delegation, and revocation events.

Agents:

- Use signed requests, not long-lived API keys.
- Include nonce and deadline in every signed intent.
- Reject replayed nonces.
- Prefer session keys with narrow permissions.

## Minimal API Surface

```text
GET  /auth/siwe/nonce
POST /auth/siwe/verify

GET  /auth/oidc/login
GET  /auth/oidc/callback

POST /auth/logout
GET  /auth/session

POST /wallets/link
DELETE /wallets/:address

POST /agents
POST /agents/:id/delegations
POST /agents/:id/revoke
```

## Implementation Order

1. Add SIWE wallet login.
2. Add server-side sessions with secure cookies.
3. Add OIDC password login with Authorization Code + PKCE.
4. Add account linking between OIDC users and wallet addresses.
5. Require wallet confirmation for agent creation and policy changes.
6. Add EIP-712 agent delegation and signed agent intents.

## Recommended Decision

For FosaGate, make **wallet login the primary path** and **OIDC password login the fallback/convenience path**.

The password account improves onboarding, but the wallet remains the security anchor for agent permissions and on-chain trust.

## References

- SIWE / ERC-4361: https://eips.ethereum.org/EIPS/eip-4361
- EIP-1271 (smart wallet signatures): https://eips.ethereum.org/EIPS/eip-1271
- EIP-6963 (wallet discovery): https://eips.ethereum.org/EIPS/eip-6963
- EIP-7702 (EOA delegation): https://eips.ethereum.org/EIPS/eip-7702
- OAuth 2.0 Security Best Current Practice, RFC 9700: https://www.rfc-editor.org/rfc/rfc9700
- OAuth 2.1 draft history, latest checked: draft 15 on 2026-03-02: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/history/
- OpenID Connect: https://openid.net/developers/how-connect-works/
- OWASP Session Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
