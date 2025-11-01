# API Intergation Guide : Acuity Schedling vs. Square Appointments

## Document Purpose
This document provides a detailed, side-by-side comparsion of the Acuity Scheduling and Square Appointments APIs.It is designed to help developers understand the key difference in authetication, API structure, request/response bodies, and integration patterns when working with either platform.

### 1. Overview


## 2. Authentication & Setup Comparison
Side-by-side comparison of authentication methods and initial setup requirements
#### 2.1 🔄 OAuth Feature Comparison

| **Feature** | **Square** | **Acuity (OAuth)** |
|--------------|-------------|--------------------|
|**Authorize URL**   |`https://connect.squareup.com/oauth2/authorize` *(sandbox: `https://connect.squareupsandbox.com/oauth2/authorize`)* | `https://acuityscheduling.com/oauth2/authorize` |  
|**Token Endpoint**  |`https://connect.squareup.com/oauth2/token`                                            |`https://acuityscheduling.com/oauth2/token`                    |
|**Disconnect /Revoke**    | `https://connect.squareup.com/oauth2/revoke` | `https://acuityscheduling.com/oauth2/disconnect` |        
|**Scopes**          | 'MERCHANT_PROFILE_READ ORDERS_READ ORDERS_WRITE PAYMENTS_READ INVENTORY_READ INVENTORY_WRITE'                               |  `api-v1` (covers appointments/availability per Acuity SDK)                  |
|**Client Vars**     | `SQUARE_CLIENT_ID`, `SQUARE_CLIENT_SECRET`             | `ACUITY_CLIENT_ID`, `ACUITY_CLIENT_SECRET`                   |
|**Token Type**      |  `Bearer`           |    `Bearer`                |
|**Refresh Token**   | Supported            |  Supported                  |
|**State Parameters**|  Used for CSRF protection  |  Used for CSRF protection                  |
| **Redirect URI (our app)** | `${SERVER_URL}/square-callback` | `${SERVER_URL}/acuity-callback` |


#### 2.2 OAuth Flow
Both square and acuity services follow the standard OAuth 2.0 flow:
##### 1. **Generate Authorization URL**
##### 2. **User Login & Consent (Privider Page)** 
> The user is redirected to **Square/Acuity** to log in and approve CallPal's requested scopes.
##### 3. **Callback with Authorization Code**
> Provider redicects back to our **backend callback** with `code` and `state`:
- Square:`POST /square-callback` 
- Acuity:`POST /acuity-callback`    
##### 4. **Token Excahnge** 
> Backend exchanges `code` for tokens at `/oauth2/token` using `client_id` + `client_secret`.

##### 5. **Secure Storage**
> **Encrypt** tokens using AES-256-CBC and store them (per-agent) in our database.

##### 6. **API Calls**
> Subsequent calls to provider APIs include `Authorization: Bearer <access_token>`.
##### 7. **Token Refresh** (if access token expires)
> Backend uses`grant_type=refresh_token` to obtain a new access token without user interaction.


#### 2.3 How to Setup in Acuity Platform
Follow these steps to enable OAuth integration between **Acuity Scheduling** and **CallPal**
1. **Open the Acuity OAuth Registration page**
>    Go to [https://acuityscheduling.com/oauth2/register](https://acuityscheduling.com/oauth2/register).  
   This page allows you to register a new OAuth 2.0 application that can connect securely to Acuity’s API.
2. **Register new Application**
 - Fill out the registration form with the following details:
   - **Application Name**: `CallPal`
     - **Redirect URI**: `${SERVER_URL}/acuity-callback`
     - **Homepage URL**: (optional, your internal app or project URL)
   - Click **Register Application**.
3. **Retrieve Your OAuth Credentials**
   - After successful registration, Acuity will display your:
     - **Client ID**
     - **Client Secret**
   - These credentials uniquely identify your app.

4. **Store Credentials Securely**
   - Add the credentials to your environment configuration file (`.env`):
     ```bash
     ACUITY_CLIENT_ID=your_client_id
     ACUITY_CLIENT_SECRET=your_client_secret
     ```
   - Do **not** commit these values to version control.
   - Use environment variables or a secure vault to manage them.
5. **Set Required OAuth Scopes**
   - Acuity uses a unified `api-v1` scope that covers all major endpoints:
     - `appointments`
     - `appointment-types`
     - `availability/times`
   - No extra granular scopes are required.
6. **Test the OAuth Connection**
   - Start your backend server.
   - Go to your CallPal app and click **“Connect Acuity”**.
   - You should be redirected to:
     ```
     https://acuityscheduling.com/oauth2/authorize?client_id=...
     ```
   - Log in and authorize CallPal.
   - Upon approval, Acuity redirects to your backend callback:
     `${SERVER_URL}/acuity-callback?code=<authorization_code>&state=<random>:<agentId>`
7. **Verify Integration**
   - Once the tokens are saved, you can confirm the connection by calling:
     ```bash
     GET /status/:agentId
     ```
   - A successful integration returns:
     ```json
     {
       "connected": true,
       "lastSyncAt": "2025-10-21T12:00:00Z"
       "appointmentTypes": []
     }
     ```

> **Note:** In production, always use HTTPS for your redirect URI and never expose `CLIENT_SECRET` in frontend code.


## 3. Unified Customer APIs
The Unified Customer APIs provide a standardized interface for managing customer data across **Square** and **Acuity Scheduling** platforms.
This layer abstracts the differences between provider-specific customer management systems, allowing you to use a single set of endpoints regardless of which platform is connected to your agent.
**Supported Operations:**
1. **Create Customer** - Add a new customer to the system
2. **Search Customers** - Find customers using various filters
3. **Search by Phone** - Quick lookup by phone number (with caching)
4. **Retrieve Customer** - Get details of a specific customer by ID

---

## Base URLs

| Platform | Base URL |
|----------|----------|
| **Our Backend** | `https://your-api.com/api/customers` |
| **Square** | `https://connect.squareup.com` |
| **Acuity** | `https://acuityscheduling.com` |

## Authentication

All endpoints require Bearer Token authentication:

## 3.1 Overview of Routes

| # | Backend Route | Method | Description | Returns |
|---|---------------|--------|-------------|---------|
| 1 | `/create/:agentId` | POST | Create new customer | Single customer object |
| 2 | `/search/:agentId` | GET | Search customers by filters | First matching customer |
| 3 | `/search-by-phone/:agentId` | GET | Search by phone number | Single customer object |
| 4 | `/retrieve/:agentId/:customerId` | GET | Get customer by ID | Single customer object |

**Global Notes:**
- All endpoints require Bearer Token authentication
- All responses include `status` and `provider` fields
- Search endpoints limited to 1 result (backend uses `.slice(0, 1)`)
---

## 3.2 Create Customer

### 3.2.1 Platform Comparison

| Platform | Endpoint | Method |
|----------|----------|--------|
| **Our Backend** | `/create/:agentId` | POST |
| **Square** | `/v2/customers` | POST |
| **Acuity** | `/api/v1/clients` | POST |

**Note:** Our backend requires `agentId` in URL path; all platforms accept customer data in request body.

**Important** Acuity check duplicate client based on firstname and lastname
If firstname and lastname are duplicated, it will return 400


### 3.2.2 Parameters

#### Our Backend 

**Required Fields:**
- `First Name` * (accepts: `firstName`, `given_name`)
- `Phone ` * (accepts: `phone`, `phone_number` - auto-normalized to `+1xxxxxxxxxx`)

**Optional Fields:**
- `Last Name` (accepts: `lastName`, `family_name`)
- `Email` (accepts: `email`, `email_address`)
- `Notes` (accepts: `notes`)

*\* = Required field*

#### Platform Parameter Comparison

| Parameter | Our Backend Input | Square | Acuity | Normalized Output format |
|-----------|-------------------|--------|--------|-------------------|
| **First Name** | ✅ **Required** <br> Accepts: `firstName`, `given_name`, `full_name` | ⚠️ Optional <br> (part of "at least one") <br> Field: `given_name` | ✅ **Required** <br> Field: `firstName` | `given_name` |
| **Last Name** | ⭕ Optional <br> Accepts: `lastName`, `family_name`, `full_name` | ⚠️ Optional <br> (part of "at least one") <br> Field: `family_name` | ⭕ Optional <br> Field: `lastName` | `family_name` <br> (empty string if not provided) |
| **Phone** | ✅ **Required** <br> Accepts: `phone`, `phone_number` | ⚠️ Optional <br> (part of "at least one") <br> Field: `phone_number` | ✅ **Required** <br> Field: `phone` | `phone_number` <br> (E.164 for Square, display format for Acuity) |
| **Email** | ⭕ Optional <br> Accepts: `email`, `email_address` | ⚠️ Optional <br> (part of "at least one") <br> Field: `email_address` | ✅ **Required** <br> Field: `email` | `email_address` <br> (empty string or `contact@callpal.app` for Acuity) |
| **Customer ID** | ❌ Not for input | Auto-generated （field:id） | phone number as customer id | `id` |
| **Company Name** | ❌ Not supported | ⚠️ Optional <br> (part of "at least one") <br> Field: `company_name` | ❌ Not supported | ❌ Not included in output |
| **Notes** | ⭕ Optional <br> Accepts: `notes` | ⭕ Optional <br> Field: `note` | ⭕ Optional <br> Field: `notes` | ❌ Not included in normalized output |

**Legend:**
- ✅ = Required
- ⭕ = Optional  
- ⚠️ = Part of Square's "at least one" requirement
- ❌ = Not supported / Not included

**Note:** 

**Square Special Rule:** Must provide at least ONE of: `given_name`, `family_name`, `company_name`, `email_address`, or `phone_number`
**Acuity Special Rule:**
- Must provide ALL THREE: `firstName`, `phone`, and `email`
- Our backend auto-fills `email` with `contact@callpal.app` if not provided

**Our Backend Rule:**
- Must provide BOTH: `firstName` and `phone`
- Auto-normalizes phone to E.164 format (`+1xxxxxxxxxx`)



### 3.2.3 Response Example
 **Our backend vs Square**
 ```
 our backend 
 {
    "status": "success",
    "provider": "square",
    "data": {
        "id": "JDKYHBWT1D4F8MFH63DBMEN8Y4",
        "given_name": "Amelia",
        "family_name": "Earhart",
        "phone_number": "+12125554240",
        "email_address": ""
    }
}

square platform:
{
  "customer": {
    "id": "JDKYHBWT1D4F8MFH63DBMEN8Y4",
    "created_at": "2016-03-23T20:21:54.859Z",
    "updated_at": "2016-03-23T20:21:54.859Z",
    "given_name": "Amelia",
    "family_name": "Earhart",
    "email_address": "xx@example.com",
    "address": {
      "address_line_1": "500 Electric Ave",
      "address_line_2": "Suite 600",
      "locality": "New York",
      "administrative_district_level_1": "NY",
      "postal_code": "10003",
      "country": "US"
    },
    "phone_number": "+1-212-555-4240",
    "reference_id": "YOUR_REFERENCE_ID",
    "note": "a customer",
    "preferences": {
      "email_unsubscribed": false
    },
    "creation_source": "THIRD_PARTY",
    "version": 0
  }
}

 ```
 **Our backend vs Acuity**
 ```
our backend:
    {
     "status": "success",
    "provider": "acuity",
    "data": {
        "given_name": "Kate",
        "family_name": "null",
        "phone_number": "(408) 777-2343",
        "email_address": "contact@callpal.app"
   }
    
Acuity platform:
[
    {
    "firstName": "Kate",
    "lastName": "null",
    "email": "contact@callpal.app",
    "phone": "(408) 777-2343",
    "notes": ""
}
]  

 ```

**Key Differences:**
- Square returns `id`, Acuity may not
- Phone format: Square uses E.164, Acuity uses US display format
- Empty values: Square uses `""`, Acuity uses `"null"` (string)

**API References:**
- Square Create Customer: https://developer.squareup.com/reference/square/customers-api/create-customer
- Acuity Create Client: https://developers.acuityscheduling.com/reference/post-clients
---

## 3.3 Search Customers

### 3.3.1 Platform Comparison

| Platform | Endpoint | Method |
|----------|----------|--------|
| **Our Backend** | `/search/:agentId` | GET |
| **Square** | `/v2/customers/search` | POST |
| **Acuity** | `/api/v1/clients` | GET |

**Note:** Our backend requires `agentId` in URL path; search filters passed as query parameters.

### 3.3.2 Parameters

#### Our Backend Query Parameter Handling

**Important:** Unlike Create Customer endpoint, Search does NOT normalize field names. The backend accepts all query parameters and passes them to the active integration with minimal processing.

**Parameter Processing:**
- ✅ `limit` → Converted to integer
- ✅ `count` → Converted to boolean
- ⚠️ `phone`/`phone_number` → Platform-specific format conversion (see below)
- ⚠️ All other parameters → Passed as-is (no field name normalization)

**Accepted Parameters (Platform-Dependent):**

| Parameter | Type | Square Support | Acuity Support | Backend Processing |
|-----------|------|----------------|----------------|-------------------|
| `search` | string | ❌ No | ✅ Yes | Passed as-is |
| `phone`, `phone_number` | string | ✅ Yes (as `filter.phone_number.exact`) | ✅ Yes (as `phone`) | **Square**: Passed as-is <br> **Acuity**: Converts to digits only |
| `email`, `email_address` | string | ✅ Yes (as `filter.email_address.exact`) | ✅ Yes (as `email`) | Passed as-is |
| `firstName`, `given_name` | string | ✅ Yes (as `filter.given_name.exact`) | ✅ Yes (as `firstName`) | Passed as-is |
| `lastName`, `family_name` | string | ✅ Yes (as `filter.family_name.exact`) | ✅ Yes (as `lastName`) | Passed as-is |
| `limit` | integer | ✅ Yes | ❌ No | Converted from string to integer |
| `count` | boolean | ✅ Yes | ❌ No | Converted from string to boolean |
| `cursor` | string | ✅ Yes | ❌ No | Passed as-is |

**Why Minimal Field Name Normalization?**
- Search is a flexible query operation with platform-specific capabilities
- Backend uses "pass-through" approach for maximum flexibility
- Each platform handles its own parameter names
- You can use either naming convention (e.g., `phone` or `phone_number`), backend routes to correct platform

**Phone Number Handling:**
```
User Input (any format)
    ↓
Square: Keeps original format (expects E.164: +14087775678)
Acuity: Strips +1, converts to digits only (4087775678)
```

**Example:**
```
// If connected to Square
GET /search/AGENT_ID?phone_number=%2B14087775678&limit=10
→ Backend passes: phone_number = "+14087775678"
→ Square receives: filter.phone_number.exact = "+14087775678", limit = 10

// If connected to Acuity
GET /search/AGENT_ID?phone=%2B14087775678
→ Backend converts: "+14087775678" → "4087775678"
→ Acuity receives: phone = "4087775678"
```

#### Platform Parameter Comparison

| Parameter | How to Send | Square Interprets As | Acuity Interprets As | Notes |
|-----------|-------------|---------------------|---------------------|-------|
| **General Search** | `?search=keyword` | ❌ Ignored | ✅ `search=keyword` | Acuity only: searches firstName, lastName, phone, email |
| **Phone** | `?phone=...` or `?phone_number=...` | `filter.phone_number.exact` | `phone` (digits only) | Backend auto-converts for Acuity |
| **Email** | `?email=...` or `?email_address=...` | `filter.email_address.exact` | `email` | Use either name |
| **First Name** | `?firstName=...` or `?given_name=...` | `filter.given_name.exact` | `firstName` | Use either name |
| **Last Name** | `?lastName=...` or `?family_name=...` | `filter.family_name.exact` | `lastName` | Use either name |
| **Limit** | `?limit=10` | `limit: 10` | ❌ Ignored | Square only, converted to int |
| **Count** | `?count=true` | `count: true` | ❌ Ignored | Square only, converted to boolean |
| **Cursor** | `?cursor=xyz` | `cursor: "xyz"` | ❌ Ignored | Square only, for pagination |

**Backend Limitation:** ⚠️ Always returns maximum 1 result (`.slice(0, 1)`), regardless of `limit` parameter

**Legend:**
- ✅ = Supported
- ❌ = Not supported

**Platform-Specific Requirements:**

**Our Backend Rule:**
- All filters are optional
- Returns maximum 1 result (backend limitation using `.slice(0, 1)`)
- **Phone format**: Accepts any format, converts appropriately for each platform

**Square Rule:**
- All filters are optional
- Search request sent as POST with body (not query params)
- Supports advanced filtering with exact match
- **Phone format**: Expects E.164 (`+14087775678`)
- Returns multiple results based on `limit`

**Acuity Rule:**
- All filters are optional
- `search` parameter searches across all fields
- **Phone format**: Backend auto-converts to digits only (`4087775678`)
- Returns all matching results (no pagination)

### 3.3.3 Response Examples

**Our Backend vs Square**
```json
// Our Backend
{
  "status": "success",
  "provider": "square",
  "data": {
    "customers": [
      {
        "id": "8J30E6SD56G1Q0ENXVAX7ERWZ8",
        "given_name": "Amy",
        "family_name": "",
        "phone_number": "+13136667878",
        "email_address": ""
      }
    ]
  }
}

// Square Platform
{
  "customers": [
    {
      "id": "8J30E6SD56G1Q0ENXVAX7ERWZ8",
      "created_at": "2018-01-23T20:21:54.859Z",
      "updated_at": "2020-04-20T10:02:43.083Z",
      "given_name": "Amy",
      "family_name": "",
      "email_address": "james@example.com",
      "address": {
        "address_line_1": "505 Electric Ave",
        "address_line_2": "Suite 600",
        "locality": "New York",
        "administrative_district_level_1": "NY",
        "postal_code": "10003",
        "country": "US"
      },
      "phone_number": "+1-313-666-7878",
      "reference_id": "YOUR_REFERENCE_ID_2",
      "preferences": {
        "email_unsubscribed": false
      },
      "creation_source": "DIRECTORY",
      "version": 7
    },
    {
      "id": "A9641GZW2H7Z56YYSD41Q12HDW",
      "created_at": "2018-01-30T14:10:54.859Z",
      "updated_at": "2018-03-08T18:25:21.342Z",
      "given_name": "Amelia",
      "family_name": "Earhart",
      "email_address": "amelia@example.com",
      "phone_number": "+1-212-555-9238",
      "note": "a customer",
      "version": 1
    }
  ],
  "cursor": "9dpS093Uy12AzeE"
}
```

**Our Backend vs Acuity**
```json
// Our Backend
{
    "status": "success",
    "provider": "acuity",
    "data": {
        "customers": [
            {
                "given_name": "Kate",
                "family_name": "Liu",
                "phone_number": "(408) 777-2343",
                "email_address": "contact@callpal.app"
            }
        ]
    }
}

// Acuity Platform
[
    {
        "firstName": "Kate",
        "lastName": "Liu",
        "email": "contact@callpal.app",
        "phone": "(408) 777-2343",
        "notes": ""
    },
    {
        "firstName": "Jane",
        "lastName": "McTest",
        "email": "jane.mctest@example.com",
        "phone": "(123) 555-0101",
        "notes": ""
    },
    {
        "firstName": "Kate",
        "lastName": "Wong",
        "email": "contact@callpal.app",
        "phone": "+14087772345",
        "notes": ""
    }
]
```

**No Results Response**
```json
{
  "status": "not_exist",
  "provider": "square",
  "data": []
}
```

**Key Differences:**
- Our backend returns normalized format with `{ status, provider, data }` wrapper
- Our backend limits to 1 result, platforms can return multiple
- Square returns `id` and metadata (created_at, updated_at, version), Acuity returns numeric `id`
- Phone format: Square uses E.164 (`+13136667878`), Acuity uses display format (`(408) 222-3456`)
- Empty values: Square uses `""`, Acuity may use `""`
- Square includes cursor for pagination, Acuity returns all matches

**API References:**
- Square Search Customers: https://developer.squareup.com/reference/square/customers-api/search-customers
- Square List Customers: https://developer.squareup.com/reference/square/customers-api/list-customers
- Acuity List Clients: https://developers.acuityscheduling.com/reference/clients


---

## 3.4 Search by Phone

### 3.4.1 Platform Comparison

| Platform | Endpoint | Method |
|----------|----------|--------|
| **Our Backend** | `/search-by-phone/:agentId` | GET |
| **Square** | `/v2/customers/search` (with phone filter) | POST |
| **Acuity** | `/api/v1/clients?phone=...` | GET |

**Note:** Our backend requires `agentId` in URL path; phone number passed as query parameter.
**Key Differences:**
- Our backend returns normalized format with `{ status, provider, data }` wrapper
- Our backend limits to 1 result maximum
- Phone format: Square returns E.164 (`+14087775678`), Acuity returns display format (`(408) 777-5678`)
- Square includes `id` and metadata, Acuity includes numeric `id`
- Empty values: Square uses `""`, Acuity uses `"null"` (string) or `contact@callpal.app` for email

### 3.4.2 Parameters

#### Our Backend 

**Required Fields:**
- `phone` or `phone_number` * - Customer phone number in E.164 format (`+14087775678`), must be URL encoded as `%2B14087775678`

**Phone Format Flow:**
1. **User Input**: E.164 URL encoded → `%2B14087775678`
2. **Backend Receives**: E.164 decoded → `+14087775678`
3. **Sent to Square**: E.164 (unchanged) → `+14087775678`
4. **Sent to Acuity**: Digits only (strips `+1`) → `4087775678`

*\* = Required field*

#### Platform Parameter Comparison

| Parameter | Our Backend Input | Square API | Acuity API | Backend Processing |
|-----------|-------------------|------------|------------|-------------------|
| **Phone Number** | ✅ **Required** <br> Accepts: `phone`, `phone_number` <br> Format: E.164 URL encoded | ✅ **Required** <br> Field: `query.filter.phone_number.exact` <br> Format: E.164 | ✅ **Required** <br> Field: `phone` <br> Format: Digits only | **Square**: Keeps `+14087775678` <br> **Acuity**: Converts to `4087775678` |

**Legend:**
- ✅ = Required

**Platform-Specific Requirements:**

**Our Backend Rule:**
- Must provide: `phone` or `phone_number` parameter
- Input format: E.164 (`+14087775678`) URL encoded as `%2B14087775678`
- Returns: Maximum 1 result (`.slice(0, 1)`)
- Auto-converts phone format based on active platform

**Square Rule:**
- Uses `POST /v2/customers/search` endpoint
- Phone sent in request body as `query.filter.phone_number.exact`
- Format: E.164 (`+14087775678`)
- Exact match required

**Acuity Rule:**
- Uses `GET /api/v1/clients?phone=...` endpoint
- Phone passed as query parameter
- Format: Digits only (`4087775678`)
- Backend automatically removes `+1` prefix when calling Acuity


### 3.4.3 Response Examples

**Our Backend vs Square**
```json
// Our Backend
{
  "status": "success",
  "provider": "square",
  "data": {
    "customers": [
      {
        "id": "GC0DEN17NBB8PSXV7H26NV4DJR",
        "given_name": "Ellen",
        "family_name": "",
        "phone_number": "+14087775678",
        "email_address": ""
      }
    ]
  }
}

// Square Platform
{
  "customers": [
    {
      "id": "GC0DEN17NBB8PSXV7H26NV4DJR",
      "created_at": "2024-01-15T10:30:00.000Z",
      "updated_at": "2024-01-15T10:30:00.000Z",
      "given_name": "Ellen",
      "family_name": "",
      "email_address": "",
      "phone_number": "+14087775678",
      "preferences": {
        "email_unsubscribed": false
      },
      "creation_source": "THIRD_PARTY",
      "version": 1
    }
  ]
}
```

**Our Backend vs Acuity**
```json
// Our Backend
{
  "status": "success",
  "provider": "acuity",
  "data": {
    "customers": [
      {
        "given_name": "Ellen",
        "family_name": "null",
        "phone_number": "(408) 777-5678",
        "email_address": "contact@callpal.app"
      }
    ]
  }
}

// Acuity Platform
[
  {
    "firstName": "Ellen",
    "lastName": "null",
    "phone": "(408) 777-5678",
    "email": "contact@callpal.app",
    "notes": ""
  }
]
```

**No Results Response**
```json
{
  "status": "not_exist",
  "provider": "square",
  "data": []
}
```

**Error Response**
```json
{
  "status": "error",
  "error": "phone or phone_number query parameter is required"
}
```

**API References:**
- Square Search Customers: https://developer.squareup.com/reference/square/customers-api/search-customers
- Acuity List Clients: https://developers.acuityscheduling.com/reference/clients

---

## 3.5 Retrieve Customer by ID

### 3.5.1 Platform Comparison

| Platform | Endpoint | Method |
|----------|----------|--------|
| **Our Backend** | `/retrieve/:agentId/:customerId` | GET |
| **Square** | `/v2/customers/{customer_id}` | GET |
| **Acuity** | `/api/v1/clients?phone=...` | GET |

**Note:** Our backend requires both `agentId` and `customerId` in URL path. For Acuity integration, `customerId` should be the phone number.
**Key Differences:**
- Our backend returns normalized format with `{ status, provider, data }` wrapper
- Square returns full customer profile with metadata (created_at, updated_at, version)
- Square response nested in `customer` object, our backend flattens to `data`
- Acuity returns array (our backend extracts first result)
- Phone format: Square uses E.164, Acuity uses display format
- Square includes `id`, Acuity may not include `id` in normalized output
- Acuity retrieval is actually a search operation (workaround for missing feature)

### 3.5.2 Parameters

#### Our Backend 

**Required Fields:**
- `agentId` * - AI Agent identifier (in URL path)
- `customerId` * - Customer identifier (in URL path, format depends on platform)

**Customer ID Format by Platform:**
- **Square**: Square Customer ID (e.g., `GC0DEN17NBB8PSXV7H26NV4DJR`)
- **Acuity**: Phone number in E.164 format, URL encoded (e.g., `%2B14087775678`) ⚠️ Workaround for missing feature

**Request Examples:**
```
// For Square integration
GET /api/customers/retrieve/AGENT_ID/GC0DEN17NBB8PSXV7H26NV4DJR

// For Acuity integration (use phone as ID)
GET /api/customers/retrieve/AGENT_ID/%2B14087775678
```

*\* = Required field*

#### Platform Parameter Comparison

| Parameter | Our Backend Input | Square API | Acuity API | Backend Processing |
|-----------|-------------------|------------|------------|-------------------|
| **Customer ID** | ✅ **Required** <br> Accepts: `:customerId` in URL path <br> Square: Customer ID <br> Acuity: Phone number | ✅ **Required** <br> Field: `{customer_id}` <br> Format: Customer ID string | ❌ Not supported <br> No direct ID retrieval | **Square**: Direct lookup <br> **Acuity**: Searches by phone |
| **Phone Number** | Used as `customerId` for Acuity | ❌ Not used for retrieval | `phone` query param <br> Format: Digits only | Backend converts E.164 to digits for Acuity |

**Legend:**
- ✅ = Required / Supported
- ❌ = Not supported

**Platform-Specific Requirements:**

**Our Backend Rule:**
- Must provide: `agentId` and `customerId` in URL path
- Square integration: Use Square customer ID
- Acuity integration: Use phone number as `customerId` (E.164 format, URL encoded)
- Returns: Single customer object

**Square Rule:**
- Uses `GET /v2/customers/{customer_id}` endpoint
- Direct lookup by Square-generated customer ID
- Fast and efficient retrieval
- Returns full customer profile with metadata

**Acuity Rule:**
- ⚠️ **No direct ID retrieval endpoint**
- Backend workaround: Uses `GET /api/v1/clients?phone=...`
- Requires phone number instead of ID
- Backend converts E.164 phone (`+14087775678`) to digits-only format (`4087775678`)
- Returns first matching customer from search results

### 3.5.3 Response Examples

**Our Backend vs Square**
```json
// Our Backend
{
  "status": "success",
  "provider": "square",
  "data": {
    "id": "GC0DEN17NBB8PSXV7H26NV4DJR",
    "given_name": "Ellen",
    "family_name": "",
    "phone_number": "+14087775678",
    "email_address": ""
  }
}

// Square Platform
{
  "customer": {
    "id": "GC0DEN17NBB8PSXV7H26NV4DJR",
    "created_at": "2024-01-15T10:30:00.000Z",
    "updated_at": "2024-01-15T10:30:00.000Z",
    "given_name": "Ellen",
    "family_name": "",
    "email_address": "",
    "phone_number": "+14087775678",
    "address": {
      "address_line_1": "123 Main St",
      "locality": "San Francisco",
      "administrative_district_level_1": "CA",
      "postal_code": "94102",
      "country": "US"
    },
    "preferences": {
      "email_unsubscribed": false
    },
    "creation_source": "THIRD_PARTY",
    "version": 1
  }
}
```

**Our Backend vs Acuity**
```json
// Our Backend
{
  "status": "success",
  "provider": "acuity",
  "data": {
    "given_name": "Ellen",
    "family_name": "null",
    "phone_number": "(408) 777-5678",
    "email_address": "contact@callpal.app"
  }
}

// Acuity Platform (search result)
[
  {
    "firstName": "Ellen",
    "lastName": "null",
    "phone": "(408) 777-5678",
    "email": "contact@callpal.app",
    "notes": ""
  }
]
```

**Error Response - Customer Not Found**
```json
{
  "status": "error",
  "error": "Customer not found"
}
```

**Error Response - Invalid Customer ID**
```json
{
  "status": "error",
  "error": "Customer ID is required"
}
```

**API References:**
- Square Retrieve Customer: https://developer.squareup.com/reference/square/customers-api/retrieve-customer
- Acuity List Clients: https://developers.acuityscheduling.com/reference/clients (used as workaround)

---


## 4. Business Catalog APIs

The Business Catalog APIs provide a standardized interface for discovering and browsing business information across **Square** and **Acuity Scheduling** platforms.
Unlike the Customer APIs, these endpoints **do not normalize input** - they return unified output formats regardless of which platform is connected.

**Supported Operations:**

#### Business Discovery
1. **List All Locations** - Get complete catalog of business locations for an agent
2. **Search Nearby Locations** - Find business locations by proximity using zipcode (with caching support)
3. **List Service Types** - Get all available service types/appointment types

---

## Base URLs

| Platform | Base URL |
|----------|----------|
| **Our Backend** | `https://your-api.com/api/business-catalog` |
| **Square** | `https://connect.squareup.com` |
| **Acuity** | `https://acuityscheduling.com` |

## Authentication

All endpoints require Bearer Token authentication

## 4.1 Overview of Routes

| # | Backend Route | Method | Description | Returns |
|---|---------------|--------|-------------|---------|
| 1 | `/agent/:agentId/locations/list` | GET | Get all business locations | Array of locations |
| 2 | `/agent/:agentId/locations/search/nearby` | GET | Search locations by zipcode | Array of nearby locations (top 3) |
| 3 | `/agent/:agentId/services/list` | GET | Get all service types | Array of services/appointment types |

**Global Notes:**
- All endpoints require Bearer Token authentication
- All responses include `status` field and unified `data` wrapper
---

## 4.2 List All Locations

### 4.2.1 Platform Comparison

| Platform | Endpoint | Method | Note |
|----------|----------|--------|------|
| **Our Backend** | `/agent/:agentId/locations/list` | GET |Our backend normalizes both to consistent format with `{ id, name, street, city, state, timezone, coordinates }`
| **Square** | `/v2/locations` | GET |Provides details about all of the seller's locations, including those with an inactive status.Locations are listed alphabetically by name
| **Acuity** | N/A (constructed from business info) | N/A |We constructed to align with square response format

**Note:** 
- Our backend requires `agentId` in URL path
- Square: Calls Square Locations API directly
- Acuity: Acuity Scheduling does not have a native multi-location concept like Square. Acuity operates on a single-business model where:
    - Each Acuity account represents one business location
    - Calendars in Acuity represent individual service providers or resources, not physical locations
    - For compatibility with our location-based system, we construct a single location from the business profile data
    - Location ID format: `acuity-{businessId}`
    - Coordinates are derived from the business address using Google Maps Geocoding API


### 4.2.2 Platform API Calls

#### Square API Call
```javascript
// Square Locations API
GET https://connect.squareup.com/v2/locations
Headers:
  Authorization: Bearer {access_token}
  Square-Version: 2025-08-20
  Content-Type: application/json
```

**Square API Reference:** https://developer.squareup.com/reference/square/locations-api/list-locations

#### Acuity API Call
```javascript
// Acuity doesn't have a locations API
// Backend constructs location from business.address in database
// Uses Google Maps API to geocode address if coordinates not available
```

**Acuity Note:** Acuity uses a single business address, not multiple locations like Square franchises.

### 4.2.3 Parameters
**Platform-Specific Behavior:**

**Square:**
- Calls `/v2/locations` API
- Returns all locations for the merchant
- Includes full address, coordinates, timezone
- Can have multiple physical locations

**Acuity:**
- No native locations API
- Backend constructs location from business profile (`business.address`)
- Geocodes address using Google Maps if coordinates not in database
- Always returns single location
- Location ID format: `acuity-{businessId}`

### 4.2.4 Response Examples

**Our Backend  VS Square **
```
OurBackend(for square)
{
    "status": "success",
    "data": {
        "locations": [
            {
                "id": "L5J0TJ9CM5A7D",
                "name": "CallPal",
                "street": "7446 San Ramon Rd",
                "city": "Dublin",
                "state": "CA",
                "timezone": "America/Los_Angeles",
                "coordinates": {
                    "latitude": 37.7054608,
                    "longitude": -121.8978831
                },
                "integration": "square"
            }
        ],
        "totalCount": 1
    }
}


Square API Response

{
  "locations": [
    {
      "id": "L5J0TJ9CM5A7D",
      "name": "CallPal",
      "address": {
        "address_line_1": "7446 San Ramon Rd",
        "locality": "Dublin",
        "administrative_district_level_1": "CA",
        "postal_code": "94568-2043",
        "country": "US"
      },
      "timezone": "America/Los_Angeles",
      "capabilities": [
        "AUTOMATIC_TRANSFERS"
      ],
      "status": "ACTIVE",
      "created_at": "2025-09-12T19:42:43.337Z",
      "merchant_id": "ML37W5FN1K9A8",
      "country": "US",
      "language_code": "en-US",
      "currency": "USD",
      "business_name": "CallPal",
      "type": "MOBILE",
      "business_hours": {
        "periods": [
          {
            "day_of_week": "MON",
            "start_local_time": "09:00:00",
            "end_local_time": "17:00:00"
          },
          {
            "day_of_week": "TUE",
            "start_local_time": "09:00:00",
            "end_local_time": "17:00:00"
          },
          {
            "day_of_week": "WED",
            "start_local_time": "09:00:00",
            "end_local_time": "17:00:00"
          },
          {
            "day_of_week": "THU",
            "start_local_time": "09:00:00",
            "end_local_time": "17:00:00"
          },
          {
            "day_of_week": "FRI",
            "start_local_time": "09:00:00",
            "end_local_time": "17:00:00"
          }
        ]
      },
      "business_email": "xx.x@gmail.com",
      "coordinates": {
        "latitude": 37.7054608,
        "longitude": -121.8978831
      },
      "mcc": "7372"
    }
  ]
}
```


**Our Backend(Acuity) VS Acuity**

```json
Our Backend (Acuity)
{
    "status": "success",
    "data": {
        "locations": [
            {
                "id": "acuity-cmhfadqzk0005ry773lig0x3p",
                "name": "Anu's Beautique, Dublin",
                "address": "6046 Dougherty Rd, Dublin, CA 94568, United States",
                "businessHours": null,
                "timezone": "UTC",
                "capabilities": [
                    "appointments"
                ],
                "street": "6046 Dougherty Rd",
                "city": "Dublin",
                "state": "CA",
                "coordinates": {
                    "latitude": 37.7202463,
                    "longitude": -121.8676464
                },
                "integration": "acuity"
            }
        ],
        "totalCount": 1
    }
}
**Note**
Acuity does have Locations API

```
---

## 4.3 Search Nearby Locations

### 4.3.1 Endpoint Overview

| Our Backend | `/agent/:agentId/locations/search/nearby` | GET |
|-------------|-------------------------------------------|-----|

**Purpose:** Find the 3 nearest business locations to a customer's zipcode using proximity search.

**Key Features:**
-  Uses Google Maps Geocoding API to convert zipcode to coordinates
-  Calculates haversine distance (in miles) to each location
-  Returns top 3 nearest locations, sorted by distance
-  Works with both Square (multiple locations) and Acuity (single location)

### 4.3.2 Platform API Calls

#### Backend Process Flow
```javascript
// 1. Convert zipcode to coordinates
GET https://maps.googleapis.com/maps/api/geocode/json?address={zipcode}

// 2. Get all locations (calls getBusinessLocations internally)
// For Square:
GET https://connect.squareup.com/v2/locations

// For Acuity:
// Constructs location from business.address

// 3. Calculate distances using haversine formula
// 4. Sort by distance and return top 3
```


### 4.3.3 Response Examples

**Our Backend (Square) vs Square API **
```json
{
    "status": "success",
    "data": {
        "locations": [
            {
                "id": "LVTFQSKY6DKB3",
                "name": "dublin_store",
                "street": "3768 Fallon Rd",
                "city": "Walnut Creek",
                "state": "CA",
                "timezone": "America/Los_Angeles"
            },
            {
                "id": "L5J0TJ9CM5A7D",
                "name": "CallPal",
                "street": "7446 San Ramon Rd",
                "city": "Dublin",
                "state": "CA",
                "timezone": "America/Los_Angeles"
            }
        ],
        "totalCount": 2
    }
}

```

**Our Backend (Acuity)**
```json
{
    "status": "success",
    "data": {
        "locations": [
            {
                "id": "acuity-cmhfadqzk0005ry773lig0x3p",
                "name": "Anu's Beautique, Dublin",
                "street": "6046 Dougherty Rd",
                "city": "Dublin",
                "state": "CA",
                "timezone": "UTC"
            }
        ],
        "totalCount": 1
    }
}
```

**Error Response - Missing Zipcode**
```json
{
  "success": false,
  "error": "Zipcode is required"
}
```

**Error Response - No Locations Found**
```json
{
  "status": "error",
  "message": "No locations found",
  "data": {
    "locations": [],
    "totalCount": 0
  }
}
```



---

## 4.4 List Service Types

### 4.4.1 Endpoint Overview

| Our Backend | `/agent/:agentId/services/list` | GET |
|-------------|--------------------------------|-----|

**Purpose:** Retrieve all available service types/catalog items for a business.

**Key Features:**
-  Returns all services from active integration (Square or Acuity)
-  Standardized response format across both platforms
-  Includes service details: name, description, duration, price


### 4.4.2 Parameters

#### Our Backend

**Required Fields:**
- `agentId` * - AI Agent identifier (in URL path)

**Optional Fields:**
- None currently supported

*\* = Required field*

#### Platform Parameter Comparison

| Parameter | Our Backend | Square | Acuity | Backend Processing |
|-----------|-------------|--------|--------|-------------------|
| **Agent ID** | ✅ Required <br> Field: `:agentId` | ❌ Not used <br> (uses access token) | ❌ Not used <br> (uses access token) | Used to fetch integration credentials |
| **Object Types** | ❌ Not exposed | ⚙️ Fixed: `["ITEM"]` | ❌ Not applicable | Backend hardcodes for Square |
| **Product Types** | ❌ Not exposed | ⚙️ Fixed: `["APPOINTMENTS_SERVICE"]` | ❌ Not applicable | Backend filters to appointment services only |

**Legend:**
- ✅ = Required / Supported
- ⚙️ = Fixed/Hardcoded by backend
- ❌ = Not applicable

**Platform-Specific Behavior:**

**Square:**
- Uses Catalog API with search filters
- Filters to `APPOINTMENTS_SERVICE` product type only
- Returns service variations with pricing, duration, team member assignments
- Can have multiple variations per service (e.g., "Haircut - Short" vs "Haircut - Long")

**Acuity:**
- Uses Appointment Types API
- Appointment types = Services in Acuity
- Returns service name, description, duration, price, category
- Simpler structure than Square (no variations)

### 4.4.3 Response Examples

**Our Backend (Square) VS Square platform API**
```json
{
    "success": true,
    "data": [
        {
            "id": "6MDLR5PSL3VT3NW3OX2SPMTM",
            "service_variation_id": "HDS3W67HMO5TQB32P3K4EUBY",
            "service_variation_version": 1759213335949,
            "name": "Men's Haircut",
            "price": 35,
            "duration": 30
        },
        {
            "id": "QKE4RUD2VBTCR2GQK7P22D3T",
            "service_variation_id": "54367N2M2W33LLSSLTRPSTOP",
            "service_variation_version": 1759213335949,
            "name": "Single Process Color",
            "price": 95,
            "duration": 75
        },
        {
            "id": "ZZLAU7GNQJ4LBL4YAJ2OUDDD",
            "service_variation_id": "RPRWKRHYIDCIRCIJC5EB43PZ",
            "service_variation_version": 1759213335949,
            "name": "Women's Haircut (Long)",
            "price": 75,
            "duration": 60
        },
        {
            "id": "NQ5OM6JPLHH74UUM24WHSC65",
            "service_variation_id": "SMRYJQNFGHIJASTRTDG2DNBL",
            "service_variation_version": 1759213335949,
            "name": "Women's Haircut (Short/Medium)",
            "price": 55,
            "duration": 45
        }
    ]
}
 
Square Platform API

{
  "items": [
    {
      "type": "ITEM",
      "id": "6MDLR5PSL3VT3NW3OX2SPMTM",
      "updated_at": "2025-09-30T06:22:15.949Z",
      "created_at": "2025-09-15T20:14:03.423Z",
      "version": 1759213335949,
      "is_deleted": false,
      "present_at_all_locations": true,
      "item_data": {
        "name": "Men's Haircut",
        "description": "Basic men's haircut with wash and style",
        "is_taxable": true,
        "variations": [
          {
            "type": "ITEM_VARIATION",
            "id": "HDS3W67HMO5TQB32P3K4EUBY",
            "updated_at": "2025-09-30T06:22:15.949Z",
            "created_at": "2025-09-15T20:14:03.423Z",
            "version": 1759213335949,
            "is_deleted": false,
            "present_at_all_locations": true,
            "item_variation_data": {
              "item_id": "6MDLR5PSL3VT3NW3OX2SPMTM",
              "name": "Regular",
              "ordinal": 0,
              "pricing_type": "FIXED_PRICING",
              "price_money": {
                "amount": 3500,
                "currency": "USD"
              },
              "service_duration": 1800000,
              "available_for_booking": true,
              "transition_time": 0,
              "sellable": true,
              "stockable": true,
              "team_member_ids": [
                "TMwWZ6ga86JwL5aW",
                "TM5ctPKDQdXyuMpe"
              ],
              "channels": [
                "CH_LdDYBXaGcIBFnUxv70PWPV6hrrRlksEhCrCpvTR29945o"
              ]
            }
          }
        ],
        "product_type": "APPOINTMENTS_SERVICE",
        "description_html": "Basic men&#39;s haircut with wash and style",
        "description_plaintext": "Basic men's haircut with wash and style",
        "channels": [
          "CH_LdDYBXaGcIBFnUxv70PWPV6hrrRlksEhCrCpvTR29945o"
        ],
        "is_archived": false
      }
    },
    {
      "type": "ITEM",
      "id": "QKE4RUD2VBTCR2GQK7P22D3T",
      "updated_at": "2025-09-30T06:22:15.949Z",
      "created_at": "2025-09-15T20:14:03.423Z",
      "version": 1759213335949,
      "is_deleted": false,
      "present_at_all_locations": true,
      "item_data": {
        "name": "Single Process Color",
        "description": "All-over color application",
        "is_taxable": true,
        "variations": [
          {
            "type": "ITEM_VARIATION",
            "id": "54367N2M2W33LLSSLTRPSTOP",
            "updated_at": "2025-09-30T06:22:15.949Z",
            "created_at": "2025-09-15T20:14:03.423Z",
            "version": 1759213335949,
            "is_deleted": false,
            "present_at_all_locations": true,
            "item_variation_data": {
              "item_id": "QKE4RUD2VBTCR2GQK7P22D3T",
              "name": "Regular",
              "ordinal": 0,
              "pricing_type": "FIXED_PRICING",
              "price_money": {
                "amount": 9500,
                "currency": "USD"
              },
              "service_duration": 4500000,
              "available_for_booking": true,
              "transition_time": 0,
              "sellable": true,
              "stockable": true,
              "team_member_ids": [
                "TMwWZ6ga86JwL5aW",
                "TM5ctPKDQdXyuMpe"
              ],
              "channels": [
                "CH_LdDYBXaGcIBFnUxv70PWPV6hrrRlksEhCrCpvTR29945o"
              ]
            }
          }
        ],
        "product_type": "APPOINTMENTS_SERVICE",
        "description_html": "All-over color application",
        "description_plaintext": "All-over color application",
        "channels": [
          "CH_LdDYBXaGcIBFnUxv70PWPV6hrrRlksEhCrCpvTR29945o"
        ],
        "is_archived": false
      }
    },
    {
      "type": "ITEM",
      "id": "ZZLAU7GNQJ4LBL4YAJ2OUDDD",
      "updated_at": "2025-09-30T06:22:15.949Z",
      "created_at": "2025-09-15T20:14:03.423Z",
      "version": 1759213335949,
      "is_deleted": false,
      "present_at_all_locations": true,
      "item_data": {
        "name": "Women's Haircut (Long)",
        "description": "Cut and style for long hair",
        "is_taxable": true,
        "variations": [
          {
            "type": "ITEM_VARIATION",
            "id": "RPRWKRHYIDCIRCIJC5EB43PZ",
            "updated_at": "2025-09-30T06:22:15.949Z",
            "created_at": "2025-09-15T20:14:03.423Z",
            "version": 1759213335949,
            "is_deleted": false,
            "present_at_all_locations": true,
            "item_variation_data": {
              "item_id": "ZZLAU7GNQJ4LBL4YAJ2OUDDD",
              "name": "Regular",
              "ordinal": 0,
              "pricing_type": "FIXED_PRICING",
              "price_money": {
                "amount": 7500,
                "currency": "USD"
              },
              "service_duration": 3600000,
              "available_for_booking": true,
              "transition_time": 0,
              "sellable": true,
              "stockable": true,
              "team_member_ids": [
                "TMwWZ6ga86JwL5aW",
                "TM5ctPKDQdXyuMpe"
              ],
              "channels": [
                "CH_LdDYBXaGcIBFnUxv70PWPV6hrrRlksEhCrCpvTR29945o"
              ]
            }
          }
        ],
        "product_type": "APPOINTMENTS_SERVICE",
        "description_html": "Cut and style for long hair",
        "description_plaintext": "Cut and style for long hair",
        "channels": [
          "CH_LdDYBXaGcIBFnUxv70PWPV6hrrRlksEhCrCpvTR29945o"
        ],
        "is_archived": false
      }
    },
    {
      "type": "ITEM",
      "id": "NQ5OM6JPLHH74UUM24WHSC65",
      "updated_at": "2025-09-30T06:22:15.949Z",
      "created_at": "2025-09-15T20:14:03.423Z",
      "version": 1759213335949,
      "is_deleted": false,
      "present_at_all_locations": true,
      "item_data": {
        "name": "Women's Haircut (Short/Medium)",
        "description": "Cut and style for short to medium length hair",
        "is_taxable": true,
        "variations": [
          {
            "type": "ITEM_VARIATION",
            "id": "SMRYJQNFGHIJASTRTDG2DNBL",
            "updated_at": "2025-09-30T06:22:15.949Z",
            "created_at": "2025-09-15T20:14:03.423Z",
            "version": 1759213335949,
            "is_deleted": false,
            "present_at_all_locations": true,
            "item_variation_data": {
              "item_id": "NQ5OM6JPLHH74UUM24WHSC65",
              "name": "Regular",
              "ordinal": 0,
              "pricing_type": "FIXED_PRICING",
              "price_money": {
                "amount": 5500,
                "currency": "USD"
              },
              "service_duration": 2700000,
              "available_for_booking": true,
              "transition_time": 0,
              "sellable": true,
              "stockable": true,
              "team_member_ids": [
                "TMwWZ6ga86JwL5aW",
                "TM5ctPKDQdXyuMpe"
              ],
              "channels": [
                "CH_LdDYBXaGcIBFnUxv70PWPV6hrrRlksEhCrCpvTR29945o"
              ]
            }
          }
        ],
        "product_type": "APPOINTMENTS_SERVICE",
        "description_html": "Cut and style for short to medium length hair",
        "description_plaintext": "Cut and style for short to medium length hair",
        "channels": [
          "CH_LdDYBXaGcIBFnUxv70PWPV6hrrRlksEhCrCpvTR29945o"
        ],
        "is_archived": false
      }
    }
  ],
  "cursor": "",
  "matched_variation_ids": [
    "HDS3W67HMO5TQB32P3K4EUBY",
    "54367N2M2W33LLSSLTRPSTOP",
    "RPRWKRHYIDCIRCIJC5EB43PZ",
    "SMRYJQNFGHIJASTRTDG2DNBL"
  ]
}


```


**Our Backend (Acuity) vs Acuity Platform API**
```json
Our Backend (Acuity)
{
    "success": true,
    "data": [
        {
            "id": 84431042,
            "name": "men's haircut",
            "description": "",
            "duration": 50,
            "price": "45.00",
            "category": "",
            "source": "acuity"
        },
        {
            "id": 84431573,
            "name": "women's haircut",
            "description": "",
            "duration": 30,
            "price": "55.00",
            "category": "",
            "source": "acuity"
        },
        {
            "id": 84431588,
            "name": "color",
            "description": "",
            "duration": 60,
            "price": "120.00",
            "category": "",
            "source": "acuity"
        }
    ]
}
```

**API References:**
- Square Catalog API: https://developer.squareup.com/reference/square_2025-08-20/catalog-api
- Acuity Appointment Types: https://developers.acuityscheduling.com/reference/appointment-types

---






















## 5.Unified Appointments APIs

The Unified Appointments APIs provide a standardized interface for managing appointments and bookings across **Square Appointments** and **Acuity Scheduling** platforms. 
This layer abstracts the differences between provider-specific APIs, allowing you to use a single set of endpoints regardless of which scheduling platform is connected to your agent.

**Supported Operations:**

#### Availability & Discovery

1. **Search Availability** - Find available time slots for appointments
2. **List Team Members** - Get bookable staff members and their profiles

#### Booking Management
3. **Create Booking** - Book a new appointment
4. **Retrieve Booking** - Get details of a specific appointment
5. **Cancel Booking** - Cancel a specific appointment

#### Customer-Specific Operations
6. **Customer Bookings** - List all future appointments for a specific customer
7. **Cancel by Customer** - Cancel all appointments for a specific customer

## Base URLs

| Platform | Base URL |
|----------|----------|
| **Our Backend** | `https://your-api.com/api/appointments` |
| **Square** | `https://connect.squareup.com` |
| **Acuity** | `https://acuityscheduling.com` |

## Authentication






 


  
 

  
 
 
  










 
 





### 10. 🔗 External Documentation

- **[Square OAuth Guide](https://developer.squareup.com/docs/oauth-api/overview)**  
 
- **[Acuity Scheduling API Authentication](https://developers.acuityscheduling.com/reference/authentication)**  
 
