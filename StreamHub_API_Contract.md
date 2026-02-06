# StreamHub - API Contract Specification

**Version:** 1.0  
**Base URL:** `https://api.streamhub.com/v1`  
**Authentication:** Bearer Token (JWT)  
**Last Updated:** February 2026

---

## Table of Contents

1. [Authentication](#authentication)
2. [Common Schemas](#common-schemas)
3. [Error Responses](#error-responses)
4. [Auth Endpoints](#auth-endpoints)
5. [User Endpoints](#user-endpoints)
6. [Video Endpoints](#video-endpoints)
7. [Comment Endpoints](#comment-endpoints)
8. [Playlist Endpoints](#playlist-endpoints)
9. [Search Endpoints](#search-endpoints)
10. [Analytics Endpoints](#analytics-endpoints)

---

## Authentication

### Headers Required

```http
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

### JWT Token Structure

```json
{
  "sub": "user-uuid",
  "email": "user@example.com",
  "role": "creator",
  "iat": 1706691200,
  "exp": 1706777600
}
```

**Token Expiry:** 24 hours  
**Refresh:** Use `/auth/refresh` endpoint

---

## Common Schemas

### User Object

```typescript
interface User {
  id: string;              // UUID
  username: string;        // 3-50 chars, alphanumeric + underscore
  email: string;           // Valid email
  role: "viewer" | "creator" | "admin";
  avatar_url: string | null;
  bio: string | null;      // Max 500 chars
  created_at: string;      // ISO 8601
  total_views: number;
  total_uploads: number;
  follower_count: number;
  following_count: number;
}
```

### Video Object

```typescript
interface Video {
  id: string;              // UUID
  creator_id: string;      // UUID
  title: string;           // 3-200 chars
  description: string | null;  // Max 5000 chars
  thumbnail_url: string;
  video_url: string;       // HLS master playlist URL
  duration: number;        // Seconds
  file_size: number;       // Bytes
  status: "processing" | "ready" | "failed" | "deleted";
  visibility: "public" | "unlisted" | "private";
  tags: string[];          // Max 10 tags
  category: string | null;
  uploaded_at: string;     // ISO 8601
  processed_at: string | null;
  view_count: number;
  like_count: number;
  comment_count: number;
  creator: {
    id: string;
    username: string;
    avatar_url: string | null;
  };
}
```

### Comment Object

```typescript
interface Comment {
  id: string;              // UUID
  user_id: string;
  video_id: string;
  parent_comment_id: string | null;
  text: string;            // 1-1000 chars
  created_at: string;
  updated_at: string;
  is_edited: boolean;
  is_deleted: boolean;
  like_count: number;
  reply_count: number;
  user: {
    id: string;
    username: string;
    avatar_url: string | null;
  };
}
```

### Pagination Metadata

```typescript
interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  total_pages: number;
}
```

### Success Response Wrapper

```typescript
interface SuccessResponse<T> {
  success: true;
  data: T;
  meta?: PaginationMeta;
}
```

---

## Error Responses

### Error Schema

```typescript
interface ErrorResponse {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Array<{
      field: string;
      message: string;
    }>;
  };
}
```

### HTTP Status Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| 200 | OK | Successful GET, PUT, DELETE |
| 201 | Created | Successful POST |
| 400 | Bad Request | Validation error |
| 401 | Unauthorized | Missing or invalid token |
| 403 | Forbidden | Valid token but insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource (e.g., username taken) |
| 422 | Unprocessable Entity | Semantic validation error |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 503 | Service Unavailable | Service temporarily down |

### Common Error Codes

```typescript
enum ErrorCode {
  VALIDATION_ERROR = "VALIDATION_ERROR",
  UNAUTHORIZED = "UNAUTHORIZED",
  FORBIDDEN = "FORBIDDEN",
  NOT_FOUND = "NOT_FOUND",
  CONFLICT = "CONFLICT",
  RATE_LIMIT_EXCEEDED = "RATE_LIMIT_EXCEEDED",
  INTERNAL_ERROR = "INTERNAL_ERROR",
  SERVICE_UNAVAILABLE = "SERVICE_UNAVAILABLE"
}
```

### Example Error Responses

**Validation Error (400):**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "title",
        "message": "Title must be at least 3 characters"
      },
      {
        "field": "tags",
        "message": "Maximum 10 tags allowed"
      }
    ]
  }
}
```

**Not Found (404):**

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Video not found"
  }
}
```

**Rate Limit (429):**

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Try again in 60 seconds."
  }
}
```

---

## Auth Endpoints

### POST /auth/register

**Description:** Register a new user

**Authentication:** None

**Rate Limit:** 3 requests per hour per IP

**Request Body:**

```typescript
{
  username: string;    // 3-50 chars, alphanumeric + underscore
  email: string;       // Valid email
  password: string;    // Min 8 chars, 1 uppercase, 1 number
  role: "viewer" | "creator";
}
```

**Validation Rules:**
- `username`: Required, 3-50 chars, must be unique, alphanumeric + underscore only
- `email`: Required, valid email format, must be unique
- `password`: Required, min 8 chars, must contain 1 uppercase, 1 lowercase, 1 number
- `role`: Required, must be "viewer" or "creator"

**Success Response (201):**

```json
{
  "success": true,
  "data": {
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "username": "johndoe",
      "email": "john@example.com",
      "role": "creator",
      "avatar_url": null,
      "bio": null,
      "created_at": "2026-02-06T10:30:00Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Error Responses:**
- `409 Conflict` - Username or email already exists
- `400 Bad Request` - Validation error

---

### POST /auth/login

**Description:** Login with email and password

**Authentication:** None

**Rate Limit:** 5 requests per minute per IP

**Request Body:**

```typescript
{
  email: string;
  password: string;
}
```

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "username": "johndoe",
      "email": "john@example.com",
      "role": "creator",
      "avatar_url": "https://cdn.streamhub.com/avatars/johndoe.jpg",
      "bio": "Content creator",
      "created_at": "2026-02-06T10:30:00Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Error Responses:**
- `401 Unauthorized` - Invalid email or password
- `429 Too Many Requests` - Too many login attempts

---

### POST /auth/refresh

**Description:** Refresh access token

**Authentication:** Refresh token in body

**Request Body:**

```typescript
{
  refresh_token: string;
}
```

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

---

### POST /auth/logout

**Description:** Logout and invalidate token

**Authentication:** Required

**Request Body:** None

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "message": "Logged out successfully"
  }
}
```

---

### POST /auth/forgot-password

**Description:** Request password reset email

**Authentication:** None

**Rate Limit:** 3 requests per hour per IP

**Request Body:**

```typescript
{
  email: string;
}
```

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "message": "Password reset email sent if account exists"
  }
}
```

**Note:** Always returns success to prevent email enumeration

---

### POST /auth/reset-password

**Description:** Reset password with token

**Authentication:** None

**Request Body:**

```typescript
{
  token: string;       // From email link
  password: string;    // New password
}
```

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "message": "Password reset successfully"
  }
}
```

**Error Responses:**
- `400 Bad Request` - Invalid or expired token

---

## User Endpoints

### GET /users/:id

**Description:** Get user profile

**Authentication:** Optional (returns public info if not authenticated)

**Path Parameters:**
- `id`: User UUID

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "username": "johndoe",
    "avatar_url": "https://cdn.streamhub.com/avatars/johndoe.jpg",
    "bio": "Content creator focusing on tech tutorials",
    "created_at": "2026-01-01T00:00:00Z",
    "total_views": 125000,
    "total_uploads": 45,
    "follower_count": 5000,
    "is_subscribed": false
  }
}
```

**Error Responses:**
- `404 Not Found` - User doesn't exist

---

### PUT /users/:id

**Description:** Update user profile (own profile only)

**Authentication:** Required

**Path Parameters:**
- `id`: User UUID (must match authenticated user)

**Request Body:**

```typescript
{
  username?: string;      // 3-50 chars
  bio?: string;           // Max 500 chars
  avatar_url?: string;    // Valid URL
}
```

**Validation Rules:**
- All fields optional
- `username`: If provided, must be unique and 3-50 chars
- `bio`: Max 500 chars
- `avatar_url`: Must be valid URL

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "username": "johndoe",
    "email": "john@example.com",
    "avatar_url": "https://cdn.streamhub.com/avatars/johndoe.jpg",
    "bio": "Updated bio",
    "role": "creator",
    "created_at": "2026-01-01T00:00:00Z"
  }
}
```

**Error Responses:**
- `403 Forbidden` - Attempting to update another user's profile
- `409 Conflict` - Username already taken

---

### POST /users/:id/subscribe

**Description:** Subscribe to a creator

**Authentication:** Required

**Path Parameters:**
- `id`: Creator user UUID

**Success Response (201):**

```json
{
  "success": true,
  "data": {
    "id": "subscription-uuid",
    "subscriber_id": "current-user-uuid",
    "creator_id": "550e8400-e29b-41d4-a716-446655440000",
    "notifications_enabled": true,
    "subscribed_at": "2026-02-06T10:30:00Z"
  }
}
```

**Error Responses:**
- `400 Bad Request` - Cannot subscribe to yourself
- `409 Conflict` - Already subscribed

---

### DELETE /users/:id/subscribe

**Description:** Unsubscribe from a creator

**Authentication:** Required

**Path Parameters:**
- `id`: Creator user UUID

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "message": "Unsubscribed successfully"
  }
}
```

---

### GET /users/:id/videos

**Description:** Get user's uploaded videos

**Authentication:** Optional

**Path Parameters:**
- `id`: User UUID

**Query Parameters:**
- `page`: number (default: 1)
- `limit`: number (default: 20, max: 50)
- `sort`: "recent" | "views" | "likes" (default: "recent")

**Success Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "video-uuid",
      "title": "JavaScript Tutorial",
      "thumbnail_url": "https://cdn.streamhub.com/thumbnails/video.jpg",
      "duration": 600,
      "view_count": 12500,
      "like_count": 450,
      "uploaded_at": "2026-02-05T10:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "total_pages": 3
  }
}
```

---

## Video Endpoints

### POST /videos/upload-url

**Description:** Get presigned upload URL for direct R2 upload

**Authentication:** Required (creator role)

**Request Body:**

```typescript
{
  filename: string;      // Original filename
  file_size: number;     // File size in bytes
  content_type: string;  // MIME type (video/mp4, video/webm)
}
```

**Validation Rules:**
- `filename`: Required, max 255 chars
- `file_size`: Required, max 2GB (2147483648 bytes)
- `content_type`: Required, must be video/mp4 or video/webm

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "video_id": "550e8400-e29b-41d4-a716-446655440000",
    "upload_url": "https://r2.cloudflare.com/streamhub/uploads/550e8400.mp4?X-Amz-Signature=...",
    "expires_at": "2026-02-06T11:30:00Z"
  }
}
```

**Note:** Upload URL expires in 1 hour

---

### POST /videos

**Description:** Create video metadata after upload

**Authentication:** Required (creator role)

**Request Body:**

```typescript
{
  video_id: string;      // From upload-url response
  title: string;         // 3-200 chars
  description?: string;  // Max 5000 chars
  tags?: string[];       // Max 10 tags, each max 30 chars
  category?: string;     // One of predefined categories
  visibility: "public" | "unlisted" | "private";
}
```

**Validation Rules:**
- `video_id`: Required, must be valid UUID from upload-url
- `title`: Required, 3-200 chars
- `description`: Optional, max 5000 chars
- `tags`: Optional, max 10 tags, each max 30 chars
- `category`: Optional, must be one of: music, gaming, education, tech, entertainment, sports, lifestyle, cooking, travel, news
- `visibility`: Required

**Success Response (201):**

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "creator_id": "creator-uuid",
    "title": "My Video",
    "description": "Description here",
    "thumbnail_url": null,
    "video_url": null,
    "status": "processing",
    "visibility": "public",
    "tags": ["tutorial", "javascript"],
    "category": "education",
    "uploaded_at": "2026-02-06T10:30:00Z",
    "processed_at": null
  }
}
```

**Error Responses:**
- `400 Bad Request` - Invalid video_id or validation error
- `403 Forbidden` - Video doesn't belong to user

---

### GET /videos/:id

**Description:** Get video details

**Authentication:** Optional (required for private/unlisted videos)

**Path Parameters:**
- `id`: Video UUID

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "JavaScript Tutorial for Beginners",
    "description": "Learn JavaScript basics in this comprehensive tutorial",
    "thumbnail_url": "https://cdn.streamhub.com/thumbnails/550e8400.jpg",
    "video_url": "https://cdn.streamhub.com/videos/550e8400/master.m3u8",
    "duration": 600,
    "status": "ready",
    "visibility": "public",
    "tags": ["javascript", "tutorial", "beginner"],
    "category": "education",
    "uploaded_at": "2026-02-05T10:00:00Z",
    "processed_at": "2026-02-05T10:15:00Z",
    "view_count": 12500,
    "like_count": 450,
    "comment_count": 78,
    "creator": {
      "id": "creator-uuid",
      "username": "johndoe",
      "avatar_url": "https://cdn.streamhub.com/avatars/johndoe.jpg",
      "follower_count": 5000
    },
    "is_liked": false,
    "is_subscribed": false
  }
}
```

**Error Responses:**
- `404 Not Found` - Video doesn't exist
- `403 Forbidden` - Private video, not authorized

---

### PUT /videos/:id

**Description:** Update video metadata (creator only)

**Authentication:** Required

**Path Parameters:**
- `id`: Video UUID

**Request Body:**

```typescript
{
  title?: string;
  description?: string;
  tags?: string[];
  category?: string;
  visibility?: "public" | "unlisted" | "private";
}
```

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "Updated Title",
    "description": "Updated description",
    "tags": ["updated", "tags"],
    "category": "education",
    "visibility": "public"
  }
}
```

**Error Responses:**
- `403 Forbidden` - Not the video creator
- `404 Not Found` - Video doesn't exist

---

### DELETE /videos/:id

**Description:** Delete video (creator only)

**Authentication:** Required

**Path Parameters:**
- `id`: Video UUID

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "message": "Video deleted successfully"
  }
}
```

**Error Responses:**
- `403 Forbidden` - Not the video creator
- `404 Not Found` - Video doesn't exist

**Note:** Soft delete (status set to "deleted"), files remain for 30 days

---

### POST /videos/:id/view

**Description:** Increment view count

**Authentication:** Optional

**Path Parameters:**
- `id`: Video UUID

**Request Body:**

```typescript
{
  watch_duration?: number;  // Seconds watched (for analytics)
  device?: string;          // "mobile" | "desktop" | "tablet"
}
```

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "view_count": 12501
  }
}
```

**Note:** Rate limited to 1 view per user per video per hour

---

### POST /videos/:id/like

**Description:** Like a video

**Authentication:** Required

**Path Parameters:**
- `id`: Video UUID

**Success Response (201):**

```json
{
  "success": true,
  "data": {
    "like_count": 451,
    "is_liked": true
  }
}
```

**Error Responses:**
- `409 Conflict` - Already liked

---

### DELETE /videos/:id/like

**Description:** Unlike a video

**Authentication:** Required

**Path Parameters:**
- `id`: Video UUID

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "like_count": 450,
    "is_liked": false
  }
}
```

---

### GET /videos

**Description:** Get video feed (For You page)

**Authentication:** Optional (personalized if authenticated)

**Query Parameters:**
- `page`: number (default: 1)
- `limit`: number (default: 20, max: 50)
- `category?: string
- `sort?: "recent" | "views" | "trending"

**Success Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "video-uuid",
      "title": "Video Title",
      "thumbnail_url": "...",
      "duration": 600,
      "view_count": 12500,
      "like_count": 450,
      "uploaded_at": "2026-02-05T10:00:00Z",
      "creator": {
        "id": "creator-uuid",
        "username": "johndoe",
        "avatar_url": "..."
      }
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 1000,
    "total_pages": 50
  }
}
```

---

## Comment Endpoints

### POST /comments

**Description:** Create a comment on a video

**Authentication:** Required

**Rate Limit:** 20 requests per minute per user

**Request Body:**

```typescript
{
  video_id: string;              // UUID
  text: string;                  // 1-1000 chars
  parent_comment_id?: string;    // UUID (for replies)
}
```

**Validation Rules:**
- `video_id`: Required, must be valid UUID
- `text`: Required, 1-1000 chars
- `parent_comment_id`: Optional, must be valid UUID if provided

**Success Response (201):**

```json
{
  "success": true,
  "data": {
    "id": "comment-uuid",
    "user_id": "user-uuid",
    "video_id": "video-uuid",
    "parent_comment_id": null,
    "text": "Great video!",
    "created_at": "2026-02-06T10:30:00Z",
    "updated_at": "2026-02-06T10:30:00Z",
    "is_edited": false,
    "is_deleted": false,
    "like_count": 0,
    "reply_count": 0,
    "user": {
      "id": "user-uuid",
      "username": "johndoe",
      "avatar_url": "..."
    }
  }
}
```

**Error Responses:**
- `404 Not Found` - Video doesn't exist
- `429 Too Many Requests` - Rate limit exceeded

---

### GET /videos/:videoId/comments

**Description:** Get comments for a video

**Authentication:** Optional

**Path Parameters:**
- `videoId`: Video UUID

**Query Parameters:**
- `page`: number (default: 1)
- `limit`: number (default: 20, max: 50)
- `sort`: "recent" | "top" (default: "recent")

**Success Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "comment-uuid",
      "user_id": "user-uuid",
      "video_id": "video-uuid",
      "parent_comment_id": null,
      "text": "Great video!",
      "created_at": "2026-02-06T10:30:00Z",
      "updated_at": "2026-02-06T10:30:00Z",
      "is_edited": false,
      "like_count": 12,
      "reply_count": 3,
      "user": {
        "id": "user-uuid",
        "username": "johndoe",
        "avatar_url": "..."
      }
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 78,
    "total_pages": 4
  }
}
```

---

### PUT /comments/:id

**Description:** Edit a comment (author only)

**Authentication:** Required

**Path Parameters:**
- `id`: Comment UUID

**Request Body:**

```typescript
{
  text: string;  // 1-1000 chars
}
```

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "comment-uuid",
    "text": "Updated comment text",
    "updated_at": "2026-02-06T11:00:00Z",
    "is_edited": true
  }
}
```

**Error Responses:**
- `403 Forbidden` - Not comment author
- `404 Not Found` - Comment doesn't exist

---

### DELETE /comments/:id

**Description:** Delete a comment (author or video creator)

**Authentication:** Required

**Path Parameters:**
- `id`: Comment UUID

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "message": "Comment deleted successfully"
  }
}
```

---

### POST /comments/:id/like

**Description:** Like a comment

**Authentication:** Required

**Path Parameters:**
- `id`: Comment UUID

**Success Response (201):**

```json
{
  "success": true,
  "data": {
    "like_count": 13,
    "is_liked": true
  }
}
```

---

## Playlist Endpoints

### POST /playlists

**Description:** Create a playlist

**Authentication:** Required

**Request Body:**

```typescript
{
  name: string;          // 1-100 chars
  description?: string;  // Max 500 chars
  visibility: "public" | "unlisted" | "private";
}
```

**Success Response (201):**

```json
{
  "success": true,
  "data": {
    "id": "playlist-uuid",
    "user_id": "user-uuid",
    "name": "My Playlist",
    "description": "Description here",
    "visibility": "public",
    "thumbnail_url": null,
    "video_count": 0,
    "created_at": "2026-02-06T10:30:00Z"
  }
}
```

---

### GET /playlists/:id

**Description:** Get playlist details

**Authentication:** Optional

**Path Parameters:**
- `id`: Playlist UUID

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "id": "playlist-uuid",
    "name": "My Playlist",
    "description": "Description here",
    "visibility": "public",
    "thumbnail_url": "...",
    "video_count": 5,
    "created_at": "2026-02-06T10:30:00Z",
    "user": {
      "id": "user-uuid",
      "username": "johndoe",
      "avatar_url": "..."
    },
    "videos": [
      {
        "id": "video-uuid",
        "title": "Video Title",
        "thumbnail_url": "...",
        "duration": 600,
        "position": 1
      }
    ]
  }
}
```

---

### POST /playlists/:id/videos

**Description:** Add video to playlist

**Authentication:** Required

**Path Parameters:**
- `id`: Playlist UUID

**Request Body:**

```typescript
{
  video_id: string;  // UUID
}
```

**Success Response (201):**

```json
{
  "success": true,
  "data": {
    "playlist_id": "playlist-uuid",
    "video_id": "video-uuid",
    "position": 6,
    "added_at": "2026-02-06T10:30:00Z"
  }
}
```

---

### DELETE /playlists/:playlistId/videos/:videoId

**Description:** Remove video from playlist

**Authentication:** Required

**Path Parameters:**
- `playlistId`: Playlist UUID
- `videoId`: Video UUID

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "message": "Video removed from playlist"
  }
}
```

---

## Search Endpoints

### GET /search

**Description:** Search videos

**Authentication:** Optional

**Query Parameters:**
- `q`: string (search query, required)
- `category?: string
- `duration?: "short" | "medium" | "long"
- `upload_date?: "today" | "week" | "month" | "year"
- `sort?: "relevance" | "views" | "recent"
- `page`: number (default: 1)
- `limit`: number (default: 20, max: 50)

**Success Response (200):**

```json
{
  "success": true,
  "data": [
    {
      "id": "video-uuid",
      "title": "JavaScript Tutorial",
      "thumbnail_url": "...",
      "duration": 600,
      "view_count": 12500,
      "uploaded_at": "2026-02-05T10:00:00Z",
      "creator": {
        "id": "creator-uuid",
        "username": "johndoe",
        "avatar_url": "..."
      }
    }
  ],
  "meta": {
    "query": "javascript tutorial",
    "page": 1,
    "limit": 20,
    "total": 234,
    "total_pages": 12
  }
}
```

---

### GET /search/autocomplete

**Description:** Get search suggestions

**Authentication:** Optional

**Query Parameters:**
- `q`: string (partial query, required, min 2 chars)

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "suggestions": [
      "javascript tutorial",
      "javascript basics",
      "javascript for beginners"
    ]
  }
}
```

---

## Analytics Endpoints

### POST /analytics/view

**Description:** Track video view event (for analytics)

**Authentication:** Optional

**Request Body:**

```typescript
{
  video_id: string;
  watch_duration: number;    // Seconds watched
  device?: string;           // "mobile" | "desktop" | "tablet"
  referrer?: string;         // Where they came from
}
```

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "message": "Event tracked"
  }
}
```

---

### GET /analytics/video/:id

**Description:** Get video analytics (creator only)

**Authentication:** Required

**Path Parameters:**
- `id`: Video UUID

**Query Parameters:**
- `period`: "7d" | "30d" | "90d" | "all" (default: "30d")

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "video_id": "video-uuid",
    "period": "30d",
    "total_views": 12500,
    "unique_viewers": 8900,
    "total_watch_time": 2500000,
    "avg_watch_duration": 200,
    "likes": 450,
    "comments": 78,
    "shares": 23,
    "views_by_day": [
      { "date": "2026-02-01", "views": 450 },
      { "date": "2026-02-02", "views": 520 }
    ],
    "demographics": {
      "devices": { "mobile": 6000, "desktop": 5000, "tablet": 1500 },
      "countries": { "US": 5000, "UK": 2000, "CA": 1500 }
    },
    "traffic_sources": {
      "search": 5000,
      "direct": 3000,
      "recommendation": 2500,
      "external": 2000
    }
  }
}
```

**Error Responses:**
- `403 Forbidden` - Not the video creator

---

### GET /analytics/creator/:id

**Description:** Get creator dashboard analytics

**Authentication:** Required

**Path Parameters:**
- `id`: Creator user UUID (must match authenticated user)

**Query Parameters:**
- `period`: "7d" | "30d" | "90d" | "all" (default: "30d")

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "period": "30d",
    "total_views": 125000,
    "total_watch_time": 25000000,
    "total_likes": 4500,
    "total_comments": 890,
    "subscriber_count": 5000,
    "video_count": 45,
    "views_by_day": [
      { "date": "2026-02-01", "views": 4500 }
    ],
    "top_videos": [
      {
        "id": "video-uuid",
        "title": "Top Video",
        "views": 45000,
        "watch_time": 9000000
      }
    ]
  }
}
```

---

## Rate Limiting

### Rate Limit Headers

Every response includes rate limit headers:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1706777600
```

### Rate Limits by Endpoint Type

| Endpoint Type | Limit | Window |
|---------------|-------|--------|
| Read (GET) | 1000 req | 1 hour |
| Write (POST/PUT) | 100 req | 1 hour |
| Auth (login) | 5 req | 1 minute |
| Upload | 10 uploads | 1 day |
| Comment | 20 req | 1 minute |

---

## WebSocket Events (Phase 2)

### Connection

```
wss://api.streamhub.com/v1/ws?token=<jwt_token>
```

### Events

**New Comment:**

```json
{
  "event": "comment:new",
  "data": {
    "video_id": "video-uuid",
    "comment": { /* Comment object */ }
  }
}
```

**Live Viewer Count:**

```json
{
  "event": "viewers:update",
  "data": {
    "video_id": "video-uuid",
    "viewer_count": 145
  }
}
```

---

**End of API Contract Specification**

*This document is the single source of truth for all API endpoints. Use it to generate backend code, frontend API clients, and API documentation.*
