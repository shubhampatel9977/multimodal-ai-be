# Multimodal AI Backend API Documentation

This document is derived from the source code in this repository. It describes the API as currently implemented, including behavior that may differ from the route names or intended application design.

## Table of Contents

- [Overview](#overview)
- [Conventions](#conventions)
- [Authentication](#authentication)
- [Global Middleware and Error Handling](#global-middleware-and-error-handling)
- [Endpoints](#endpoints)
  - [Health](#health)
  - [Authentication endpoints](#authentication-endpoints)
  - [AI model endpoints](#ai-model-endpoints)
  - [Project endpoints](#project-endpoints)
  - [Conversation endpoints](#conversation-endpoints)
  - [Message endpoints](#message-endpoints)
  - [Chat endpoints](#chat-endpoints)
  - [Account endpoints](#account-endpoints)
- [Source Layout](#source-layout)

## Overview

### Base URL

`http://localhost:8080` by default. The port is read from `PORT` and defaults to `8080`; the deployed host/base URL is **Not identifiable from the codebase.**

All application endpoints except the health check are mounted beneath `/api`.

### Response envelope

Most non-streaming controller responses use the following JSON envelope:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Human-readable message",
  "data": {}
}
```

Error responses returned by these controllers use:

```json
{
  "statusCode": 400,
  "success": false,
  "message": "Reason for the failure"
}
```

The `data` property is omitted when the controller passes no data. `/api/chat/stream` is the exception: it returns Server-Sent Events (SSE), and its controller-level error response has no `statusCode` property.

### Common object shapes

`User`:

```json
{
  "id": "clx_user_01",
  "name": "Ada Lovelace",
  "email": "ada@example.com",
  "role": "USER"
}
```

`AIModel`:

```json
{
  "id": "clx_model_01",
  "label": "GPT model",
  "value": "provider/model-id",
  "provider": "OpenRouter",
  "description": "General-purpose model",
  "isDefault": false,
  "isActive": true,
  "createdAt": "2026-07-23T10:00:00.000Z",
  "updatedAt": "2026-07-23T10:00:00.000Z"
}
```

`Project`:

```json
{
  "id": "clx_project_01",
  "name": "Research",
  "userId": "clx_user_01",
  "createdAt": "2026-07-23T10:00:00.000Z",
  "updatedAt": "2026-07-23T10:00:00.000Z"
}
```

`Conversation` returned by conversation/message endpoints includes `model` with `id`, `label`, and `provider`:

```json
{
  "id": "clx_conversation_01",
  "title": "New Chat",
  "isTemporary": false,
  "userId": "clx_user_01",
  "projectId": null,
  "modelId": "clx_model_01",
  "createdAt": "2026-07-23T10:00:00.000Z",
  "updatedAt": "2026-07-23T10:00:00.000Z",
  "model": {
    "id": "clx_model_01",
    "label": "GPT model",
    "provider": "OpenRouter"
  }
}
```

`Message`:

```json
{
  "id": "clx_message_01",
  "conversationId": "clx_conversation_01",
  "role": "ASSISTANT",
  "content": "Hello! How can I help?",
  "promptTokens": 18,
  "completionTokens": 8,
  "totalTokens": 26,
  "createdAt": "2026-07-23T10:00:00.000Z"
}
```

## Conventions

- All JSON request bodies require `Content-Type: application/json`.
- No endpoint reads query parameters; all query-parameter sections therefore state **None**.
- IDs are strings (Prisma `cuid` values). No format validation is applied to path IDs.
- Route-level input validation uses Zod via direct `schema.parse(req.body)` calls in controllers. Unknown-key handling is **Not identifiable from the codebase.**
- `DateTime` values are serialized by Express/JSON as ISO 8601 strings.
- Examples use representative values. Actual CUIDs and generated AI content vary.

## Authentication

Authentication uses signed JWTs stored in HTTP-only cookies:

| Cookie | Purpose | Lifetime | Attributes |
| --- | --- | --- | --- |
| `accessToken` | Authenticates protected requests | 15 minutes | `HttpOnly`, `Path=/`; `Secure` in production; `SameSite=None` in production and `Lax` otherwise |
| `refreshToken` | Obtains rotated access/refresh tokens | 7 days | Same attributes as `accessToken` |

Protected routes use `authorize()`, which reads only `req.cookies.accessToken` and verifies it with `ACCESS_TOKEN_SECRET`. Although CORS permits an `Authorization` header, bearer-token authentication is not implemented. Send requests with cookies enabled (for example, `credentials: "include"` in a browser client).

JWT payloads contain `userId`, `email`, and `role` (`USER` or `ADMIN`). No registered endpoint supplies an allowed-role list to `authorize`, so no role restriction is currently enforced. In particular, `/api/models/admin` is public.

Google sign-in verifies a Google ID token using `GOOGLE_CLIENT_ID`. Local passwords are BCrypt-hashed with `SALT_ROUNDS` (default `12`).

## Global Middleware and Error Handling

Middleware registration order in `src/app.ts`:

1. `helmet()` security headers.
2. `compression()` for all responses except `/api/chat/stream`.
3. `morgan("dev")` when `APP_ENV` is not `production`.
4. `express.json()` JSON parsing.
5. `cookie-parser` cookie parsing.
6. CORS middleware.
7. The health route and API routes.

CORS permits configured `CLIENT_URL` (and requests without an `Origin`), allows `GET`, `POST`, `PUT`, `PATCH`, and `DELETE`, allows `Content-Type` and `Authorization` headers, and enables credentials. Other origins cause the CORS middleware to call `next(new Error(...))`.

There is no custom global Express error-handling middleware. Errors caught by individual controllers are converted to their documented responses. Uncaught errors, including CORS errors and malformed JSON parsing errors, use Express's default error handling; their precise payload is **Not identifiable from the codebase.**

`src/middlewares/rateLimiter.ts` defines a limit of 20 requests per 15 minutes but it is not registered in `app.ts` or any route file. Therefore no active rate limit is identifiable from the route configuration.

## Endpoints

### Health

#### `GET /`

**Description:** Confirms that the Express server is running.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: None.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```text
Server is running...
```

- Error status codes: Not identifiable from the codebase for this simple handler.
- Error example: Not identifiable from the codebase.

**Authentication:** Not required. No roles/permissions.

**Controller Flow:** Route: `src/app.ts` inline handler; Controller: inline handler; Service: none; Repository/database interaction: none.

### Authentication endpoints

#### `POST /api/register`

**Description:** Creates a local email/password user account.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Content-Type: application/json`.
- Request body:

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `name` | string | Required | User display name; minimum 2 characters. |
| `email` | string | Required | Email address; must pass Zod email validation. |
| `password` | string | Required | Local password; minimum 6 characters. |

```json
{ "name": "Ada Lovelace", "email": "ada@example.com", "password": "secure-password" }
```

**Response**

- Success status: `201 Created`.
- Success example:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "User registered successfully",
  "data": { "id": "clx_user_01", "name": "Ada Lovelace", "email": "ada@example.com" }
}
```

- Error status codes: `400 Bad Request` for validation failures, an existing email, or database/service errors caught by the controller.
- Error example:

```json
{ "statusCode": 400, "success": false, "message": "User already exists" }
```

**Authentication:** Not required. No roles/permissions.

**Controller Flow:** Route: `src/modules/auth/auth.routes.ts`; Controller: `register` in `src/modules/auth/auth.controller.ts`; Service: `registerUser` in `src/modules/auth/auth.service.ts`; Repository/database: Prisma `User.findUnique` then `User.create`; password is BCrypt-hashed before persistence.

#### `POST /api/login`

**Description:** Authenticates a local email/password account, creates a persisted refresh token, and sets authentication cookies.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Content-Type: application/json`.
- Request body:

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `email` | string | Required | Must pass Zod email validation. |
| `password` | string | Required | Must contain at least 1 character. |

```json
{ "email": "ada@example.com", "password": "secure-password" }
```

**Response**

- Success status: `200 OK`; also sets `accessToken` and `refreshToken` cookies.
- Success example:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Login successful",
  "data": { "id": "clx_user_01", "name": "Ada Lovelace", "email": "ada@example.com", "role": "USER" }
}
```

- Error status codes: `400 Bad Request` for invalid input, unknown user, invalid password, Google-only account, or other caught failures.
- Error example:

```json
{ "statusCode": 400, "success": false, "message": "Invalid email or password" }
```

**Authentication:** Not required. No roles/permissions.

**Controller Flow:** Route: `src/modules/auth/auth.routes.ts`; Controller: `login`; Service: `loginUser`; Repository/database: Prisma `User.findUnique`, `RefreshToken.create`; BCrypt password comparison; JWT access/refresh generation; cookies set by `src/security/cookies.ts`.

#### `POST /api/google`

**Description:** Signs in or registers a user from a Google ID token, then sets authentication cookies.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Content-Type: application/json`.
- Request body:

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `token` | string | Required in practice | Google ID token. A `googleLoginSchema` exists but is not invoked by the controller; no request validation is applied before Google verification. |

```json
{ "token": "google-id-token" }
```

**Response**

- Success status: `200 OK`; also sets `accessToken` and `refreshToken` cookies.
- Success example:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Google login successful",
  "data": { "id": "clx_user_01", "name": "Ada Lovelace", "email": "ada@example.com", "role": "USER" }
}
```

- Error status codes: `400 Bad Request` for an invalid/missing Google token or other caught Google/database failure.
- Error example:

```json
{ "statusCode": 400, "success": false, "message": "Invalid Google token" }
```

**Authentication:** Not required. No roles/permissions.

**Controller Flow:** Route: `src/modules/auth/auth.routes.ts`; Controller: `googleLoginController`; Service: `googleLogin`; external interaction: Google `OAuth2Client.verifyIdToken`; Repository/database: Prisma `User.findUnique`, optional `User.create`/`User.update`, then `RefreshToken.create`; JWTs are generated and stored in cookies.

#### `POST /api/logout`

**Description:** Deletes the supplied refresh token, if present, and clears both authentication cookies.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: None. The `refreshToken` cookie is optional; without it the endpoint still clears cookies.
- Request body: None.

**Response**

- Success status: `200 OK`; clears `accessToken` and `refreshToken` cookies.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Logout successful" }
```

- Error status codes: `500 Internal Server Error` if the controller's logout operation fails.
- Error example:

```json
{ "statusCode": 500, "success": false, "message": "Logout failed" }
```

**Authentication:** Not required. No roles/permissions.

**Controller Flow:** Route: `src/modules/auth/auth.routes.ts`; Controller: `logout`; Service: `logoutUser`; Repository/database: Prisma `RefreshToken.deleteMany` when a refresh-token cookie is supplied; cookie clearing is performed by `clearAuthCookies`.

#### `POST /api/refresh-token`

**Description:** Rotates a valid, persisted refresh token and sets new access and refresh token cookies.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Cookie: refreshToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`; sets new `accessToken` and `refreshToken` cookies.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Token refreshed" }
```

- Error status codes: `401 Unauthorized` when the cookie is missing, not in the database, invalid, or expired.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Refresh token missing" }
```

```json
{ "statusCode": 401, "success": false, "message": "Invalid refresh token" }
```

**Authentication:** Access-token authentication is not required. A valid refresh-token cookie is required. No roles/permissions.

**Controller Flow:** Route: `src/modules/auth/auth.routes.ts`; Controller: `refreshToken`; Service: `refreshUserToken`; Repository/database: Prisma `RefreshToken.findUnique` and `RefreshToken.update`; refresh JWT is verified and both tokens are regenerated.

#### `GET /api/me`

**Description:** Returns the currently authenticated user's selected profile fields.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "User fetched successfully",
  "data": { "id": "clx_user_01", "name": "Ada Lovelace", "email": "ada@example.com", "role": "USER", "createdAt": "2026-07-23T10:00:00.000Z" }
}
```

- Error status codes: `401 Unauthorized` for missing/invalid access token; `500 Internal Server Error` for controller/service failures.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized - Missing token" }
```

```json
{ "statusCode": 500, "success": false, "message": "Failed to fetch user" }
```

**Authentication:** Required: `accessToken` cookie. Any valid `USER` or `ADMIN` role; no route-level role restriction.

**Controller Flow:** Route: `src/modules/auth/auth.routes.ts`; Middleware: `authorize()`; Controller: `userInfo`; Service: `getCurrentUser`; Repository/database: Prisma `User.findUnique` selecting `id`, `name`, `email`, `role`, and `createdAt`.

### AI model endpoints

> **Access note:** These routes do not apply `authorize()` or another permission middleware. Every model endpoint below is publicly accessible in the current code, including `/api/models/admin` and write operations.

#### `GET /api/models`

**Description:** Lists active AI models, sorted by label ascending.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: None.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Models fetched successfully",
  "data": [{ "id": "clx_model_01", "label": "GPT model", "provider": "OpenRouter", "isDefault": true, "description": "General-purpose model" }]
}
```

- Error status codes: `500 Internal Server Error` for database/controller failures.
- Error example:

```json
{ "statusCode": 500, "success": false, "message": "Failed to fetch models" }
```

**Authentication:** Not required. No roles/permissions are enforced.

**Controller Flow:** Route: `src/modules/ai-model/model.routes.ts`; Controller: `getModelsController`; Service: `getActiveModels`; Repository/database: Prisma `AIModel.findMany` filtered by `isActive: true` and selecting `id`, `label`, `provider`, `isDefault`, `description`.

#### `GET /api/models/admin`

**Description:** Lists all AI model records, including inactive models, newest first. The `/admin` path name does not enforce admin authorization.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: None.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Models fetched successfully", "data": [{ "id": "clx_model_01", "label": "GPT model", "value": "provider/model-id", "provider": "OpenRouter", "description": "General-purpose model", "isDefault": true, "isActive": true, "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:00:00.000Z" }] }
```

- Error status codes: `500 Internal Server Error` for database/controller failures.
- Error example:

```json
{ "statusCode": 500, "success": false, "message": "Failed to fetch models" }
```

**Authentication:** Not required. Required roles/permissions: none enforced.

**Controller Flow:** Route: `src/modules/ai-model/model.routes.ts`; Controller: `getAllModelsController`; Service: `getAllModels`; Repository/database: Prisma `AIModel.findMany`, ordered by `createdAt` descending.

#### `POST /api/models`

**Description:** Creates an AI model configuration. When `isDefault` is true, existing default models are unset first.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Content-Type: application/json`.
- Request body:

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `label` | string | Required | Display label; minimum 1 character. Must be unique. |
| `value` | string | Required | Provider model identifier used for OpenRouter requests; minimum 1 character. Must be unique. |
| `provider` | string | Required | Provider display/value; minimum 1 character. |
| `description` | string | Optional | Model description; no additional validation. |
| `isDefault` | boolean | Optional | Defaults to `false`; if `true`, clears `isDefault` from all other models. |

```json
{ "label": "GPT model", "value": "provider/model-id", "provider": "OpenRouter", "description": "General-purpose model", "isDefault": true }
```

**Response**

- Success status: `201 Created`.
- Success example:

```json
{ "statusCode": 201, "success": true, "message": "Model created successfully", "data": { "id": "clx_model_01", "label": "GPT model", "value": "provider/model-id", "provider": "OpenRouter", "description": "General-purpose model", "isDefault": true, "isActive": true, "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:00:00.000Z" } }
```

- Error status codes: `400 Bad Request` for validation errors, duplicate label/value, or other caught database failures.
- Error example:

```json
{ "statusCode": 400, "success": false, "message": "Model already exists" }
```

**Authentication:** Not required. Required roles/permissions: none enforced.

**Controller Flow:** Route: `src/modules/ai-model/model.routes.ts`; Controller: `createModelController`; Service: `createModel`; Repository/database: Prisma `AIModel.findFirst`, optional `AIModel.updateMany` to clear defaults, then `AIModel.create`.

#### `PATCH /api/models/:id`

**Description:** Partially updates a model. If `isDefault` is true, clears the default flag from all other models first.

**Request**

- Path parameters: `id` (string, required) — AI model ID; no format validation.
- Query parameters: None.
- Required headers: `Content-Type: application/json`.
- Request body: all fields are optional; at least one field is not enforced by the Zod schema.

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `label` | string | Optional | Minimum 1 character. |
| `value` | string | Optional | Minimum 1 character. |
| `provider` | string | Optional | Minimum 1 character. |
| `description` | string | Optional | Description. |
| `isDefault` | boolean | Optional | If true, clears all current defaults first. |

```json
{ "description": "Updated description", "isDefault": true }
```

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Model updated successfully", "data": { "id": "clx_model_01", "label": "GPT model", "value": "provider/model-id", "provider": "OpenRouter", "description": "Updated description", "isDefault": true, "isActive": true, "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:05:00.000Z" } }
```

- Error status codes: `400 Bad Request` for validation, a missing model, uniqueness conflicts, or other caught errors.
- Error example:

```json
{ "statusCode": 400, "success": false, "message": "Model not found" }
```

**Authentication:** Not required. Required roles/permissions: none enforced.

**Controller Flow:** Route: `src/modules/ai-model/model.routes.ts`; Controller: `updateModelController`; Service: `updateModel`; Repository/database: Prisma `AIModel.findUnique`, optional `AIModel.updateMany`, then `AIModel.update`.

#### `DELETE /api/models/:id`

**Description:** Soft-deletes a model by setting `isActive` to `false`; it does not delete the database row.

**Request**

- Path parameters: `id` (string, required) — AI model ID; no format validation.
- Query parameters: None.
- Required headers: None.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Model deleted successfully" }
```

- Error status codes: `400 Bad Request` when the model does not exist or another caught failure occurs.
- Error example:

```json
{ "statusCode": 400, "success": false, "message": "Model not found" }
```

**Authentication:** Not required. Required roles/permissions: none enforced.

**Controller Flow:** Route: `src/modules/ai-model/model.routes.ts`; Controller: `deleteModelController`; Service: `deleteModel`; Repository/database: Prisma `AIModel.findUnique`, then `AIModel.update({ isActive: false })`.

### Project endpoints

All project endpoints require a valid `accessToken` cookie and operate only on projects owned by the authenticated user.

#### `POST /api/projects`

**Description:** Creates a project for the authenticated user.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Content-Type: application/json`; `Cookie: accessToken=<JWT>`.
- Request body:

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `name` | string | Required | Project name; trimmed, 1–100 characters, and unique per user. |

```json
{ "name": "Research" }
```

**Response**

- Success status: `201 Created`.
- Success example:

```json
{ "statusCode": 201, "success": true, "message": "Project created successfully", "data": { "id": "clx_project_01", "name": "Research", "userId": "clx_user_01", "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:00:00.000Z" } }
```

- Error status codes: `401 Unauthorized` for a missing/invalid access token; `400 Bad Request` for validation, duplicate project, or other caught failure.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized - Missing token" }
```

```json
{ "statusCode": 400, "success": false, "message": "Project already exists" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; no additional role restriction.

**Controller Flow:** Route: `src/modules/project/project.routes.ts`; Middleware: router-level `authorize()`; Controller: `createProjectController`; Service: `createProject`; Repository/database: Prisma `Project.findFirst` (owner/name), then `Project.create`.

#### `GET /api/projects`

**Description:** Lists the authenticated user's projects, most recently updated first.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Projects fetched successfully", "data": [{ "id": "clx_project_01", "name": "Research", "userId": "clx_user_01", "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:05:00.000Z" }] }
```

- Error status codes: `401 Unauthorized` for a missing/invalid access token; `500 Internal Server Error` for controller/database failure.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 500, "success": false, "message": "Failed to fetch projects" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; no additional role restriction.

**Controller Flow:** Route: `src/modules/project/project.routes.ts`; Middleware: router-level `authorize()`; Controller: `getProjectsController`; Service: `getProjects`; Repository/database: Prisma `Project.findMany` filtered by `userId`, ordered by `updatedAt` descending.

#### `GET /api/projects/:id`

**Description:** Returns one project if it belongs to the authenticated user.

**Request**

- Path parameters: `id` (string, required) — project ID; no format validation.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Project fetched successfully", "data": { "id": "clx_project_01", "name": "Research", "userId": "clx_user_01", "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:00:00.000Z" } }
```

- Error status codes: `401 Unauthorized` for a missing/invalid access token; `404 Not Found` for a nonexistent or non-owned project.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 404, "success": false, "message": "Project not found" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; ownership is required.

**Controller Flow:** Route: `src/modules/project/project.routes.ts`; Middleware: router-level `authorize()`; Controller: `getProjectController`; Service: `getProjectById`; Repository/database: Prisma `Project.findFirst` filtered by `id` and `userId`.

#### `PATCH /api/projects/:id`

**Description:** Partially updates an owned project.

**Request**

- Path parameters: `id` (string, required) — project ID; no format validation.
- Query parameters: None.
- Required headers: `Content-Type: application/json`; `Cookie: accessToken=<JWT>`.
- Request body: all fields are optional; an empty object is accepted by the schema.

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `name` | string | Optional | Trimmed and 1–100 characters. Database uniqueness per user still applies. |

```json
{ "name": "Updated Research" }
```

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Project updated successfully", "data": { "id": "clx_project_01", "name": "Updated Research", "userId": "clx_user_01", "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:05:00.000Z" } }
```

- Error status codes: `401 Unauthorized` for a missing/invalid access token; `400 Bad Request` for validation, a missing/non-owned project, uniqueness constraints, or other caught errors.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 400, "success": false, "message": "Project not found" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; ownership is required.

**Controller Flow:** Route: `src/modules/project/project.routes.ts`; Middleware: router-level `authorize()`; Controller: `updateProjectController`; Service: `updateProject`; Repository/database: Prisma `Project.findFirst` filtered by `id` and `userId`, then `Project.update`.

#### `DELETE /api/projects/:id`

**Description:** Permanently deletes an owned project. Prisma's schema cascade deletes associated conversations and messages.

**Request**

- Path parameters: `id` (string, required) — project ID; no format validation.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Project deleted successfully" }
```

- Error status codes: `401 Unauthorized` for a missing/invalid access token; `400 Bad Request` for a missing/non-owned project or other caught failure.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 400, "success": false, "message": "Project not found" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; ownership is required.

**Controller Flow:** Route: `src/modules/project/project.routes.ts`; Middleware: router-level `authorize()`; Controller: `deleteProjectController`; Service: `deleteProject`; Repository/database: Prisma `Project.findFirst` filtered by `id` and `userId`, then `Project.delete`; cascading behavior is declared in `prisma/schema.prisma`.

### Conversation endpoints

All conversation endpoints require a valid `accessToken` cookie. Conversation lookups and deletion are ownership-scoped.

#### `POST /api/conversations`

**Description:** Creates a conversation using an active AI model, optionally associated with one of the caller's projects.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Content-Type: application/json`; `Cookie: accessToken=<JWT>`.
- Request body:

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `modelId` | string | Required | Active AI model ID; non-empty string. Existence and active state are verified. |
| `projectId` | string | Optional | Project ID; when supplied it must be owned by the authenticated user. Non-empty is not enforced. |
| `isTemporary` | boolean | Optional | Defaults to `false`. Temporary conversations are excluded from list endpoints. |

```json
{ "modelId": "clx_model_01", "projectId": "clx_project_01", "isTemporary": false }
```

**Response**

- Success status: `201 Created`.
- Success example:

```json
{ "statusCode": 201, "success": true, "message": "Conversation created successfully", "data": { "id": "clx_conversation_01", "title": "New Chat", "isTemporary": false, "userId": "clx_user_01", "projectId": "clx_project_01", "modelId": "clx_model_01", "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:00:00.000Z", "model": { "id": "clx_model_01", "label": "GPT model", "value": "provider/model-id", "provider": "OpenRouter", "description": "General-purpose model", "isDefault": true, "isActive": true, "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:00:00.000Z" } } }
```

- Error status codes: `401 Unauthorized` for missing/invalid access token; `400 Bad Request` for validation, inactive/missing model, absent/non-owned project, or other caught errors.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 400, "success": false, "message": "Model not found" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role. `projectId`, when present, must belong to the user.

**Controller Flow:** Route: `src/modules/conversation/conversation.routes.ts`; Middleware: router-level `authorize()`; Controller: `createConversationController`; Service: `createConversation`; Repository/database: Prisma `AIModel.findFirst` (active), optional `Project.findFirst` (owner), then `Conversation.create` including the complete model relation.

#### `GET /api/conversations`

**Description:** Lists the caller's non-temporary standalone conversations (no project), newest updated first.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Conversations fetched successfully", "data": [{ "id": "clx_conversation_01", "title": "Research ideas", "isTemporary": false, "userId": "clx_user_01", "projectId": null, "modelId": "clx_model_01", "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:05:00.000Z", "model": { "id": "clx_model_01", "label": "GPT model", "provider": "OpenRouter" } }] }
```

- Error status codes: `401 Unauthorized` for missing/invalid access token; `500 Internal Server Error` for controller/database failure.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 500, "success": false, "message": "Failed to fetch conversations" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role.

**Controller Flow:** Route: `src/modules/conversation/conversation.routes.ts`; Middleware: router-level `authorize()`; Controller: `getConversationsController`; Service: `getStandaloneConversations`; Repository/database: Prisma `Conversation.findMany` filtered by `userId`, `projectId: null`, `isTemporary: false`, with selected model fields.

#### `GET /api/conversations/project/:projectId`

**Description:** Lists the caller's non-temporary conversations associated with the supplied project ID, newest updated first. The service filters by `userId` and `projectId` but does not separately verify that the project exists.

**Request**

- Path parameters: `projectId` (string, required) — project ID; no format validation.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK` (including when no matching conversations exist).
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Project conversations fetched successfully", "data": [{ "id": "clx_conversation_01", "title": "Research ideas", "isTemporary": false, "userId": "clx_user_01", "projectId": "clx_project_01", "modelId": "clx_model_01", "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:05:00.000Z", "model": { "id": "clx_model_01", "label": "GPT model", "provider": "OpenRouter" } }] }
```

- Error status codes: `401 Unauthorized` for missing/invalid access token; `500 Internal Server Error` for controller/database failure.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 500, "success": false, "message": "Failed to fetch conversations" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role. Matching is scoped to the caller's conversations.

**Controller Flow:** Route: `src/modules/conversation/conversation.routes.ts`; Middleware: router-level `authorize()`; Controller: `getProjectConversationsController`; Service: `getProjectConversations`; Repository/database: Prisma `Conversation.findMany` filtered by `userId`, `projectId`, `isTemporary: false`, with selected model fields.

#### `GET /api/conversations/:id`

**Description:** Gets a conversation owned by the caller, including selected model fields.

**Request**

- Path parameters: `id` (string, required) — conversation ID; no format validation.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Conversation fetched successfully", "data": { "id": "clx_conversation_01", "title": "Research ideas", "isTemporary": false, "userId": "clx_user_01", "projectId": null, "modelId": "clx_model_01", "createdAt": "2026-07-23T10:00:00.000Z", "updatedAt": "2026-07-23T10:05:00.000Z", "model": { "id": "clx_model_01", "label": "GPT model", "provider": "OpenRouter" } } }
```

- Error status codes: `401 Unauthorized` for missing/invalid access token; `404 Not Found` for a nonexistent or non-owned conversation.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 404, "success": false, "message": "Conversation not found" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; ownership is required.

**Controller Flow:** Route: `src/modules/conversation/conversation.routes.ts`; Middleware: router-level `authorize()`; Controller: `getConversationController`; Service: `getConversationById`; Repository/database: Prisma `Conversation.findFirst` filtered by `id` and `userId`, including selected model fields.

#### `DELETE /api/conversations/:id`

**Description:** Permanently deletes a conversation owned by the caller. Prisma's schema cascade deletes its messages.

**Request**

- Path parameters: `id` (string, required) — conversation ID; no format validation.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Conversation deleted successfully" }
```

- Error status codes: `401 Unauthorized` for missing/invalid access token; `400 Bad Request` for a nonexistent/non-owned conversation or other caught failure.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 400, "success": false, "message": "Conversation not found" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; ownership is required.

**Controller Flow:** Route: `src/modules/conversation/conversation.routes.ts`; Middleware: router-level `authorize()`; Controller: `deleteConversationController`; Service: `deleteConversation`; Repository/database: Prisma `Conversation.findFirst` filtered by `id` and `userId`, then `Conversation.delete`; cascading behavior is declared in `prisma/schema.prisma`.

### Message endpoints

#### `GET /api/conversations/:id/messages`

**Description:** Gets an owned conversation with its selected model data and all messages, ordered oldest to newest.

**Request**

- Path parameters: `id` (string, required) — conversation ID; no format validation.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Messages fetched successfully",
  "data": {
    "conversation": {
      "id": "clx_conversation_01",
      "title": "Research ideas",
      "isTemporary": false,
      "userId": "clx_user_01",
      "projectId": null,
      "modelId": "clx_model_01",
      "createdAt": "2026-07-23T10:00:00.000Z",
      "updatedAt": "2026-07-23T10:05:00.000Z",
      "model": { "id": "clx_model_01", "label": "GPT model", "provider": "OpenRouter" }
    },
    "messages": [{ "id": "clx_message_01", "conversationId": "clx_conversation_01", "role": "USER", "content": "Explain JWTs", "promptTokens": 0, "completionTokens": 0, "totalTokens": 0, "createdAt": "2026-07-23T10:00:00.000Z" }]
  }
}
```

- Error status codes: `401 Unauthorized` for missing/invalid access token; `404 Not Found` for a nonexistent or non-owned conversation.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 404, "success": false, "message": "Conversation not found" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; conversation ownership is required.

**Controller Flow:** Route: `src/modules/message/message.routes.ts`, mounted at `/api/conversations`; Middleware: router-level `authorize()`; Controller: `getMessagesController`; Service: `getConversationMessages`; Repository/database: Prisma `Conversation.findFirst` filtered by `id` and `userId`, then `Message.findMany` filtered by `conversationId`, ordered by `createdAt` ascending.

### Chat endpoints

Both chat endpoints require an access-token cookie and a conversation owned by the caller. They persist the user's message before calling OpenRouter, use up to 20 earliest messages from the conversation as history, and request up to 2,000 output tokens. For a conversation whose title is `New Chat`, they then call OpenRouter again to generate a title of up to five words. Failures after persisting a user message can leave that message in the database; rollback/transaction behavior is not implemented.

#### `POST /api/chat/send`

**Description:** Sends a message to the conversation's configured OpenRouter model and returns the persisted assistant message after completion.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Content-Type: application/json`; `Cookie: accessToken=<JWT>`.
- Request body:

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `conversationId` | string | Required | Conversation ID; non-empty string. It must belong to the caller. |
| `message` | string | Required | User prompt; trimmed and must contain at least one character. |

```json
{ "conversationId": "clx_conversation_01", "message": "Explain JWTs simply." }
```

**Response**

- Success status: `200 OK`.
- Success example:

```json
{ "statusCode": 200, "success": true, "message": "Message sent successfully", "data": { "message": { "id": "clx_message_02", "conversationId": "clx_conversation_01", "role": "ASSISTANT", "content": "A JWT is a signed token...", "promptTokens": 18, "completionTokens": 8, "totalTokens": 26, "createdAt": "2026-07-23T10:01:00.000Z" } } }
```

- Error status codes: `401 Unauthorized` for missing/invalid access token; `400 Bad Request` for validation, a nonexistent/non-owned conversation, OpenRouter errors, title-generation errors, or other caught failures.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 400, "success": false, "message": "Conversation not found" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; conversation ownership is required.

**Controller Flow:** Route: `src/modules/chat/chat.routes.ts`; Middleware: router-level `authorize()`; Controller: `sendMessageController`; Service: `sendMessage`; Repository/database: Prisma `Conversation.findFirst` (owner, including model), `Message.create` (user), `Message.findMany` (history), `Message.create` (assistant), optional `Conversation.update` (title). External service: `openrouter.chat.completions.create`.

#### `POST /api/chat/stream`

**Description:** Sends a message and streams the assistant response using Server-Sent Events. This route is explicitly excluded from response compression.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Content-Type: application/json`; `Cookie: accessToken=<JWT>`. The response sets `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`, and `X-Accel-Buffering: no`.
- Request body:

| Field | Type | Required | Description and validation |
| --- | --- | --- | --- |
| `conversationId` | string | Required | Conversation ID; non-empty string. It must belong to the caller. |
| `message` | string | Required | User prompt; trimmed and must contain at least one character. |

```json
{ "conversationId": "clx_conversation_01", "message": "Explain JWTs simply." }
```

**Response**

- Success status: `200 OK` with `text/event-stream` body. No conventional JSON response envelope is sent.
- Success event examples:

```text
data: {"content":"A JWT is "}

data: {"content":"a signed token..."}

data: {"done":true}

```

- Error status codes: `401 Unauthorized` for missing/invalid access token before the stream handler runs; `400 Bad Request` for request-body validation errors caught in the controller. Once stream headers have been sent, service errors are returned as SSE data rather than a reliable HTTP error status.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "success": false, "message": "Conversation not found" }
```

```text
data: {"error":true,"message":"Conversation not found"}

```

**Authentication:** Required: `accessToken` cookie. Any valid role; conversation ownership is required.

**Controller Flow:** Route: `src/modules/chat/chat.routes.ts`; Middleware: router-level `authorize()`; Controller: `sendMessageStreamController`; Service: `sendMessageStream`; Repository/database: Prisma `Conversation.findFirst` (owner, including model), `Message.create` (user), `Message.findMany` (history), `Message.create` (assistant), optional `Conversation.update` (title). External service: OpenRouter streaming `chat.completions.create({ stream: true, stream_options: { include_usage: true } })`.

### Account endpoints

#### `GET /api/account`

**Description:** Returns the authenticated user's profile and aggregate usage statistics.

**Request**

- Path parameters: None.
- Query parameters: None.
- Required headers: `Cookie: accessToken=<JWT>`.
- Request body: None.

**Response**

- Success status: `200 OK`.
- Success example:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Account details fetched successfully",
  "data": {
    "profile": { "id": "clx_user_01", "name": "Ada Lovelace", "email": "ada@example.com", "role": "USER", "provider": "LOCAL" },
    "stats": { "chatCount": 3, "projectCount": 2, "messageCount": 18, "totalTokens": 2420 }
  }
}
```

- Error status codes: `401 Unauthorized` for missing/invalid access token; `404 Not Found` for a missing user or any error caught by the controller.
- Error examples:

```json
{ "statusCode": 401, "success": false, "message": "Unauthorized" }
```

```json
{ "statusCode": 404, "success": false, "message": "User not found" }
```

**Authentication:** Required: `accessToken` cookie. Any valid role; no additional role restriction.

**Controller Flow:** Route: `src/modules/account/account.routes.ts`; Middleware: router-level `authorize()`; Controller: `getAccountController`; Service: `getAccountDetails`; Repository/database: Prisma `User.findUnique` (profile), `Project.count`, `Conversation.count` (`isTemporary: false`), and `Message.aggregate` (message count and total token sum for the user's conversations).

## Source Layout

| Concern | Location |
| --- | --- |
| Application setup and global middleware | `src/app.ts` |
| Server startup and database connection | `src/server.ts` |
| API mount points | `src/routes/index.ts` |
| Feature route/controller/service folders | `src/modules/auth`, `src/modules/ai-model`, `src/modules/project`, `src/modules/conversation`, `src/modules/message`, `src/modules/chat`, `src/modules/account` |
| Request validation schemas | Feature `*.validation.ts` files under `src/modules` |
| Authentication middleware | `src/middlewares/auth.middleware.ts` |
| JWT and cookie helpers | `src/utils/jwt.ts`, `src/security/cookies.ts` |
| Prisma client and database schema | `src/config/prisma.ts`, `prisma/schema.prisma` |
| OpenRouter client | `src/config/openrouter.ts` |
| Standard API response helpers | `src/utils/apiResponse.ts` |

The project has no separate repository layer. Services call Prisma directly, so the repository/database interaction for each endpoint is implemented in the corresponding service file.
