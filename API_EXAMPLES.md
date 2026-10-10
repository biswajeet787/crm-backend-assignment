# Sample API Requests and Responses

These examples target the deployed API at [https://crm-backend-assignment-pvqh.onrender.com](https://crm-backend-assignment-pvqh.onrender.com). Replace sample IDs and credentials with values from your database. Request bodies are JSON. Protected routes require an access token unless an example says otherwise.

Set the Postman `baseUrl` variable to:

```text
https://crm-backend-assignment-pvqh.onrender.com/api
```


Sample response bodies are abbreviated where indicated; actual responses may include additional fields from the model schema.

## 1. Log in

Login is public. A successful response returns an access token and sets a refresh cookie.

### Request

```http
POST {{baseUrl}}/auth/login
Content-Type: application/json

{
  "email": "alex.morgan@example.com",
  "password": "replace-with-your-password"
}
```

### Response

```http
HTTP/1.1 200 OK
Set-Cookie: crm_refresh=<opaque-refresh-token>; Path=/api/auth; HttpOnly; SameSite=Lax; Max-Age=604800
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "Signed in",
  "data": {
    "user": {
      "id": "64f100000000000000000001",
      "name": "Alex Morgan",
      "email": "alex.morgan@example.com",
      "phone": "5558675309",
      "role": "sales_executive",
      "isActive": true,
      "managerId": null,
      "manager": null,
      "createdAt": "2026-10-01T10:00:00.000Z",
      "updatedAt": "2026-10-01T10:00:00.000Z"
    },
    "accessToken": "<access-token>"
  }
}
```

Send the `accessToken` as `Authorization: Bearer <access-token>` on protected requests. The refresh token is only sent as the `crm_refresh` cookie. Registration uses the same response shape with status `201` and message `Account created`.

## 2. Create a lead

Sales executives can create a lead and are assigned the lead by default. Admins and sales managers can also create and assign leads.

### Request

```http
POST {{baseUrl}}/leads
Authorization: Bearer {{accessToken}}
Content-Type: application/json

{
  "name": "Jordan Lee",
  "email": "jordan.lee@example.com",
  "phone": "5551234567",
  "company": "Northwind Labs",
  "source": "referral",
  "priority": "high",
  "description": "Interested in a sales workflow demo."
}
```

### Response

```http
HTTP/1.1 201 Created
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "Lead created",
  "data": {
    "_id": "64f100000000000000000101",
    "name": "Jordan Lee",
    "email": "jordan.lee@example.com",
    "phone": "5551234567",
    "company": "Northwind Labs",
    "source": "referral",
    "status": "new",
    "priority": "high",
    "assignedTo": "64f100000000000000000001",
    "createdBy": "64f100000000000000000001",
    "description": "Interested in a sales workflow demo.",
    "convertedCustomerId": null,
    "convertedDealId": null,
    "createdAt": "2026-10-10T08:30:00.000Z",
    "updatedAt": "2026-10-10T08:30:00.000Z"
  }
}
```

## 3. List assigned leads

Use the assigned-leads route for a sales executive. The regular `GET /leads` route requires `leads:read`, which is not granted to sales executives.

### Request

```http
GET {{baseUrl}}/leads/assigned-to-me?page=1&limit=20&status=new
Authorization: Bearer {{accessToken}}
```

### Response excerpt

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "Assigned leads fetched",
  "data": {
    "items": [
      {
        "_id": "64f100000000000000000101",
        "name": "Jordan Lee",
        "email": "jordan.lee@example.com",
        "company": "Northwind Labs",
        "source": "referral",
        "status": "new",
        "priority": "high",
        "assignedTo": {
          "_id": "64f100000000000000000001",
          "name": "Alex Morgan",
          "email": "alex.morgan@example.com",
          "role": "sales_executive"
        },
        "createdAt": "2026-10-10T08:30:00.000Z",
        "updatedAt": "2026-10-10T08:30:00.000Z"
      }
    ]
  },
  "meta": {
    "currentPage": 1,
    "pageSize": 20,
    "totalRecords": 1,
    "totalPages": 1
  }
}
```

Other list routes use the same top-level `data.items` and `meta` pagination pattern. Query fields vary by resource; `page` defaults to `1` and `limit` defaults to `20` (maximum `100`).

## 4. Change a lead's status

Status transitions are validated. For example, a `new` lead can move to `contacted`.

### Request

```http
PATCH {{baseUrl}}/leads/64f100000000000000000101/status
Authorization: Bearer {{accessToken}}
Content-Type: application/json

{
  "status": "contacted"
}
```

### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "Lead status updated",
  "data": {
    "_id": "64f100000000000000000101",
    "name": "Jordan Lee",
    "email": "jordan.lee@example.com",
    "phone": "5551234567",
    "company": "Northwind Labs",
    "status": "contacted",
    "source": "referral",
    "priority": "high",
    "assignedTo": "64f100000000000000000001",
    "createdBy": "64f100000000000000000001",
    "description": "Interested in a sales workflow demo.",
    "convertedCustomerId": null,
    "convertedDealId": null,
    "createdAt": "2026-10-10T08:30:00.000Z",
    "updatedAt": "2026-10-10T09:00:00.000Z"
  }
}
```

## 5. Convert a qualified lead

Only admins and sales managers can convert a qualified lead. Conversion requires a MongoDB replica set or sharded cluster because it uses a transaction.

### Request

```http
POST {{baseUrl}}/leads/64f100000000000000000101/convert
Authorization: Bearer {{managerAccessToken}}
Content-Type: application/json

{
  "customer": {
    "name": "Jordan Lee",
    "email": "jordan.lee@example.com",
    "phone": "5551234567",
    "company": "Northwind Labs",
    "address": "Seattle, WA"
  },
  "deal": {
    "name": "Northwind CRM Rollout",
    "amount": 25000,
    "probability": 75,
    "expectedClosingDate": "2027-01-31T00:00:00.000Z",
    "description": "CRM rollout and onboarding."
  }
}
```

### Response excerpt

```http
HTTP/1.1 201 Created
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "Lead converted",
  "data": {
    "customer": {
      "_id": "64f100000000000000000201",
      "name": "Jordan Lee",
      "email": "jordan.lee@example.com",
      "company": "Northwind Labs",
      "leadId": "64f100000000000000000101",
      "status": "active"
    },
    "deal": {
      "_id": "64f100000000000000000301",
      "name": "Northwind CRM Rollout",
      "customerId": "64f100000000000000000201",
      "leadId": "64f100000000000000000101",
      "amount": 25000,
      "probability": 75,
      "expectedRevenue": 18750,
      "stage": "qualification"
    },
    "lead": {
      "_id": "64f100000000000000000101",
      "status": "converted",
      "convertedCustomerId": "64f100000000000000000201",
      "convertedDealId": "64f100000000000000000301"
    }
  }
}
```

The response excerpt omits additional document fields. If `customer` is omitted from the request, the API uses the lead's contact details where available. The deal's expected revenue is calculated from amount and probability.

## 6. Create and complete a follow-up activity

### Create request

```http
POST {{baseUrl}}/activities
Authorization: Bearer {{accessToken}}
Content-Type: application/json

{
  "type": "call",
  "title": "Discuss implementation timeline",
  "description": "Confirm target launch date and stakeholders.",
  "dueAt": "2027-02-03T16:00:00.000Z",
  "entityType": "deal",
  "entityId": "64f100000000000000000301"
}
```

### Create response

```json
{
  "success": true,
  "message": "Activity created",
  "data": {
    "_id": "64f100000000000000000401",
    "type": "call",
    "title": "Discuss implementation timeline",
    "description": "Confirm target launch date and stakeholders.",
    "assignedTo": "64f100000000000000000001",
    "dueAt": "2027-02-03T16:00:00.000Z",
    "status": "pending",
    "entityType": "deal",
    "entityId": "64f100000000000000000301",
    "createdBy": "64f100000000000000000001",
    "completedAt": null,
    "createdAt": "2026-10-10T09:15:00.000Z",
    "updatedAt": "2026-10-10T09:15:00.000Z"
  }
}
```

### Complete request

```http
PATCH {{baseUrl}}/activities/64f100000000000000000401/complete
Authorization: Bearer {{accessToken}}
```

Completion returns `200 OK`, message `Activity completed`, and the activity with `status: "completed"` and a populated `completedAt` timestamp. The complete response uses the standard success envelope.

## 7. Read sales summary analytics

### Request

```http
GET {{baseUrl}}/analytics/summary
Authorization: Bearer {{accessToken}}
```

### Response

```json
{
  "success": true,
  "message": "Sales summary fetched",
  "data": {
    "totalLeads": 30,
    "newLeads": 9,
    "qualifiedLeads": 3,
    "convertedLeads": 6,
    "totalCustomers": 6,
    "totalDeals": 4,
    "openDeals": 2,
    "wonDeals": 1,
    "lostDeals": 1,
    "totalRevenue": 25000,
    "expectedRevenue": 40000,
    "conversionRate": 20,
    "pendingActivities": 5,
    "overdueActivities": 2
  }
}
```

The numeric values above are illustrative. Analytics are scoped according to the authenticated user's role and record visibility.

## 8. Example errors

Missing or invalid access token:

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json
```

```json
{
  "success": false,
  "message": "Authentication is required"
}
```

Invalid request data (for example, a phone number with fewer than ten digits):

```json
{
  "success": false,
  "message": "Request validation failed",
  "details": [
    {
      "path": "phone",
      "message": "Phone number must be exactly 10 digits"
    }
  ]
}
```

An authenticated user who lacks the required permission receives `403 Forbidden` with `"You do not have permission to perform this action"`. Successful responses use `{ "success": true, "message": "...", "data": ... }`; paginated responses also include top-level `meta`.
