# Assignment Setup Guide

This guide walks through installing and running the CRM Sales Management API locally.

## Prerequisites

- An active Node.js LTS release with npm
- MongoDB and a connection URI
- PowerShell or Terminal on Windows, or a terminal on macOS/Linux

Lead conversion uses MongoDB transactions. Use a replica set or sharded MongoDB deployment for that workflow; a standalone MongoDB server can run the API but cannot convert leads.

## 1. Install dependencies

From the project root, open a terminal and run:

powershell
Set-Location .\source-code
npm install

For macOS or Linux, use `cd source-code` instead of `Set-Location`.

## 2. Configure the environment

Create a file named `.env` in the `source-code/` directory. Add the following settings and replace the sample secrets with your own unique values. Each JWT secret must be at least 32 characters, and the two secrets should differ.

dotenv

NODE_ENV=development
PORT=8000
MONGODB_URI=mongodb://127.0.0.1:27017/crm_sales_management
JWT_SECRET=replace-with-a-unique-secret-at-least-32-characters-long
JWT_REFRESH_SECRET=replace-with-a-different-unique-secret-at-least-32-characters
ACCESS_TOKEN_TTL=2h
CORS_ORIGIN=http://localhost:5173

If MongoDB is hosted remotely, replace `MONGODB_URI` with its connection string. Set `CORS_ORIGIN` to your frontend's origin, or a comma-separated list of allowed origins. Keep `.env` private; it is ignored by Git.

## 3. Start MongoDB and the API

Make sure the MongoDB deployment from `MONGODB_URI` is running. In a terminal with the current directory set to `source-code/`, start the development server:

cmd 
npm run dev

The API listens on `http://localhost:8000` by default. Change `PORT` in `.env` to use another port. The HTTP server starts listening before the database connection finishes.

## 4. Verify the API

Open a second terminal and request the health endpoint:

```powershell
Invoke-RestMethod http://localhost:8000/health
```

The response reports the API status and MongoDB connection state. Confirm that `database` is `connected` before using CRM endpoints.

## 5. Create the initial admin account

Admin seeding is optional if you only need to try public registration and standard user endpoints. To use admin-only features, add these settings to `source-code/.env`:

```dotenv
SEED_ADMIN_NAME=Your Admin Name
SEED_ADMIN_EMAIL=admin@example.com
SEED_ADMIN_PHONE=1234567890
SEED_ADMIN_PASSWORD=replace-with-a-password-at-least-12-characters-long
```

From `source-code/`, run:

```powershell
npm run seed:admin
```

The script creates one active admin account. It exits with an error if an account already exists for that email. Sign in using `POST /api/auth/login`, then remove `SEED_ADMIN_PASSWORD` from `.env` after seeding.

## 6. Send requests from Postman

The request definitions are in `api-documentation/postman/collections/CRM Sales Management APIs`. Set the `baseUrl` variable to:

```text
http://localhost:8000/api
```

Sign in first, then send the returned `accessToken` as a bearer token for protected routes. Postman should retain the HTTP-only refresh cookie for session refresh and logout requests.

## Useful project commands

Run these from `source-code/`:

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start the development server with automatic restarts. |
| `npm run typecheck` | Check TypeScript types. |
| `npm run build` | Compile the TypeScript source into `dist/`. |
| `npm start` | Start the compiled server. Run `npm run build` first. |
| `npm run seed:admin` | Create the initial admin account from the `SEED_ADMIN_*` variables. |
| `npm run db:indexes` | Create indexes for the MongoDB models. |

## Troubleshooting

- **Environment validation fails:** Check that `MONGODB_URI`, `JWT_SECRET`, and `JWT_REFRESH_SECRET` are set. Confirm both JWT secrets are at least 32 characters.

- **Health response says MongoDB is disconnected:** Check that MongoDB is running and that `MONGODB_URI` is correct and reachable.

- **Lead conversion fails due to transactions:** Use a replica set or sharded MongoDB deployment.

- **Browser requests fail CORS checks:** Set `CORS_ORIGIN` to the exact frontend origin, including its scheme and port.

- **Port is already in use:** Set `PORT` to a free port and use that port in the health check and Postman `baseUrl`.
