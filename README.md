# 🏛️ Museo API

![.NET](https://img.shields.io/badge/.NET-8.0%20%7C%209.0-512BD4?style=flat&logo=dotnet)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat&logo=docker)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-316192?style=flat&logo=postgresql)
![Security](https://img.shields.io/badge/Security-JWT%20%2B%20Refresh%20Token-green)
![Pattern](https://img.shields.io/badge/Architecture-Repository%20Pattern-orange)

This document provides a detailed technical overview of the "Museo" RESTful API, designed for the integral management of a digital art ecosystem. It delves into architectural decisions, security structure, and offers a complete guide to every available endpoint.

---

## 👥 Developer

| **Sebastian Alejandro Arce Antezana |

---

## 👨‍💻 About This Project & My Role

This project was initially built as a collaborative capstone project. As the **Backend Developer**, my primary focus was designing a robust, scalable, and secure architecture. 

**My Key Contributions:**
* **Architectural Design:** Implemented the Repository Pattern and Service Layer to enforce the Single Responsibility Principle (SRP).
* **Data Flow & Serialization:** Solved complex Entity Framework circular reference issues using precise DTO projections.
* **Security Implementation:** Engineered a stateless authentication system using JWT and Refresh Tokens with BCrypt password hashing.
* **DevOps:** Containerized the application and database using Docker Compose for seamless local development and deployment.

## 🏗️ Architecture and Design Decisions

The API has been built upon a clean, layered architecture, prioritizing maintainability, scalability, and security.

### 1. Repository Pattern & Service Layer
Logic is strictly separated to adhere to the Single Responsibility Principle (SRP).
*   **Repository Layer (`Repositories/`):** Acts as an abstraction over the Entity Framework Core `DbContext`. Its sole responsibility is executing CRUD operations against the database. It utilizes `Eager Loading` via `Include`/`ThenInclude` to optimize queries requiring relational data.
*   **Service Layer (`Services/`):** Orchestrates business logic. It invokes repositories to retrieve entities and transforms them into DTOs, applies complex validations (e.g., enforcing specific business rules), and handles transaction logic.

### 2. Advanced Relationship & Cycle Management (DTOs)
One of the major challenges in APIs with complex data models is circular references during JSON serialization.
*   **Identified Problem:** A `Museum` has `Canvas` items, and a `Canvas` belongs to a `Museum`. Direct serialization (`Json(museum)`) causes a `JsonException` due to an infinite loop.
*   **Implemented Solution:** We adopted a **projection approach using Data Transfer Objects (DTOs)**. Instead of simply ignoring properties (`[JsonIgnore]`), specific view models are created for each endpoint. This not only resolves the cycle but also allows for precise and efficient API response modeling, sending only the data the client needs and hiding the internal database structure.

### 3. Stateless Security (JWT & Refresh Tokens)
Security is based on a stateless authentication scheme to guarantee horizontal scalability.
*   **Access Token (JWT):** Short-lived token (60 min) signed with a secret key (HMAC-SHA256). It contains `Claims` (such as `UserId` and `Role`) that allow the backend to identify and authorize the user on every request.
*   **Refresh Token:** Long-lived token (14 days), securely stored in the database and associated with the user. It is used to generate new Access Tokens without requiring the user to re-enter credentials.
*   **Additional Protection:** `Rate Limiting` was implemented in `Program.cs` to mitigate brute-force attacks on authentication endpoints.

---

## 🚀 Deployment and Execution Guide

### Option 1: Docker (Recommended Production Environment)
The project is fully containerized for fast and consistent deployment.

1.  **Clone Repository:**
    ```
    git clone https://github.com/Majo2505/Museo_Grupo7.git && cd Museo_Grupo7
    ```

2.  **Start Containers:**
    This command will build the API image and start the container alongside the PostgreSQL database.
    ```
    docker-compose up -d --build
    ```

3.  **Apply Migrations (First Time Only):**
    Even if the API starts, the database will be empty. To create the tables, run:
    ```
    dotnet ef database update
    ```
    *(Ensure the ConnectionString in `appsettings.Development.json` points to `localhost` and port `5432` if running this command from outside Docker).*

4.  **Access:**
    *   **Swagger UI:** `http://localhost:8080/swagger`
    *   **Database:** Accessible at `localhost:5432`

### Option 2: Local (Development Environment)
1.  Verify that .NET SDK 8.0+ and PostgreSQL are installed.
2.  Configure the `ConnectionString` in `appsettings.Development.json` to point to your local Postgres instance.
3.  Open a terminal in the project root and run the following commands:
    ```
    dotnet ef database update # Creates/Updates DB schema
    dotnet run               # Starts the API
    ```

---

## 📚 Complete Endpoint Reference

### Authentication Module (`/api/Auth`)

This module manages registration, login, and token renewal.

---
#### **Endpoint: `POST /api/Auth/register`**
*   **Description:** Registers a new user in the system. Passwords are hashed using BCrypt. By default, the "Visitor" role is assigned.
*   **Security:** Public (`AllowAnonymous`).
*   **Request Body (`RegisterDto`):**
    ```
    {
      "username": "new_user",
      "email": "user@example.com",
      "password": "SecurePassword123!"
    }
    ```
*   **Response (201 Created):**
    Returns a `Location` header pointing to the created resource. The body is empty.
*   **Possible Errors:**
    *   `400 Bad Request`: If the email is already in use or data fails validation (e.g., weak password).

---
#### **Endpoint: `POST /api/Auth/login`**
*   **Description:** Authenticates a user with email and password. If credentials are valid, generates and returns an Access Token and a Refresh Token.
*   **Security:** Public (`AllowAnonymous`).
*   **Request Body (`LoginDto`):**
    ```
    {
      "email": "user@example.com",
      "password": "SecurePassword123!"
    }
    ```
*   **Response (200 OK - `LoginResponseDto`):**
    ```
    {
      "user": {
        "id": "a1b2c3d4-...",
        "username": "new_user",
        "email": "user@example.com"
      },
      "role": "Visitor",
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refreshToken": "long_random_string_for_refreshing...",
      "tokenType": "Bearer",
      "expiresIn": 3600
    }
    ```
*   **Possible Errors:**
    *   `401 Unauthorized`: If credentials are incorrect.

---
#### **Endpoint: `POST /api/Auth/refresh`**
*   **Description:** Allows obtaining a new Access Token using a valid, unrevoked Refresh Token.
*   **Security:** Public (`AllowAnonymous`).
*   **Request Body (`RefreshRequestDto`):**
    ```
    {
      "refreshToken": "long_random_string_for_refreshing..."
    }
    ```
*   **Response (200 OK - `LoginResponseDto`):**
    Returns the same structure as the Login endpoint, with renewed tokens.
*   **Possible Errors:**
    *   `401 Unauthorized`: If the Refresh Token is invalid, expired, or revoked.

---
### Comments Module (`/api/Comments`)

This module demonstrates access to protected resources and identity extraction from the token.

---
#### **Endpoint: `GET /api/Comments/canvas/{canvasId}`**
*   **Description:** Retrieves all comments associated with an artwork (`Canvas`), ordered from newest to oldest.
*   **Security:** Public (`AllowAnonymous`).
*   **Response (200 OK):**
    ```
    [
      {
        "id": "c1d2e3f4-...",
        "content": "A spectacular masterpiece!",
        "createdAt": "2025-11-28T10:05:00Z",
        "canvasId": "CANVAS_ID",
        "userId": "USER_ID",
        "username": "new_user"
      }
    ]
    ```

---
#### **Endpoint: `POST /api/Comments`**
*   **Description:** Creates a new comment for an artwork. The system automatically extracts the `UserId` from the JWT token, guaranteeing the author is the authenticated user.
*   **Security:** Protected (`[Authorize]`). Requires a valid `Bearer Token` in the `Authorization` header.
*   **Request Body (`CreateCommentDto`):**
    ```
    {
      "content": "The interplay of light and shadow is masterful.",
      "canvasId": "a1b2c3d4-..."
    }
    ```
*   **Response (201 Created):**
    Returns the newly created comment, including the `username` obtained from the user relationship.
    ```
    {
      "id": "f4e3d2c1-...",
      "content": "The interplay of light and shadow is masterful.",
      "createdAt": "2025-11-28T10:10:00Z",
      "canvasId": "a1b2c3d4-...",
      "userId": "AUTHENTICATED_USER_ID",
      "username": "new_user"
    }
    ```
*   **Possible Errors:**
    *   `401 Unauthorized`: If no valid token is provided.
    *   `404 Not Found`: If the `canvasId` does not exist.

---
#### **Endpoint: `PUT /api/Comments/{commentId}`**
*   **Description:** Updates the content of an existing comment. Users can only modify their own comments.
*   **Security:** Protected (`[Authorize]`). Validates that the token's `UserId` matches the comment's `UserId`.
*   **Request Body (`UpdateCommentDto`):**
    ```
    {
      "content": "Correction: The interplay of light and shadow is sublime."
    }
    ```
*   **Response (200 OK):** Returns the updated comment.
*   **Possible Errors:**
    *   `403 Forbidden`: If the user tries to modify a comment that does not belong to them.
    *   `404 Not Found`: If the `commentId` does not exist.

---
#### **Endpoint: `DELETE /api/Comments/{commentId}`**
*   **Description:** Deletes a comment. Only the comment author or an administrator can perform this action.
*   **Security:** Protected (`[Authorize]`). Validates authorship.
*   **Response (204 No Content):** Returns no body if deletion is successful.
*   **Possible Errors:**
    *   `403 Forbidden`: If the user is not the author.
    *   `404 Not Found`: If the `commentId` does not exist.

---

### Artist Module (`/api/Artist`)

This controller manages the artist catalog. It stands out for using DTO projections to list works for each artist without falling into recursion.

---
#### **Endpoint: `GET /api/Artist`**
*   **Description:** Retrieves the full list of registered artists. Includes a projection of their artwork titles (`CanvasTitles`), avoiding fetching the entire nested `Canvas` and `Museum` objects.
*   **Security:** Public (`AllowAnonymous`).
*   **Response (200 OK - `IEnumerable<ArtistResponseDto>`):**
    ```
    [
      {
        "id": "e4b3c2d1-...",
        "name": "Vincent van Gogh",
        "description": "Dutch Post-Impressionist painter.",
        "specialty": "Oil",
        "typeOfWork": "Post-Impressionism",
        "canvasTitles": [
          "The Starry Night",
          "Sunflowers"
        ]
      }
    ]
    ```

---
#### **Endpoint: `GET /api/Artist/{id}`**
*   **Description:** Gets details for a specific artist by their GUID.
*   **Security:** Public (`AllowAnonymous`).
*   **Response (200 OK - `ArtistResponseDto`):** Same structure as the list, for a single item.
*   **Possible Errors:**
    *   `404 Not Found`: If the ID does not exist.

---
#### **Endpoint: `POST /api/Artist`**
*   **Description:** Creates a new artist profile in the database.
*   **Security:** Protected (`[Authorize]`). Ideally restricted to `Admin` roles (depending on configured policies).
*   **Request Body (`CreateArtistDto`):**
    ```
    {
      "name": "Pablo Picasso",
      "description": "Spanish painter and sculptor, creator of Cubism.",
      "specialty": "Painting, Sculpture",
      "typeOfWork": "Cubism"
    }
    ```
*   **Response (201 Created):** Returns the created artist with their generated ID.

---

### Canvas Module (`/api/Canvas`)

This is the relational core of the system. A `Canvas` connects `Museum` (N:1) and `Artist` (N:M).

---
#### **Endpoint: `GET /api/Canvas`**
*   **Description:** Lists all available artworks.
*   **Optimization:** Uses DTOs to display artist names (`ArtistNames`) instead of complex `Work` objects, simplifying consumption for the frontend.
*   **Security:** Public (`AllowAnonymous`).
*   **Response (200 OK - `IEnumerable<CanvasResponseDto>`):**
    ```
    [
      {
        "id": "a1b2c3d4-...",
        "title": "Guernica",
        "technique": "Oil on canvas",
        "dateOfEntry": "1937-06-04T00:00:00Z",
        "museumId": "MUSEUM_GUID",
        "artistNames": [
          "Pablo Picasso"
        ]
      }
    ]
    ```

---
#### **Endpoint: `POST /api/Canvas`**
*   **Description:** Registers a new artwork. This endpoint is **transactional**: it creates the record in the `Canvas` table and, simultaneously, inserts the necessary records into the intermediate `Works` table to link artists.
*   **Security:** Protected (`[Authorize]`).
*   **Request Body (`CreateCanvasDto`):**
    ```
    {
      "title": "The Persistence of Memory",
      "technique": "Oil on canvas",
      "dateOfEntry": "1931-01-01T00:00:00Z",
      "museumId": "MOMA_GUID",
      "artistIds": [
        "DALI_ARTIST_GUID"
      ]
    }
    ```
*   **Response (201 Created):** Returns the created artwork.
*   **Possible Errors:**
    *   `404 Not Found`: If `museumId` or any of the `artistIds` do not exist in the database.

---

### Museum Module (`/api/Museum`)

Manages the main entities where artworks are housed.

---
#### **Endpoint: `GET /api/Museum`**
*   **Description:** Returns the catalog of museums.
*   **Projection:** Includes a nested list of artworks (`Canvas`) owned by each museum. To prevent cycles, nested `Canvas` objects do **NOT** include the `Museum` property back, but **DO** include the list of their artist names.
*   **Security:** Public (`AllowAnonymous`).
*   **Response (200 OK - `IEnumerable<MuseumResponseDto>`):**
    ```
    [
      {
        "id": "m1u2s3e4-...",
        "name": "Reina Sofía Museum",
        "description": "National museum of 20th-century art.",
        "openingYear": 1992,
        "cityId": "MADRID_GUID",
        "canvas": [
          {
            "id": "c1a2n3v4...",
            "title": "Guernica",
            "artistNames": ["Pablo Picasso"]
          }
        ]
      }
    ]
    ```

---
#### **Endpoint: `POST /api/Museum`**
*   **Description:** Registers a new museum in a specific city.
*   **Security:** Protected (`[Authorize]`).
*   **Request Body (`CreateMuseumDto`):**
    ```
    {
      "name": "Prado Museum",
      "description": "One of the most important in the world.",
      "openingYear": 1819,
      "cityId": "MADRID_GUID"
    }
    ```
*   **Response (201 Created):** Returns the created museum.

---

### City Module (`/api/City`)

Simple geographic entity grouping museums.

---
#### **Endpoint: `GET /api/City`**
*   **Description:** Lists registered cities.
*   **Security:** Public (`AllowAnonymous`).
*   **Response (200 OK):**
    ```
    [
      {
        "id": "c1i2t3y4...",
        "name": "Madrid",
        "country": "Spain",
        "museum": {
            "id": "...",
            "name": "Prado Museum"
        }
      }
    ]
    ```
*(Note: The nested `museum` object here does not expand its `canvas` list to keep the response lightweight).*

---
#### **Endpoint: `POST /api/City`**
*   **Description:** Adds a new city to the system.
*   **Security:** Protected (`[Authorize]`).
*   **Request Body:**
    ```
    {
      "name": "Paris",
      "country": "France"
    }
    ```
*   **Response (201 Created):** Returns the created city.

---

## ⚙️ Configuration and Environment Variables

The project uses `DotNetEnv` to load sensitive configurations from a `.env` file in the root, protecting credentials in version control.

| Variable | Description | Default Value (Dev) |
| :--- | :--- | :--- |
| `POSTGRES_DB` | Database Name | `museodb` |
| `POSTGRES_USER` | Database User | `museouser` |
| `POSTGRES_PASSWORD` | Database Password | `supersecret` |
| `JWT_KEY` | Secret key for Token signing | *(Must be robust)* |
| `JWT_REFRESHDAYS` | Refresh Token validity days | `14` |

---
© 2025 Museo API - Systems Engineering Project
