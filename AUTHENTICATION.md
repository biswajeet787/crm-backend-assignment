# Authentication and Authorization

This document describes the authentication behavior implemented by the CRM API. Routes in this guide use the `/api` prefix, for example `POST /api/auth/login`.

## Session model

The API uses two credentials:

| Credential | Purpose | Lifetime and handling |
| --- | --- | --- |
| Access token | Authorizes protected API requests using `Authorization: Bearer <token>`. | A signed JWT. Its lifetime is set by `ACCESS_TOKEN_TTL` and defaults to `2h`. |
| Refresh token | Obtains a replacement access token after the access token expires. | A random token held in the `crm_refresh` HTTP-only cookie. It expires after 7 days and is rotated whenever `/refresh` succeeds. |

The refresh token is not returned in the JSON response. The server stores an HMAC hash of it, rather than the raw token. Each user record has one active refresh-token hash, so signing in again replaces the prior refresh session for that account.

Protected requests must include a valid access token:

http
Authorization: Bearer <accessToken>

For each protected request, the API verifies the token, loads its user from MongoDB, and rejects missing or inactive accounts. The user's current role is loaded from the database for permission checks.

## Authentication endpoints

| Method and path | Access | Behavior |
| --- | --- | --- |
| `POST /api/auth/register` | Public | Creates an account and starts a session. Returns `201`. |

| `POST /api/auth/login` | Public | Checks email and password, then starts a session. |

| `POST /api/auth/refresh` | Refresh cookie | Rotates the refresh token and returns a new access token. |

| `POST /api/auth/logout` | Bearer token with `auth:logout` permission | Invalidates the refresh session when its cookie is present, then clears the cookie. |

| `GET /api/auth/me` | Bearer token with `auth:read` permission | Returns the authenticated user's profile. |

Registration and login require JSON request bodies. Email addresses are normalized to lowercase. Registration requires a name (2-120 characters), email, ten-digit phone number, and password (8-72 characters). Login requires an email and a non-empty password (up to 72 characters). Unknown request fields are rejected.

### Register

Example request body:

```json

{
  "name": "Alex Example",
  "email": "alex@example.com",
  "phone": "5551234567",
  "password": "replace-with-a-password"
}

The response includes `data.user` and `data.accessToken`; the refresh token is set as a cookie. Passwords are hashed before they are stored.

**Role assignment behavior:** registration defaults to `sales_executive`, but the current public registration schema also accepts an optional `role` field with any supported role (`admin`, `sales_manager`, or `sales_executive`). Because registration is unauthenticated, a caller can currently request an elevated role. For deployments where public sign-up must not grant roles, restrict registration to `sales_executive` on the server and use the admin user-management route or the admin seed script to create privileged accounts.

### Login

```http
POST /api/auth/login
Content-Type: application/json

{
  "email": "alex@example.com",
  "password": "replace-with-a-password"
}

Invalid credentials return `401`. An inactive account returns `403`. On success, the JSON response contains the user profile and access token, and the response sets the refresh cookie.

### Refresh a session

Send the `crm_refresh` cookie to `POST /api/auth/refresh`. Browsers send it automatically when cookie and CORS settings permit; API clients such as Postman should retain and send the cookie. A successful refresh returns a new access token and replaces the cookie with a newly rotated refresh token. A missing, expired, invalid, or inactive refresh session returns `401`.

### Logout

Send the access token and the `crm_refresh` cookie to `POST /api/auth/logout`. The API invalidates the stored refresh-token hash when a matching cookie is present, clears the cookie, and returns a success response. The current access token remains cryptographically valid until expiration, but protected requests are rejected if the account is inactive.

### Read the current profile

Send a bearer access token to `GET /api/auth/me`. The response contains the current user profile, including role, active state, manager, and timestamps. Password and refresh-token data are not included in the profile response.

## Refresh cookie and browser clients

The cookie is named `crm_refresh` and has these attributes:

- `HttpOnly`, so browser JavaScript cannot read the token.
- `Secure` when `NODE_ENV=production`.
- `SameSite=Lax`.
- Path `/api/auth` and a 7-day maximum age.

The Express CORS configuration allows credentials. Set `CORS_ORIGIN` to the exact frontend origin or comma-separated origins. Browser clients must include credentials on requests that need the refresh cookie; for example, use `credentials: "include"` with `fetch`.

## Roles and permissions

The API supports three roles. Permission checks are performed by route and some services also restrict access to records the user is allowed to see.

| Role | Current permission summary |
| --- | --- |
| `admin` | All configured permissions, including user management, lead assignment and conversion, CRM records, and team analytics. |

| `sales_manager` | Lead, customer, deal, activity, and timeline operations; summary, pipeline, and team-performance analytics. User-management permissions are not enabled. |

| `sales_executive` | Customer, deal, activity, timeline, and standard analytics permissions; can create and update leads and change lead status. The dedicated assigned-leads endpoint is available to this role. User-management, lead assignment/conversion, and team-performance permissions are not enabled. |

An authenticated request with insufficient permission returns `403`. A missing or invalid access token returns `401`.

## Token configuration and rate limits

Set these variables in `source-code/.env`:

| Variable | Requirement |
| --- | --- |
| `JWT_SECRET` | At least 32 characters; signs access tokens. |
| `JWT_REFRESH_SECRET` | At least 32 characters; HMAC-hashes refresh tokens. Use a different value from `JWT_SECRET`. |
| `ACCESS_TOKEN_TTL` | Number plus unit (`s`, `m`, `h`, or `d`); defaults to `2h`. |

The register, login, and refresh endpoints are limited to 10 requests per 15 minutes per client IP. The Express app also applies a global limit of 100 requests per 15 minutes per client IP.

## Response shapes

Successful authentication responses use the standard success envelope:

```json
{
  "success": true,
  "message": "Signed in",
  "data": {
    "user": { "id": "...", "role": "sales_executive" },
    "accessToken": "<accessToken>"
  }
}

Errors use `{ "success": false, "message": "..." }`, with `details` included when available. Common status codes are `400` for invalid request data, `401` for missing or invalid credentials, `403` for inactive accounts or insufficient permission, `409` for duplicate values such as an existing email, and `429` when a rate limit is exceeded.
