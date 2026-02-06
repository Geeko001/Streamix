# StreamHub - Technical Design & Architecture Document

**Version:** 1.0  
**Last Updated:** February 2026  
**Status:** Draft

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Patterns](#architecture-patterns)
3. [Service Breakdown](#service-breakdown)
4. [Database Design](#database-design)
5. [API Specifications](#api-specifications)
6. [Data Flow](#data-flow)
7. [Infrastructure](#infrastructure)
8. [Security Architecture](#security-architecture)
9. [Performance & Scalability](#performance--scalability)
10. [Monitoring & Observability](#monitoring--observability)

---

## 1. System Overview

### 1.1 High-Level Architecture

```
                                    ┌─────────────────┐
                                    │   Cloudflare    │
                                    │   DNS + WAF     │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
            ┌───────▼────────┐      ┌───────▼────────┐      ┌───────▼────────┐
            │   Web Client   │      │  Mobile Client │      │  Admin Panel   │
            │   (Next.js)    │      │ (React Native) │      │   (React)      │
            └───────┬────────┘      └───────┬────────┘      └───────┬────────┘
                    │                        │                        │
                    └────────────────────────┼────────────────────────┘
                                             │
                                    ┌────────▼────────┐
                                    │   API Gateway   │
                                    │   (Kong/NGINX)  │
                                    │  Rate Limiting  │
                                    │  Load Balancer  │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
            ┌───────▼────────┐      ┌───────▼────────┐      ┌───────▼────────┐
            │  Auth Service  │      │ Video Service  │      │ User Service   │
            │   (Node.js)    │      │   (Node.js)    │      │   (Node.js)    │
            └───────┬────────┘      └───────┬────────┘      └───────┬────────┘
                    │                        │                        │
                    └────────────────────────┼────────────────────────┘
                                             │
                                    ┌────────▼────────┐
                                    │  Message Queue  │
                                    │   (RabbitMQ)    │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
            ┌───────▼────────┐      ┌───────▼────────┐      ┌───────▼────────┐
            │    Video       │      │      AI        │      │   Analytics    │
            │   Processor    │      │  Moderation    │      │    Worker      │
            │   (FFmpeg)     │      │   (Python)     │      │   (Python)     │
            └───────┬────────┘      └───────┬────────┘      └───────┬────────┘
                    │                        │                        │
                    └────────────────────────┼────────────────────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
            ┌───────▼────────┐      ┌───────▼────────┐      ┌───────▼────────┐
            │   PostgreSQL   │      │     Redis      │      │ Elasticsearch  │
            │   (Primary)    │      │    (Cache)     │      │    (Search)    │
            └────────────────┘      └────────────────┘      └────────────────┘
                                             │
                                    ┌────────▼────────┐
                                    │  Cloudflare R2  │
                                    │ (Video Storage) │
                                    └─────────────────┘
```

### 1.2 Design Principles

1. **Microservices Architecture:** Loose coupling, independent deployment
2. **Event-Driven:** Async processing via message queues
3. **API-First:** RESTful APIs with versioning
4. **Stateless Services:** Horizontal scaling capability
5. **Cache-First:** Redis for hot data, reduce DB load
6. **CDN-Everywhere:** Cloudflare for global delivery
7. **Security by Default:** Authentication on all endpoints
8. **Observability:** Logging, metrics, tracing built-in

---

## 2. Architecture Patterns

### 2.1 Microservices Design

#### Service Communication Patterns

```
┌──────────────┐
│   Client     │
└──────┬───────┘
       │ HTTP/REST
       ▼
┌──────────────┐
│ API Gateway  │
└──────┬───────┘
       │
       ├─────► Auth Service (Synchronous HTTP)
       │
       ├─────► Video Service (Synchronous HTTP)
       │             │
       │             └─────► Message Queue (Async)
       │                           │
       │                           ├─────► Video Processor
       │                           ├─────► AI Moderation
       │                           └─────► Analytics Worker
       │
       └─────► User Service (Synchronous HTTP)
```

**Pattern Breakdown:**

- **Synchronous (HTTP/REST):** Client ↔ API Gateway ↔ Services
  - Use for: Read operations, queries, immediate responses
  - Timeout: 30 seconds max
  
- **Asynchronous (Message Queue):** Services ↔ Workers
  - Use for: Heavy processing, uploads, analytics
  - Retry policy: Exponential backoff (3 attempts)

### 2.2 Database Pattern: CQRS Lite

```
Write Path:
┌─────────┐      ┌──────────────┐      ┌──────────────┐
│ Client  │─────►│ Write Service│─────►│  PostgreSQL  │
└─────────┘      └──────┬───────┘      │   (Master)   │
                        │              └──────────────┘
                        │
                        ├─────► Redis (Invalidate Cache)
                        │
                        └─────► Event Queue ──► Analytics Worker


Read Path:
┌─────────┐      ┌──────────────┐      ┌──────────────┐
│ Client  │─────►│ Read Service │─────►│     Redis    │
└─────────┘      └──────┬───────┘      │    (Cache)   │
                        │              └──────────────┘
                        │                     │ Cache Miss
                        │                     ▼
                        └──────────────► PostgreSQL (Read Replica)
```

**Benefits:**
- Separate read/write optimization
- Cache invalidation on writes
- Read replicas for scalability

### 2.3 Caching Strategy

```
Request Flow with Caching:

1. Client Request
   └─► API Gateway
       └─► Check Redis Cache (TTL: varies by endpoint)
           │
           ├─► Cache HIT: Return data (99% of reads)
           │
           └─► Cache MISS:
               └─► Query PostgreSQL
                   └─► Store in Redis (with TTL)
                       └─► Return data to client
```

**Cache Layers:**

| Data Type | TTL | Storage |
|-----------|-----|---------|
| User session | 24 hours | Redis |
| Video metadata | 5 minutes | Redis |
| Trending videos | 10 minutes | Redis |
| Search results | 2 minutes | Redis |
| User profile | 30 minutes | Redis |
| Static assets | 1 year | Cloudflare CDN |
| Video files | Forever | Cloudflare R2 + CDN |

---

## 3. Service Breakdown

### 3.1 Auth Service

**Responsibilities:**
- User registration & login
- JWT token generation & validation
- OAuth integration (Google, GitHub)
- Password reset flow
- Session management

**Technology Stack:**
- Runtime: Node.js (TypeScript)
- Framework: Express.js
- Auth Library: Passport.js + JWT
- Password Hashing: bcrypt (cost factor: 12)

**API Endpoints:**

```
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
GET    /api/v1/auth/oauth/:provider
GET    /api/v1/auth/oauth/:provider/callback
```

**Database Tables:**
- `users`
- `sessions`
- `oauth_connections`
- `password_reset_tokens`

**Performance:**
- Login: <200ms (p95)
- Token validation: <50ms (cached in Redis)
- Rate limit: 5 login attempts per minute per IP

---

### 3.2 Video Service

**Responsibilities:**
- Video metadata CRUD
- Upload URL generation (presigned R2 URLs)
- Video processing orchestration
- Streaming URL generation
- Engagement tracking (views, likes)

**Technology Stack:**
- Runtime: Node.js (TypeScript)
- Framework: Fastify (faster than Express)
- File Upload: Presigned URLs (no server upload)
- Storage: Cloudflare R2

**API Endpoints:**

```
POST   /api/v1/videos/upload-url          # Get presigned upload URL
POST   /api/v1/videos                     # Create video metadata
GET    /api/v1/videos/:id                 # Get video details
PUT    /api/v1/videos/:id                 # Update video
DELETE /api/v1/videos/:id                 # Delete video
GET    /api/v1/videos/:id/stream-url      # Get HLS streaming URL
POST   /api/v1/videos/:id/view            # Increment view count
POST   /api/v1/videos/:id/like            # Like video
DELETE /api/v1/videos/:id/like            # Unlike video
GET    /api/v1/videos/trending            # Get trending videos
GET    /api/v1/videos/search              # Search videos
```

**Upload Flow:**

```
1. Client requests upload URL
   POST /api/v1/videos/upload-url
   ├─► Video Service generates presigned R2 URL (valid for 1 hour)
   └─► Returns: { uploadUrl, videoId }

2. Client uploads directly to R2
   PUT <uploadUrl> with video file
   ├─► Cloudflare R2 receives file
   └─► Returns 200 OK

3. Client confirms upload
   POST /api/v1/videos with metadata
   ├─► Video Service creates DB record (status: 'processing')
   ├─► Publishes to video-processing queue
   └─► Returns video object

4. Background worker processes video
   └─► Updates status to 'ready' or 'failed'
```

**Database Tables:**
- `videos`
- `video_streams` (HLS variants)
- `likes`
- `views` (aggregated)
- `watch_history`

---

### 3.3 User Service

**Responsibilities:**
- User profile management
- Follow/subscription system
- Playlist CRUD
- Creator dashboard data aggregation

**API Endpoints:**

```
GET    /api/v1/users/:id                  # Get user profile
PUT    /api/v1/users/:id                  # Update profile
GET    /api/v1/users/:id/videos           # Get user's videos
POST   /api/v1/users/:id/follow           # Follow user
DELETE /api/v1/users/:id/follow           # Unfollow user
GET    /api/v1/users/:id/followers        # Get followers
GET    /api/v1/users/:id/following        # Get following

POST   /api/v1/playlists                  # Create playlist
GET    /api/v1/playlists/:id              # Get playlist
PUT    /api/v1/playlists/:id              # Update playlist
DELETE /api/v1/playlists/:id              # Delete playlist
POST   /api/v1/playlists/:id/videos       # Add video to playlist
```

**Database Tables:**
- `users` (shared with Auth)
- `subscriptions`
- `playlists`
- `playlist_videos`

---

### 3.4 Comment Service

**Responsibilities:**
- Comment CRUD
- Threaded replies
- Comment moderation
- Spam detection

**API Endpoints:**

```
POST   /api/v1/comments                   # Create comment
GET    /api/v1/comments/:id               # Get comment
PUT    /api/v1/comments/:id               # Edit comment
DELETE /api/v1/comments/:id               # Delete comment
GET    /api/v1/videos/:videoId/comments   # Get video comments
POST   /api/v1/comments/:id/reply         # Reply to comment
POST   /api/v1/comments/:id/like          # Like comment
POST   /api/v1/comments/:id/report        # Report comment
```

**Database Tables:**
- `comments`
- `comment_likes`
- `comment_reports`

**Spam Detection:**
- Rate limit: 5 comments per minute per user
- AI moderation: Check for toxicity (Perspective API)
- Auto-hide if toxicity score > 0.8

---

### 3.5 Search Service

**Responsibilities:**
- Full-text search across videos
- Filter & sort operations
- Auto-complete suggestions
- Search analytics

**Technology Stack:**
- Search Engine: Elasticsearch 8+
- Alternative: Meilisearch (simpler, cheaper)

**API Endpoints:**

```
GET    /api/v1/search                     # Search videos
GET    /api/v1/search/autocomplete        # Auto-complete
GET    /api/v1/search/suggestions         # Suggested searches
```

**Elasticsearch Index Schema:**

```json
{
  "videos": {
    "mappings": {
      "properties": {
        "id": { "type": "keyword" },
        "title": { 
          "type": "text",
          "analyzer": "standard",
          "fields": {
            "keyword": { "type": "keyword" }
          }
        },
        "description": { "type": "text" },
        "tags": { "type": "keyword" },
        "creator_id": { "type": "keyword" },
        "creator_name": { "type": "text" },
        "views": { "type": "integer" },
        "likes": { "type": "integer" },
        "upload_date": { "type": "date" },
        "duration": { "type": "integer" },
        "category": { "type": "keyword" }
      }
    }
  }
}
```

**Search Query Example:**

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "cooking tutorial",
            "fields": ["title^3", "description", "tags^2"]
          }
        }
      ],
      "filter": [
        { "range": { "upload_date": { "gte": "now-30d" } } },
        { "term": { "category": "cooking" } }
      ]
    }
  },
  "sort": [
    { "views": "desc" }
  ]
}
```

---

### 3.6 Analytics Service

**Responsibilities:**
- View tracking & aggregation
- Creator dashboard metrics
- A/B testing data collection
- Real-time analytics

**Technology Stack:**
- Runtime: Python (FastAPI)
- Time-series DB: TimescaleDB (PostgreSQL extension)
- Real-time: Apache Kafka (future) or Redis Streams

**API Endpoints:**

```
POST   /api/v1/analytics/view             # Track view event
POST   /api/v1/analytics/engagement       # Track engagement
GET    /api/v1/analytics/video/:id        # Get video analytics
GET    /api/v1/analytics/creator/:id      # Get creator analytics
GET    /api/v1/analytics/trending         # Get trending data
```

**Event Schema:**

```json
{
  "event_type": "video_view",
  "timestamp": "2026-02-06T10:30:00Z",
  "user_id": "uuid",
  "video_id": "uuid",
  "session_id": "uuid",
  "watch_duration": 120,
  "device": "mobile",
  "country": "US",
  "referrer": "search"
}
```

**Analytics Aggregation:**

```sql
-- Daily video stats (TimescaleDB continuous aggregate)
CREATE MATERIALIZED VIEW video_stats_daily
WITH (timescaledb.continuous) AS
SELECT 
  time_bucket('1 day', timestamp) AS day,
  video_id,
  COUNT(*) AS views,
  COUNT(DISTINCT user_id) AS unique_viewers,
  AVG(watch_duration) AS avg_watch_duration,
  SUM(watch_duration) AS total_watch_time
FROM video_views
GROUP BY day, video_id;
```

---

### 3.7 Recommendation Service

**Responsibilities:**
- Personalized video recommendations
- Similar video suggestions
- Trending algorithm
- Cold-start handling (new users)

**Technology Stack:**
- Runtime: Python (FastAPI)
- ML Framework: PyTorch / TensorFlow
- Vector DB: Pinecone or Qdrant
- Model: Two-Tower Neural Network

**API Endpoints:**

```
GET    /api/v1/recommendations/for-you    # Personalized feed
GET    /api/v1/recommendations/similar/:videoId  # Similar videos
GET    /api/v1/recommendations/trending   # Trending (smart algorithm)
```

**Recommendation Algorithm (Hybrid):**

```python
# Pseudocode for recommendation engine

def get_recommendations(user_id, k=20):
    # 1. Content-based filtering
    user_history = get_user_watch_history(user_id)
    content_scores = compute_content_similarity(user_history)
    
    # 2. Collaborative filtering
    similar_users = find_similar_users(user_id)
    collab_scores = aggregate_similar_user_views(similar_users)
    
    # 3. Trending boost
    trending_videos = get_trending_videos()
    trending_scores = compute_trending_scores(trending_videos)
    
    # 4. Hybrid scoring (weighted combination)
    final_scores = (
        0.5 * content_scores +
        0.3 * collab_scores +
        0.2 * trending_scores
    )
    
    # 5. Diversity & freshness
    recommendations = diversify_results(final_scores, k)
    
    return recommendations
```

**Vector Embeddings:**

```python
# Video embedding generation
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

def generate_video_embedding(video):
    text = f"{video.title} {video.description} {' '.join(video.tags)}"
    embedding = model.encode(text)
    return embedding  # 384-dimensional vector

# Store in vector database (Pinecone)
index.upsert([
    (video.id, embedding, {"title": video.title, "views": video.views})
])

# Query similar videos
results = index.query(query_embedding, top_k=10, include_metadata=True)
```

---

### 3.8 Moderation Service (AI)

**Responsibilities:**
- NSFW content detection
- Violence/hate speech detection
- Copyright audio fingerprinting
- Auto-flag suspicious content
- Admin review queue

**Technology Stack:**
- Runtime: Python (FastAPI)
- ML Models:
  - NSFW: CLIP or ResNet50
  - Violence: Custom CNN
  - Text: Perspective API
  - Audio: AcoustID

**API Endpoints:**

```
POST   /api/v1/moderation/check-video     # Analyze video
POST   /api/v1/moderation/check-text      # Analyze comment/title
POST   /api/v1/moderation/check-audio     # Check copyright
GET    /api/v1/moderation/queue           # Admin review queue
POST   /api/v1/moderation/review/:id      # Submit review decision
```

**NSFW Detection Pipeline:**

```python
import torch
from PIL import Image
from transformers import CLIPProcessor, CLIPModel

model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

def check_nsfw(image_path):
    image = Image.open(image_path)
    
    # Define labels
    labels = ["safe content", "explicit content", "suggestive content"]
    
    inputs = processor(
        text=labels, 
        images=image, 
        return_tensors="pt", 
        padding=True
    )
    
    outputs = model(**inputs)
    logits_per_image = outputs.logits_per_image
    probs = logits_per_image.softmax(dim=1)
    
    # Get scores
    nsfw_score = probs[0][1].item() + probs[0][2].item()
    
    return {
        "nsfw_score": nsfw_score,
        "flagged": nsfw_score > 0.7,
        "action": "block" if nsfw_score > 0.9 else "review" if nsfw_score > 0.7 else "allow"
    }
```

---

### 3.9 Video Processing Worker

**Responsibilities:**
- Video transcoding to HLS
- Thumbnail generation
- Multiple quality variants (360p, 480p, 720p, 1080p)
- Metadata extraction (duration, resolution, codec)

**Technology Stack:**
- Container: Docker
- Tool: FFmpeg
- Queue: BullMQ (Redis-based)
- Storage: Cloudflare R2

**Processing Pipeline:**

```bash
# 1. Download video from R2
aws s3 cp s3://bucket/uploads/video.mp4 /tmp/input.mp4

# 2. Generate HLS variants
ffmpeg -i /tmp/input.mp4 \
  -vf scale=w=640:h=360:force_original_aspect_ratio=decrease \
  -c:a aac -ar 48000 -b:a 128k \
  -c:v h264 -profile:v main -crf 20 -g 48 -keyint_min 48 \
  -sc_threshold 0 -b:v 800k -maxrate 856k -bufsize 1200k \
  -hls_time 4 -hls_playlist_type vod -hls_segment_filename /tmp/360p_%03d.ts \
  /tmp/360p.m3u8

# Repeat for 480p, 720p, 1080p

# 3. Generate master playlist
cat > /tmp/master.m3u8 <<EOF
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=842x480
480p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720
720p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p.m3u8
EOF

# 4. Generate thumbnail
ffmpeg -i /tmp/input.mp4 -ss 00:00:05 -vframes 1 /tmp/thumbnail.jpg

# 5. Upload to R2
aws s3 sync /tmp/ s3://bucket/videos/{video_id}/

# 6. Update database
UPDATE videos SET status = 'ready', processed_at = NOW() WHERE id = {video_id};
```

**Worker Code (Node.js + BullMQ):**

```typescript
import { Worker } from 'bullmq';
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

const worker = new Worker('video-processing', async (job) => {
  const { videoId, inputUrl } = job.data;
  
  try {
    // Update status
    await db.query('UPDATE videos SET status = $1 WHERE id = $2', ['processing', videoId]);
    
    // Download
    await execAsync(`aws s3 cp ${inputUrl} /tmp/${videoId}.mp4`);
    
    // Process
    await generateHLS(videoId);
    await generateThumbnail(videoId);
    
    // Upload
    await execAsync(`aws s3 sync /tmp/${videoId}/ s3://bucket/videos/${videoId}/`);
    
    // Update status
    await db.query('UPDATE videos SET status = $1, processed_at = NOW() WHERE id = $2', 
                   ['ready', videoId]);
    
    // Clean up
    await execAsync(`rm -rf /tmp/${videoId}*`);
    
    return { success: true };
  } catch (error) {
    await db.query('UPDATE videos SET status = $1 WHERE id = $2', ['failed', videoId]);
    throw error;
  }
}, {
  connection: redisConnection,
  concurrency: 2, // Process 2 videos at a time
});
```

---

## 4. Database Design

### 4.1 PostgreSQL Schema

#### Users Table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(20) DEFAULT 'viewer' CHECK (role IN ('viewer', 'creator', 'admin')),
  avatar_url TEXT,
  bio TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Metadata (denormalized for performance)
  total_views BIGINT DEFAULT 0,
  total_uploads INTEGER DEFAULT 0,
  follower_count INTEGER DEFAULT 0,
  following_count INTEGER DEFAULT 0
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);
```

#### Videos Table

```sql
CREATE TABLE videos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(200) NOT NULL,
  description TEXT,
  thumbnail_url TEXT,
  video_url TEXT, -- R2 base path
  duration INTEGER, -- seconds
  file_size BIGINT, -- bytes
  format VARCHAR(20),
  status VARCHAR(20) DEFAULT 'processing' CHECK (status IN ('processing', 'ready', 'failed', 'deleted')),
  visibility VARCHAR(20) DEFAULT 'public' CHECK (visibility IN ('public', 'unlisted', 'private')),
  tags TEXT[], -- PostgreSQL array
  category VARCHAR(50),
  uploaded_at TIMESTAMP DEFAULT NOW(),
  processed_at TIMESTAMP,
  
  -- Denormalized engagement metrics
  view_count BIGINT DEFAULT 0,
  like_count INTEGER DEFAULT 0,
  comment_count INTEGER DEFAULT 0,
  share_count INTEGER DEFAULT 0,
  
  -- Metadata
  resolution VARCHAR(20),
  codec VARCHAR(50),
  bitrate INTEGER
);

CREATE INDEX idx_videos_creator_id ON videos(creator_id);
CREATE INDEX idx_videos_status ON videos(status);
CREATE INDEX idx_videos_visibility ON videos(visibility);
CREATE INDEX idx_videos_uploaded_at ON videos(uploaded_at DESC);
CREATE INDEX idx_videos_category ON videos(category);
CREATE INDEX idx_videos_tags ON videos USING GIN(tags);
CREATE INDEX idx_videos_view_count ON videos(view_count DESC);
```

#### Video Streams Table (HLS Variants)

```sql
CREATE TABLE video_streams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
  quality VARCHAR(20) NOT NULL, -- '360p', '480p', '720p', '1080p'
  hls_url TEXT NOT NULL,
  file_size BIGINT,
  bitrate INTEGER,
  resolution VARCHAR(20),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_video_streams_video_id ON video_streams(video_id);
CREATE UNIQUE INDEX idx_video_streams_video_quality ON video_streams(video_id, quality);
```

#### Likes Table

```sql
CREATE TABLE likes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(user_id, video_id)
);

CREATE INDEX idx_likes_user_id ON likes(user_id);
CREATE INDEX idx_likes_video_id ON likes(video_id);
CREATE INDEX idx_likes_created_at ON likes(created_at DESC);
```

#### Comments Table

```sql
CREATE TABLE comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
  parent_comment_id UUID REFERENCES comments(id) ON DELETE CASCADE,
  text TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  is_edited BOOLEAN DEFAULT FALSE,
  is_deleted BOOLEAN DEFAULT FALSE,
  like_count INTEGER DEFAULT 0,
  reply_count INTEGER DEFAULT 0
);

CREATE INDEX idx_comments_video_id ON comments(video_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
CREATE INDEX idx_comments_parent_id ON comments(parent_comment_id);
CREATE INDEX idx_comments_created_at ON comments(created_at DESC);
```

#### Subscriptions Table

```sql
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subscriber_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  creator_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  notifications_enabled BOOLEAN DEFAULT TRUE,
  subscribed_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(subscriber_id, creator_id),
  CHECK (subscriber_id != creator_id)
);

CREATE INDEX idx_subscriptions_subscriber_id ON subscriptions(subscriber_id);
CREATE INDEX idx_subscriptions_creator_id ON subscriptions(creator_id);
```

#### Watch History Table (TimescaleDB)

```sql
CREATE TABLE watch_history (
  id UUID DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
  watched_at TIMESTAMP NOT NULL DEFAULT NOW(),
  watch_duration INTEGER, -- seconds actually watched
  last_position INTEGER, -- last playback position
  completed BOOLEAN DEFAULT FALSE,
  device VARCHAR(50),
  country VARCHAR(2),
  
  PRIMARY KEY (id, watched_at)
);

-- Convert to TimescaleDB hypertable (time-series optimization)
SELECT create_hypertable('watch_history', 'watched_at');

CREATE INDEX idx_watch_history_user_id ON watch_history(user_id, watched_at DESC);
CREATE INDEX idx_watch_history_video_id ON watch_history(video_id, watched_at DESC);
```

#### Playlists Table

```sql
CREATE TABLE playlists (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  visibility VARCHAR(20) DEFAULT 'public' CHECK (visibility IN ('public', 'unlisted', 'private')),
  thumbnail_url TEXT,
  video_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_playlists_user_id ON playlists(user_id);
```

#### Playlist Videos Table

```sql
CREATE TABLE playlist_videos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  playlist_id UUID NOT NULL REFERENCES playlists(id) ON DELETE CASCADE,
  video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
  position INTEGER NOT NULL,
  added_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(playlist_id, video_id)
);

CREATE INDEX idx_playlist_videos_playlist_id ON playlist_videos(playlist_id, position);
```

### 4.2 Database Triggers (Auto-update denormalized counts)

```sql
-- Trigger: Update video like_count when like is added/removed
CREATE OR REPLACE FUNCTION update_video_like_count()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE videos SET like_count = like_count + 1 WHERE id = NEW.video_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE videos SET like_count = like_count - 1 WHERE id = OLD.video_id;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_video_like_count
AFTER INSERT OR DELETE ON likes
FOR EACH ROW EXECUTE FUNCTION update_video_like_count();

-- Trigger: Update user follower_count
CREATE OR REPLACE FUNCTION update_follower_count()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE users SET follower_count = follower_count + 1 WHERE id = NEW.creator_id;
    UPDATE users SET following_count = following_count + 1 WHERE id = NEW.subscriber_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE users SET follower_count = follower_count - 1 WHERE id = OLD.creator_id;
    UPDATE users SET following_count = following_count - 1 WHERE id = OLD.subscriber_id;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_follower_count
AFTER INSERT OR DELETE ON subscriptions
FOR EACH ROW EXECUTE FUNCTION update_follower_count();
```

### 4.3 Redis Cache Structure

```
# Session storage
session:{token} → {userId, expiresAt, device}
TTL: 24 hours

# User profile cache
user:{userId} → JSON(user object)
TTL: 30 minutes

# Video metadata cache
video:{videoId} → JSON(video object)
TTL: 5 minutes

# Trending videos (sorted set)
trending:hour → ZSET {videoId: score}
trending:day → ZSET {videoId: score}
trending:week → ZSET {videoId: score}
TTL: 10 minutes (regenerated)

# Search results cache
search:{query_hash} → JSON(results)
TTL: 2 minutes

# Rate limiting
ratelimit:login:{ip} → count
TTL: 1 minute

ratelimit:upload:{userId} → count
TTL: 1 hour

# View count buffer (for high-frequency writes)
views:buffer:{videoId} → count
Flush to DB every 5 minutes
```

---

## 5. API Specifications

### 5.1 RESTful API Design Principles

1. **Versioning:** `/api/v1/...`
2. **Resource-based URLs:** `/videos`, `/users`, `/comments`
3. **HTTP Methods:** GET (read), POST (create), PUT (update), DELETE (delete)
4. **Status Codes:** 200 (OK), 201 (Created), 400 (Bad Request), 401 (Unauthorized), 404 (Not Found), 500 (Server Error)
5. **Pagination:** `?page=1&limit=20`
6. **Filtering:** `?category=music&duration=short`
7. **Sorting:** `?sort=-views` (descending)

### 5.2 Authentication

**JWT Token Structure:**

```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user-uuid",
    "role": "creator",
    "iat": 1706691200,
    "exp": 1706777600
  }
}
```

**Authentication Header:**

```
Authorization: Bearer <jwt_token>
```

### 5.3 API Response Format

**Success Response:**

```json
{
  "success": true,
  "data": {
    "id": "123",
    "title": "My Video"
  },
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

**Error Response:**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "title",
        "message": "Title is required"
      }
    ]
  }
}
```

### 5.4 Example API Endpoints

#### POST /api/v1/videos (Upload Video)

**Request:**

```http
POST /api/v1/videos HTTP/1.1
Host: api.streamhub.com
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "My Awesome Video",
  "description": "This is a great video about coding",
  "tags": ["coding", "tutorial", "javascript"],
  "category": "education",
  "visibility": "public"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "uploadUrl": "https://r2.cloudflare.com/presigned-url...",
    "expiresAt": "2026-02-06T12:00:00Z"
  }
}
```

#### GET /api/v1/videos/:id (Get Video)

**Request:**

```http
GET /api/v1/videos/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
Host: api.streamhub.com
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "My Awesome Video",
    "description": "This is a great video about coding",
    "thumbnail": "https://cdn.streamhub.com/thumbnails/550e8400.jpg",
    "duration": 600,
    "views": 1523,
    "likes": 45,
    "creator": {
      "id": "creator-uuid",
      "username": "johndoe",
      "avatar": "https://cdn.streamhub.com/avatars/johndoe.jpg"
    },
    "uploadedAt": "2026-02-05T10:30:00Z",
    "streamUrl": "https://cdn.streamhub.com/videos/550e8400/master.m3u8"
  }
}
```

#### GET /api/v1/search (Search Videos)

**Request:**

```http
GET /api/v1/search?q=javascript tutorial&category=education&sort=-views&page=1&limit=20 HTTP/1.1
Host: api.streamhub.com
```

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "id": "video-1",
      "title": "JavaScript Tutorial for Beginners",
      "thumbnail": "...",
      "views": 50000,
      "creator": {...}
    }
  ],
  "meta": {
    "query": "javascript tutorial",
    "page": 1,
    "limit": 20,
    "total": 234,
    "totalPages": 12
  }
}
```

---

## 6. Data Flow

### 6.1 Video Upload Flow

```
┌─────────┐                                           ┌──────────────┐
│ Client  │                                           │  Video       │
│         │  1. Request upload URL                    │  Service     │
│         │──────────────────────────────────────────►│              │
│         │                                           │              │
│         │  2. Presigned URL + videoId               │              │
│         │◄──────────────────────────────────────────│              │
└────┬────┘                                           └──────────────┘
     │
     │  3. Upload video directly
     │
     ▼
┌──────────────┐
│ Cloudflare   │
│      R2      │
│              │
└──────┬───────┘
       │
       │  4. Upload complete
       │
       ▼
┌─────────┐                                           ┌──────────────┐
│ Client  │  5. Confirm upload                        │  Video       │
│         │──────────────────────────────────────────►│  Service     │
│         │                                           │              │
│         │  6. Video created (processing)            │              │
│         │◄──────────────────────────────────────────│              │
└─────────┘                                           └──────┬───────┘
                                                             │
                                                             │ 7. Publish to queue
                                                             ▼
                                                      ┌──────────────┐
                                                      │   Message    │
                                                      │    Queue     │
                                                      └──────┬───────┘
                                                             │
                                                             │ 8. Consume job
                                                             ▼
                                                      ┌──────────────┐
                                                      │    Video     │
                                                      │  Processor   │
                                                      │              │
                                                      │ - Transcode  │
                                                      │ - Generate   │
                                                      │   thumbnails │
                                                      │ - Upload R2  │
                                                      └──────┬───────┘
                                                             │
                                                             │ 9. Update status
                                                             ▼
                                                      ┌──────────────┐
                                                      │  PostgreSQL  │
                                                      │ status:ready │
                                                      └──────────────┘
```

### 6.2 Video Playback Flow

```
┌─────────┐                                           ┌──────────────┐
│ Client  │  1. Request video page                    │     API      │
│         │──────────────────────────────────────────►│   Gateway    │
│         │                                           └──────┬───────┘
│         │                                                  │
│         │                                                  │ 2. Fetch from cache
│         │                                                  ▼
│         │                                           ┌──────────────┐
│         │                                           │    Redis     │
│         │                                           │    Cache     │
│         │  3. Video metadata                        └──────┬───────┘
│         │◄─────────────────────────────────────────────────┘
│         │                                                  │ Cache miss
│         │                                                  ▼
│         │                                           ┌──────────────┐
│         │                                           │  PostgreSQL  │
│         │                                           └──────────────┘
│         │
│         │  4. Request HLS stream
│         │──────────────────────────────────────────►
│         │                                           ┌──────────────┐
│         │  5. HLS manifest                          │ Cloudflare   │
│         │◄──────────────────────────────────────────│   CDN + R2   │
│         │                                           └──────────────┘
│         │
│         │  6. Request video segments (.ts files)
│         │──────────────────────────────────────────►
│         │                                           
│         │  7. Stream segments
│         │◄──────────────────────────────────────────
└─────────┘
     │
     │  8. Track view event (async)
     ▼
┌──────────────┐
│  Analytics   │
│   Service    │
└──────────────┘
```

### 6.3 Recommendation Generation Flow

```
┌─────────────┐
│   Cron Job  │  Runs every 10 minutes
└──────┬──────┘
       │
       │  1. Trigger recommendation refresh
       ▼
┌──────────────────┐
│  Recommendation  │
│     Service      │
└──────┬───────────┘
       │
       │  2. Fetch user watch history
       ▼
┌──────────────────┐
│   PostgreSQL     │
└──────┬───────────┘
       │
       │  3. Compute embeddings
       ▼
┌──────────────────┐
│   Vector DB      │  Similarity search
│   (Pinecone)     │
└──────┬───────────┘
       │
       │  4. Hybrid scoring
       ▼
┌──────────────────┐
│  ML Model        │  Two-tower network
│  (PyTorch)       │
└──────┬───────────┘
       │
       │  5. Store recommendations
       ▼
┌──────────────────┐
│     Redis        │  Cache for 10 minutes
│  rec:{userId}    │
└──────────────────┘
```

---

## 7. Infrastructure

### 7.1 Deployment Architecture

```
┌───────────────────────────────────────────────────────────┐
│                     Cloudflare                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │     DNS     │  │     WAF     │  │     CDN     │       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
└───────────────────────────┬───────────────────────────────┘
                            │
                ┌───────────┼───────────┐
                │           │           │
        ┌───────▼────┐  ┌──▼────┐  ┌──▼────┐
        │  Vercel    │  │Railway│  │Fly.io │
        │ (Frontend) │  │  API  │  │Workers│
        └────────────┘  └───┬───┘  └───────┘
                            │
                ┌───────────┼───────────┐
                │           │           │
        ┌───────▼────┐  ┌──▼────┐  ┌──▼────┐
        │PostgreSQL  │  │ Redis │  │   R2  │
        │ (Supabase) │  │(Upstash)│ │(Videos)│
        └────────────┘  └───────┘  └───────┘
```

### 7.2 Service Hosting

| Service | Hosting | Reason |
|---------|---------|--------|
| **Frontend (Web)** | Vercel | Automatic Next.js optimization, global CDN |
| **API Gateway** | Railway / Fly.io | Docker support, auto-scaling |
| **Auth Service** | Railway | Stateless, easy deployment |
| **Video Service** | Railway | Handles API traffic |
| **Workers** | Fly.io | Close to R2 (latency), Docker |
| **AI Service** | Fly.io (GPU) | GPU support for ML models |
| **PostgreSQL** | Supabase (free tier) | Managed DB, auto-backups |
| **Redis** | Upstash | Serverless Redis, pay-per-use |
| **Elasticsearch** | Elastic Cloud | Managed, scalable search |
| **Object Storage** | Cloudflare R2 | Zero egress fees, fast CDN |

### 7.3 Docker Compose (Local Development)

```yaml
version: '3.8'

services:
  postgres:
    image: timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_DB: streamhub
      POSTGRES_USER: streamhub
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: streamhub
      RABBITMQ_DEFAULT_PASS: password

  api-gateway:
    build: ./services/api-gateway
    ports:
      - "3000:3000"
    depends_on:
      - postgres
      - redis
    environment:
      DATABASE_URL: postgres://streamhub:password@postgres:5432/streamhub
      REDIS_URL: redis://redis:6379

  auth-service:
    build: ./services/auth-service
    ports:
      - "3001:3001"
    depends_on:
      - postgres
      - redis

  video-service:
    build: ./services/video-service
    ports:
      - "3002:3002"
    depends_on:
      - postgres
      - redis
      - rabbitmq

  video-processor:
    build: ./workers/video-processor
    depends_on:
      - rabbitmq
    environment:
      RABBITMQ_URL: amqp://streamhub:password@rabbitmq:5672

  ai-moderation:
    build: ./workers/ai-moderation
    depends_on:
      - rabbitmq
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

volumes:
  postgres_data:
  redis_data:
```

### 7.4 CI/CD Pipeline (GitHub Actions)

```yaml
name: Deploy API Services

on:
  push:
    branches: [main]
    paths:
      - 'services/**'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test
      - run: npm run lint

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker image
        run: docker build -t streamhub-api:${{ github.sha }} .
      
      - name: Push to Registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
          docker push streamhub-api:${{ github.sha }}
      
      - name: Deploy to Railway
        run: |
          railway up --service api-gateway
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
```

---

## 8. Security Architecture

### 8.1 Authentication & Authorization

```
Request Flow with Auth:

1. Client Request
   └─► API Gateway
       ├─► Extract JWT from Authorization header
       ├─► Verify signature (HS256 with secret key)
       ├─► Check expiration
       ├─► Check Redis blacklist (for logout)
       │
       ├─► Valid: Forward to service with userId in header
       │
       └─► Invalid: Return 401 Unauthorized
```

**JWT Secret Rotation:**
- Rotate secret every 90 days
- Store in environment variables (not in code)
- Use different secrets for prod/staging/dev

### 8.2 Rate Limiting

```
Rate Limit Tiers:

┌──────────────────┬────────────┬──────────────┐
│   Endpoint       │ Limit      │ Window       │
├──────────────────┼────────────┼──────────────┤
│ POST /login      │ 5 req      │ 1 minute     │
│ POST /register   │ 3 req      │ 1 hour       │
│ POST /videos     │ 10 uploads │ 1 day        │
│ POST /comments   │ 20 req     │ 1 minute     │
│ GET /*           │ 1000 req   │ 1 minute     │
└──────────────────┴────────────┴──────────────┘

Implementation (Redis):

const rateLimit = async (key, limit, window) => {
  const current = await redis.incr(key);
  if (current === 1) {
    await redis.expire(key, window);
  }
  return current <= limit;
};

// Usage
const allowed = await rateLimit(`ratelimit:login:${ip}`, 5, 60);
if (!allowed) {
  return res.status(429).json({ error: "Too many requests" });
}
```

### 8.3 Input Validation & Sanitization

```typescript
import { z } from 'zod';

// Video creation schema
const createVideoSchema = z.object({
  title: z.string()
    .min(3, 'Title too short')
    .max(200, 'Title too long')
    .trim(),
  description: z.string()
    .max(5000, 'Description too long')
    .trim()
    .optional(),
  tags: z.array(z.string())
    .max(10, 'Too many tags')
    .optional(),
  visibility: z.enum(['public', 'unlisted', 'private'])
});

// Validate request
app.post('/api/v1/videos', async (req, res) => {
  try {
    const data = createVideoSchema.parse(req.body);
    // Sanitize HTML in description
    data.description = sanitizeHtml(data.description, {
      allowedTags: [], // No HTML allowed
      allowedAttributes: {}
    });
    // Process...
  } catch (error) {
    return res.status(400).json({ error: error.errors });
  }
});
```

### 8.4 SQL Injection Prevention

```typescript
// ❌ BAD (Vulnerable to SQL injection)
const query = `SELECT * FROM users WHERE username = '${username}'`;

// ✅ GOOD (Parameterized queries)
const query = 'SELECT * FROM users WHERE username = $1';
const result = await db.query(query, [username]);
```

### 8.5 CORS Configuration

```typescript
import cors from 'cors';

app.use(cors({
  origin: [
    'https://streamhub.com',
    'https://www.streamhub.com',
    'http://localhost:3000' // Dev only
  ],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### 8.6 Secrets Management

```bash
# Environment Variables (Never commit .env)

# Database
DATABASE_URL=postgres://user:password@host:5432/db
REDIS_URL=redis://host:6379

# Auth
JWT_SECRET=your-super-secret-key-min-32-chars
OAUTH_GOOGLE_CLIENT_ID=xxx
OAUTH_GOOGLE_CLIENT_SECRET=xxx

# Storage
R2_ACCOUNT_ID=xxx
R2_ACCESS_KEY_ID=xxx
R2_SECRET_ACCESS_KEY=xxx
R2_BUCKET_NAME=streamhub-videos

# External APIs
PERSPECTIVE_API_KEY=xxx
STRIPE_SECRET_KEY=xxx
```

**Production Secrets:**
- Use environment variables (Railway/Vercel)
- Never commit to git
- Rotate regularly
- Use different secrets per environment

---

## 9. Performance & Scalability

### 9.1 Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| API Response Time (p95) | <500ms | Datadog APM |
| Page Load Time | <2s | Lighthouse |
| Video Start Time | <3s | Custom metric |
| Database Query Time (p95) | <100ms | pg_stat_statements |
| Search Latency (p95) | <200ms | Elasticsearch metrics |
| Uptime | 99.9% | Uptime Robot |

### 9.2 Caching Strategy

```
Multi-Layer Caching:

┌─────────────┐
│   Browser   │  Cache-Control: public, max-age=31536000 (static assets)
└──────┬──────┘
       │
┌──────▼──────┐
│ Cloudflare  │  CDN cache (videos, images, CSS/JS)
│     CDN     │  TTL: 1 year for immutable assets
└──────┬──────┘
       │
┌──────▼──────┐
│    Redis    │  Application cache (metadata, sessions)
│    Cache    │  TTL: varies (5min - 24hr)
└──────┬──────┘
       │
┌──────▼──────┐
│ PostgreSQL  │  Database (source of truth)
└─────────────┘
```

### 9.3 Database Optimization

**Read Replicas:**

```
Write:
Client ──► API ──► PostgreSQL Primary

Read (90% of queries):
Client ──► API ──► PostgreSQL Replica 1
                └──► PostgreSQL Replica 2
```

**Connection Pooling:**

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  max: 20, // Max connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

**Query Optimization:**

```sql
-- ❌ Slow: Full table scan
SELECT * FROM videos WHERE LOWER(title) LIKE '%tutorial%';

-- ✅ Fast: Use full-text search index
SELECT * FROM videos WHERE title_tsv @@ to_tsquery('tutorial');

-- Create GIN index for full-text search
CREATE INDEX idx_videos_title_tsv ON videos USING GIN(to_tsvector('english', title));
```

### 9.4 Horizontal Scaling

**API Services:**

```
Load Balancer (NGINX)
  ├─► API Instance 1 (Node.js)
  ├─► API Instance 2 (Node.js)
  ├─► API Instance 3 (Node.js)
  └─► API Instance N (Auto-scale based on CPU/memory)

Auto-scaling Rules:
- Scale up when CPU > 70% for 5 minutes
- Scale down when CPU < 30% for 10 minutes
- Min instances: 2
- Max instances: 10
```

**Worker Scaling:**

```typescript
// BullMQ concurrency
const worker = new Worker('video-processing', processVideo, {
  concurrency: 5, // Process 5 videos simultaneously
  limiter: {
    max: 10, // Max 10 jobs per second
    duration: 1000
  }
});
```

### 9.5 CDN & Asset Optimization

**Cloudflare R2 + CDN:**

```
Video Delivery:
User Request
  └─► Cloudflare CDN (nearest edge location)
      ├─► Cache HIT: Serve immediately
      └─► Cache MISS: Fetch from R2, cache, then serve

Benefits:
- Global edge network (>200 locations)
- Zero egress fees
- Automatic compression (Brotli/Gzip)
- HTTP/2 & HTTP/3 support
```

**Image Optimization:**

```html
<!-- Next.js Image Component (automatic optimization) -->
<Image 
  src="/thumbnail.jpg" 
  width={640} 
  height={360}
  loading="lazy"
  quality={75}
  format="webp"
/>

<!-- Generates:
  - WebP format (smaller size)
  - Multiple sizes (responsive)
  - Lazy loading
  - Blur placeholder
-->
```

---

## 10. Monitoring & Observability

### 10.1 Logging

**Winston Logger:**

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  defaultMeta: { service: 'video-service' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// Usage
logger.info('Video uploaded', { 
  videoId, 
  userId, 
  fileSize,
  duration: Date.now() - startTime 
});

logger.error('Video processing failed', { 
  videoId, 
  error: error.message,
  stack: error.stack 
});
```

**Log Levels:**
- `error`: Critical errors (failed uploads, DB errors)
- `warn`: Warnings (slow queries, deprecated APIs)
- `info`: Important events (user signup, video upload)
- `debug`: Detailed debugging (dev only)

### 10.2 Metrics

**Prometheus + Grafana:**

```typescript
import promClient from 'prom-client';

// Metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_ms',
  help: 'Duration of HTTP requests in ms',
  labelNames: ['method', 'route', 'status_code']
});

const videoUploads = new promClient.Counter({
  name: 'video_uploads_total',
  help: 'Total video uploads'
});

// Middleware
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    httpRequestDuration
      .labels(req.method, req.route?.path, res.statusCode)
      .observe(Date.now() - start);
  });
  next();
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

**Key Metrics to Track:**
- Request rate (req/sec)
- Error rate (%)
- Response time (p50, p95, p99)
- Database query time
- Queue depth
- Video processing time
- CDN hit rate

### 10.3 Error Tracking

**Sentry Integration:**

```typescript
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1, // 10% of transactions
});

// Error handler
app.use((err, req, res, next) => {
  Sentry.captureException(err, {
    user: { id: req.user?.id },
    tags: { route: req.route?.path }
  });
  
  res.status(500).json({ error: 'Internal server error' });
});
```

### 10.4 Health Checks

```typescript
app.get('/health', async (req, res) => {
  const checks = {
    uptime: process.uptime(),
    timestamp: Date.now(),
    database: await checkDatabase(),
    redis: await checkRedis(),
    storage: await checkR2()
  };
  
  const healthy = Object.values(checks).every(v => v !== false);
  res.status(healthy ? 200 : 503).json(checks);
});

async function checkDatabase() {
  try {
    await db.query('SELECT 1');
    return true;
  } catch (error) {
    logger.error('Database health check failed', { error });
    return false;
  }
}
```

### 10.5 Alerting

**Alert Rules (Prometheus Alertmanager):**

```yaml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          
      - alert: SlowResponses
        expr: histogram_quantile(0.95, http_request_duration_ms) > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "95th percentile response time > 1s"
          
      - alert: DatabaseDown
        expr: up{job="postgres"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down"
```

---

## 11. Future Optimizations

### Phase 2 (Months 7-12)

1. **Database Sharding:** Split users/videos by ID range
2. **Geo-Distributed Databases:** Multi-region PostgreSQL
3. **Advanced Caching:** Redis Cluster with replication
4. **Search Optimization:** Elasticsearch cluster (3+ nodes)
5. **AI Model Optimization:** TensorRT for faster inference

### Phase 3 (Months 13-18)

1. **Kubernetes Migration:** Container orchestration
2. **Service Mesh:** Istio for advanced traffic management
3. **Event Streaming:** Apache Kafka for real-time data
4. **Graph Database:** Neo4j for social graph queries
5. **Edge Computing:** Cloudflare Workers for processing

---

## Appendix

### A. Technology Stack Summary

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Frontend** | Next.js 14, React, Tailwind CSS | Web UI |
| **Mobile** | React Native | iOS/Android apps |
| **API Gateway** | Kong / NGINX | Routing, auth, rate limiting |
| **Backend Services** | Node.js, TypeScript, Fastify | Microservices |
| **AI Services** | Python, FastAPI, PyTorch | ML models |
| **Database** | PostgreSQL 16 + TimescaleDB | Primary data store |
| **Cache** | Redis 7 | Session, caching |
| **Search** | Elasticsearch 8 | Full-text search |
| **Queue** | BullMQ / RabbitMQ | Async processing |
| **Storage** | Cloudflare R2 | Video/image storage |
| **CDN** | Cloudflare | Global content delivery |
| **Monitoring** | Sentry, Prometheus, Grafana | Observability |
| **CI/CD** | GitHub Actions | Automation |
| **Hosting** | Vercel, Railway, Fly.io | Deployment |

### B. Glossary

- **HLS (HTTP Live Streaming):** Adaptive bitrate streaming protocol
- **JWT (JSON Web Token):** Stateless authentication token
- **CQRS:** Command Query Responsibility Segregation
- **CDN:** Content Delivery Network
- **APM:** Application Performance Monitoring
- **TTL:** Time To Live (cache expiration)
- **p95:** 95th percentile (performance metric)

---

**End of Technical Design Document**

*Version: 1.0 | Last Updated: February 2026*
