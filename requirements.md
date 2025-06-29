# Backend API Specification for "StayHub" (Airbnb Clone)

This document outlines the technical specifications for the backend API of the StayHub platform. It details the endpoints, data models, validation rules, and performance criteria for core features.

## Table of Contents
1.  [Global Conventions](#1-global-conventions)
2.  [User Authentication](#2-user-authentication)
3.  [Property Management](#3-property-management)
4.  [Booking System](#4-booking-system)

---

## 1. Global Conventions

### 1.1. Base URL
All API endpoints are prefixed with the following base URL. Versioning is included to allow for future iterations.

```
https://api.stayhub.com/v1
```

### 1.2. Authentication
Most endpoints require authentication. The API uses JSON Web Tokens (JWT) for stateless authentication.

-   The client must include the JWT in the `Authorization` header for all protected routes.
-   Format: `Authorization: Bearer <your_jwt_token>`
-   Endpoints that do not require authentication are explicitly marked as `Authentication: None`.

### 1.3. Data Format
All data is sent and received in JSON format.

-   Request headers must include `Content-Type: application/json`.
-   Response bodies will be `application/json`.

### 1.4. Standard Error Response
Errors are communicated using standard HTTP status codes. The response body will contain a JSON object with more details.

**Example Error Response (`4xx` or `5xx`):**
```json
{
  "error": "Validation Error",
  "details": "The 'email' field must be a valid email address."
}
```

| Status Code     | Meaning               | When to Use                                             |
| --------------- | --------------------- | ------------------------------------------------------- |
| `400 Bad Request` | Invalid client input  | Failed validation, malformed request.                   |
| `401 Unauthorized`| Missing/Invalid Token | JWT is missing, expired, or invalid.                    |
| `403 Forbidden`   | Insufficient Permissions | User is authenticated but not allowed to perform the action. |
| `404 Not Found`   | Resource Not Found    | The requested resource (e.g., property, user) does not exist. |
| `409 Conflict`    | Resource Conflict     | Creating a resource that already exists (e.g., duplicate email). |
| `500 Server Error`| Internal Server Error | An unexpected error occurred on the server.             |

### 1.5. Pagination
For endpoints that return a list of resources, pagination is supported via query parameters.

-   `limit`: (Integer, default: 20, max: 100) The number of items to return.
-   `offset`: (Integer, default: 0) The number of items to skip.

**Example Paginated Response:**
```json
{
  "data": [ ... list of items ... ],
  "pagination": {
    "total": 150,
    "limit": 20,
    "offset": 0
  }
}
```

---

## 2. User Authentication

Handles user registration, login, and profile management.

### 2.1. Register New User
-   **Endpoint:** `POST /auth/register`
-   **Description:** Creates a new user account.
-   **Authentication:** `None`

**Request Body:**
```json
{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com",
  "password": "AStrongPassword123!"
}
```

**Validation Rules:**
-   `firstName`: Required, String, 2-50 characters.
-   `lastName`: Required, String, 2-50 characters.
-   `email`: Required, String, must be a valid email format, must be unique in the database.
-   `password`: Required, String, minimum 8 characters, at least one uppercase, one lowercase, one number, and one special character.

**Success Response (`201 Created`):**
Returns the newly created user object (without the password) and a JWT.
```json
{
  "user": {
    "id": "cuid_12345",
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "createdAt": "2023-10-27T10:00:00Z"
  },
  "token": "ey..."
}
```

**Error Responses:**
-   `400 Bad Request`: If validation fails.
-   `409 Conflict`: If the email address is already in use.

**Performance Criteria:**
-   P95 latency < 500ms.

### 2.2. Login User
-   **Endpoint:** `POST /auth/login`
-   **Description:** Authenticates a user and returns a JWT.
-   **Authentication:** `None`

**Request Body:**
```json
{
  "email": "john.doe@example.com",
  "password": "AStrongPassword123!"
}
```

**Validation Rules:**
-   `email`: Required, must be a valid email format.
-   `password`: Required.

**Success Response (`200 OK`):**
```json
{
  "user": {
    "id": "cuid_12345",
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com"
  },
  "token": "ey..."
}
```

**Error Responses:**
-   `400 Bad Request`: If validation fails.
-   `401 Unauthorized`: If credentials are incorrect.

**Performance Criteria:**
-   P95 latency < 300ms.

### 2.3. Get Current User Profile
-   **Endpoint:** `GET /users/me`
-   **Description:** Retrieves the profile of the currently authenticated user.
-   **Authentication:** `Required (JWT)`

**Request Body:** None.

**Success Response (`200 OK`):**
```json
{
  "id": "cuid_12345",
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com",
  "createdAt": "2023-10-27T10:00:00Z"
}
```

**Error Responses:**
-   `401 Unauthorized`: If the user is not authenticated.

**Performance Criteria:**
-   P95 latency < 150ms.

---

## 3. Property Management

Handles the creation, retrieval, updating, and deletion of properties.

### 3.1. Create a New Property
-   **Endpoint:** `POST /properties`
-   **Description:** Adds a new property listing. The authenticated user becomes the host.
-   **Authentication:** `Required (JWT)`

**Request Body:**
```json
{
  "title": "Cozy Downtown Apartment",
  "description": "A beautiful apartment in the heart of the city.",
  "address": {
    "street": "123 Main St",
    "city": "Metropolis",
    "state": "CA",
    "zipCode": "90210",
    "country": "USA"
  },
  "propertyType": "Apartment",
  "pricePerNight": 150.00,
  "maxGuests": 4,
  "numBeds": 2,
  "numBaths": 1,
  "amenities": ["Wi-Fi", "Kitchen", "Air Conditioning"]
}
```

**Validation Rules:**
-   All fields are required.
-   `title`: String, 5-100 characters.
-   `description`: String, 10-1000 characters.
-   `address`: Object with required string fields.
-   `propertyType`: Enum, must be one of `["Apartment", "House", "Villa", "Cabin"]`.
-   `pricePerNight`: Number, must be greater than 0.
-   `maxGuests`, `numBeds`, `numBaths`: Integer, must be greater than 0.
-   `amenities`: Array of strings.

**Success Response (`201 Created`):**
Returns the full property object, including the generated `id` and `hostId`.
```json
{
  "id": "prop_abcde",
  "hostId": "cuid_12345",
  "title": "Cozy Downtown Apartment",
  "description": "A beautiful apartment in the heart of the city.",
  ...
  "createdAt": "2023-10-27T11:00:00Z"
}
```

**Performance Criteria:**
-   P95 latency < 400ms.

### 3.2. List and Search Properties
-   **Endpoint:** `GET /properties`
-   **Description:** Retrieves a paginated list of all properties. Supports filtering.
-   **Authentication:** `None`

**Query Parameters (Optional):**
-   `city`: (String) Filter by city.
-   `minPrice`: (Number) Filter by minimum price per night.
-   `maxPrice`: (Number) Filter by maximum price per night.
-   `numGuests`: (Integer) Filter by minimum guest capacity.
-   `limit`, `offset` for pagination.

**Success Response (`200 OK`):**
```json
{
  "data": [
    {
      "id": "prop_abcde",
      "title": "Cozy Downtown Apartment",
      "city": "Metropolis",
      "propertyType": "Apartment",
      "pricePerNight": 150.00,
      "mainImageUrl": "https://cdn.stayhub.com/prop_abcde/main.jpg"
    },
    ...
  ],
  "pagination": {
    "total": 50,
    "limit": 20,
    "offset": 0
  }
}
```

**Performance Criteria:**
-   P95 latency < 250ms. Must be highly optimized with database indexing on filterable fields.

### 3.3. Get a Single Property
-   **Endpoint:** `GET /properties/{id}`
-   **Description:** Retrieves detailed information for a single property.
-   **Authentication:** `None`

**Path Parameters:**
-   `id`: The unique identifier of the property.

**Success Response (`200 OK`):**
Returns the full property object, including host details.
```json
{
  "id": "prop_abcde",
  "title": "Cozy Downtown Apartment",
  ...
  "amenities": ["Wi-Fi", "Kitchen", "Air Conditioning"],
  "host": {
    "id": "cuid_12345",
    "firstName": "John"
  },
  "images": [
    "https://cdn.stayhub.com/prop_abcde/1.jpg",
    "https://cdn.stayhub.com/prop_abcde/2.jpg"
  ]
}
```

**Error Responses:**
-   `404 Not Found`: If property with `{id}` does not exist.

**Performance Criteria:**
-   P95 latency < 200ms.

### 3.4. Update a Property
-   **Endpoint:** `PATCH /properties/{id}`
-   **Description:** Updates details for a specific property.
-   **Authentication:** `Required (JWT)`

**Authorization:**
-   The authenticated user must be the host of the property.

**Request Body:**
Contains any subset of the fields from the "Create Property" request.
```json
{
  "pricePerNight": 160.00,
  "description": "A newly renovated, beautiful apartment in the heart of the city."
}
```

**Success Response (`200 OK`):**
Returns the updated full property object.

**Error Responses:**
-   `403 Forbidden`: If the user is not the host of the property.
-   `404 Not Found`: If the property does not exist.

**Performance Criteria:**
-   P95 latency < 350ms.

---

## 4. Booking System

Manages the process of booking properties.

### 4.1. Create a New Booking
-   **Endpoint:** `POST /bookings`
-   **Description:** Creates a booking for a property on behalf of the authenticated user.
-   **Authentication:** `Required (JWT)`

**Request Body:**
```json
{
  "propertyId": "prop_abcde",
  "startDate": "2024-08-10",
  "endDate": "2024-08-15"
}
```

**Validation Rules:**
-   `propertyId`: Required, must correspond to an existing property.
-   `startDate`, `endDate`: Required, valid `YYYY-MM-DD` format.
-   `endDate` must be after `startDate`.
-   `startDate` cannot be in the past.
-   The authenticated user cannot book their own property.
-   **Crucial:** The property must not have any conflicting bookings for the requested date range (inclusive). This check must be atomic.

**Success Response (`201 Created`):**
Returns the new booking object, with server-calculated `totalPrice`.
```json
{
  "id": "book_xyz789",
  "guestId": "cuid_67890",
  "propertyId": "prop_abcde",
  "startDate": "2024-08-10",
  "endDate": "2024-08-15",
  "totalPrice": 750.00,
  "status": "confirmed",
  "createdAt": "2023-10-27T12:00:00Z"
}
```

**Error Responses:**
-   `400 Bad Request`: For invalid dates or other validation failures.
-   `403 Forbidden`: If the user attempts to book their own property.
-   `404 Not Found`: If `propertyId` does not exist.
-   `409 Conflict`: If the dates are unavailable for booking.

**Performance Criteria:**
-   P95 latency < 600ms due to the date conflict check.

### 4.2. List User's Bookings
-   **Endpoint:** `GET /bookings/my-bookings`
-   **Description:** Retrieves a list of all bookings made by the currently authenticated user.
-   **Authentication:** `Required (JWT)`

**Query Parameters (Optional):**
-   `status`: (String) Filter by booking status (`confirmed`, `cancelled`, `completed`).
-   `limit`, `offset` for pagination.

**Success Response (`200 OK`):**
```json
{
  "data": [
    {
      "id": "book_xyz789",
      "property": {
        "id": "prop_abcde",
        "title": "Cozy Downtown Apartment",
        "city": "Metropolis"
      },
      "startDate": "2024-08-10",
      "endDate": "2024-08-15",
      "totalPrice": 750.00,
      "status": "confirmed"
    }
  ],
  "pagination": {
    "total": 5,
    "limit": 20,
    "offset": 0
  }
}
```
**Performance Criteria:**
-   P95 latency < 300ms.

### 4.3. List Bookings for a Host's Property
-   **Endpoint:** `GET /properties/{id}/bookings`
-   **Description:** Retrieves all bookings for a specific property.
-   **Authentication:** `Required (JWT)`

**Authorization:**
-   The authenticated user must be the host of the property with `{id}`.

**Success Response (`200 OK`):**
Returns a list of bookings for the specified property.
```json
{
  "data": [
    {
      "id": "book_xyz789",
      "guest": {
        "id": "cuid_67890",
        "firstName": "Jane"
      },
      "startDate": "2024-08-10",
      "endDate": "2024-08-15",
      "totalPrice": 750.00,
      "status": "confirmed"
    }
  ],
  "pagination": { ... }
}
```

**Error Responses:**
-   `403 Forbidden`: If the user is not the host of the property.
-   `404 Not Found`: If the property does not exist.

**Performance Criteria:**
-   P95 latency < 300ms.
