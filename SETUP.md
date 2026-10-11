# CRM Sales Management API — Setup Guide

This guide explains how to configure, run, and test the CRM Sales Management API locally and access the deployed application.

## Live Deployment

The CRM Sales Management API is deployed on Render.

- **Live API:** https://crm-backend-assignment-onc1.onrender.com/
- **Health Check:** https://crm-backend-assignment-onc1.onrender.com/health
- **API Base URL:** https://crm-backend-assignment-onc1.onrender.com/api

The root endpoint returns a welcome message confirming that the API is running. The health endpoint reports the API status and database connection state.

### Live API Homepage

The following screenshot shows the deployed API's root endpoint responding successfully.

![CRM Sales Management API running on Render](screenshots/api-homepage.png)

## Prerequisites

- An active Node.js LTS release with npm
- MongoDB running locally or a reachable MongoDB Atlas deployment
- PowerShell or Command Prompt on Windows, or a terminal on macOS/Linux
- Postman for API testing

Lead conversion uses MongoDB transactions. Use a replica set or sharded MongoDB deployment for this workflow.

## 1. Install Dependencies

From the project root, open a terminal and run:

```powershell
Set-Location .\source-code
npm install
```

On macOS or Linux, use:

```bash
cd source-code
npm install
```

## 2. Configure Environment Variables

Create a file named `.env` inside the `source-code/` directory.

Example configuration for local development:

```dotenv
NODE_ENV=development
PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017/crm_sales_management
JWT_SECRET=replace-with-a-unique-secret-at-least-32-characters-long
JWT_REFRESH_SECRET=replace-with-a-different-unique-secret-at-least-32-characters
ACCESS_TOKEN_TTL=2h
CORS_ORIGIN=http://localhost:5173
```

Replace the sample MongoDB URI with your actual connection string if you use MongoDB Atlas. Generate unique JWT secrets, each at least 32 characters long, and keep `.env` private.

Set `CORS_ORIGIN` to the origin of your frontend application. If you do not have a browser-based frontend, configure this value according to your backend's environment validation requirements.

For the deployed service, environment variables are configured in the Render dashboard. Do not commit real credentials or secrets to GitHub.

## 3. Start the API Locally

Ensure that MongoDB is running and reachable through `MONGODB_URI`.

From the `source-code/` directory, run:

```bash
npm run dev
```

The API listens on port `5000` by default.

Local URLs:

- **Base URL:** http://localhost:5000/
- **Health Check:** http://localhost:5000/health
- **API Base URL:** http://localhost:5000/api

If you change `PORT` in `.env`, use the configured port when testing locally.

## 4. Verify the API

Open a browser or a second terminal and request:

```powershell
Invoke-RestMethod http://localhost:5000/health
```

The response should report the API status and MongoDB connection state. Confirm that the `database` field is `connected` before testing database-dependent endpoints.

You can also open the deployed health endpoint:

https://crm-backend-assignment-onc1.onrender.com/health

## 5. Create the Initial Admin Account

Admin seeding is optional when testing public registration and standard user endpoints.

To create an administrator, add the following variables to `source-code/.env`:

```dotenv
SEED_ADMIN_NAME=Your Admin Name
SEED_ADMIN_EMAIL=admin@example.com
SEED_ADMIN_PHONE=1234567890
SEED_ADMIN_PASSWORD=replace-with-a-strong-password
```

From the `source-code/` directory, run:

```bash
npm run seed:admin
```

Use the configured administrator credentials to sign in through `POST /api/auth/login`. After successful seeding, remove `SEED_ADMIN_PASSWORD` from `.env` if it is no longer needed.

Do not publish real administrator credentials.

## 6. Test the API with Postman

The Postman request definitions are available in:

`api-documentation/postman/collections/CRM Sales Management APIs`

Configure the Postman `baseUrl` variable using one of the following values.

**Local development**

```text
http://localhost:5000/api
```

**Live deployment**

```text
https://crm-backend-assignment-onc1.onrender.com/api
```

Register or sign in first. Use the returned access token as a bearer token for protected endpoints. Postman should retain the HTTP-only refresh cookie for refresh and logout requests.

## Useful Project Commands

Run these commands from the `source-code/` directory.

| Command | Purpose |
|---|---|
| `npm install` | Install dependencies |
| `npm run dev` | Start the development server |
| `npm run typecheck` | Check TypeScript types |
| `npm run build` | Compile TypeScript into `dist/` |
| `npm start` | Start the compiled application |
| `npm run seed:admin` | Create the initial administrator |
| `npm run db:indexes` | Create database indexes |

## Troubleshooting

- **Environment validation fails:** Check the required environment variables and ensure both JWT secrets meet the minimum length.
- **MongoDB is disconnected:** Verify `MONGODB_URI`, database credentials, network access rules, and MongoDB availability.
- **Lead conversion fails:** Verify that MongoDB supports transactions through a replica set or sharded deployment.
- **Browser requests fail CORS checks:** Configure `CORS_ORIGIN` with the correct frontend origin, including the scheme and port where applicable.
- **Port is already in use:** Choose a free port and update the local health-check and Postman URLs.
- **The API returns `404` for an unknown path:** Verify the endpoint path and HTTP method. The root endpoint (`/`) and health endpoint (`/health`) are available for basic checks.

## Security Notes

- Never commit `.env` files, database passwords, JWT secrets, or administrator credentials.
- Keep `.env.example` populated with placeholders only.
- Avoid publishing screenshots containing personal information, password hashes, access tokens, refresh-token hashes, or other sensitive data.