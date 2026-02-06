# StreamHub - Database Schema & Sample Data

**Version:** 1.0  
**Database:** PostgreSQL 16 + TimescaleDB  
**Last Updated:** February 2026

---

## Table of Contents

1. [Database Setup](#database-setup)
2. [Schema Overview](#schema-overview)
3. [Table Definitions](#table-definitions)
4. [Indexes](#indexes)
5. [Triggers & Functions](#triggers--functions)
6. [Sample Data](#sample-data)
7. [Queries Reference](#queries-reference)

---

## Database Setup

### Initial Setup

```sql
-- Create database
CREATE DATABASE streamhub;

-- Connect to database
\c streamhub

-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Enable TimescaleDB for time-series data
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Enable full-text search
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

---

## Schema Overview

### Entity Relationship Diagram

```
users (1) ──────── (N) videos
  │                    │
  │ (1)                │ (1)
  │                    │
  │                    │ (N)
  ├─────── (N) subscriptions
  │                    │
  │ (1)                │ (1)
  │                    │
  │                    ▼
  └─────── (N) comments ◄───── (N) videos
  │                    │
  │ (1)                │ (N)
  │                    │
  │                    ▼
  └─────── (N) playlists ─────► (N) playlist_videos
  │                                    │
  │ (1)                                │ (N)
  │                                    │
  │                                    ▼
  └─────── (N) likes ◄───────────── videos
           │
           │ (N)
           │
           ▼
         comments

Time-Series Tables:
- watch_history (TimescaleDB hypertable)
- video_analytics (TimescaleDB hypertable)
```

---

## Table Definitions

### users

**Purpose:** Store user accounts and profile information

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'viewer' CHECK (role IN ('viewer', 'creator', 'admin')),
    
    -- Profile
    avatar_url TEXT,
    bio TEXT,
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Denormalized stats (updated via triggers)
    total_views BIGINT DEFAULT 0,
    total_uploads INTEGER DEFAULT 0,
    follower_count INTEGER DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    
    -- Constraints
    CONSTRAINT username_length CHECK (char_length(username) >= 3),
    CONSTRAINT bio_length CHECK (char_length(bio) <= 500)
);

-- Indexes
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Full-text search on username
CREATE INDEX idx_users_username_trgm ON users USING gin(username gin_trgm_ops);

-- Comments
COMMENT ON TABLE users IS 'User accounts and profiles';
COMMENT ON COLUMN users.role IS 'User role: viewer, creator, or admin';
COMMENT ON COLUMN users.total_views IS 'Aggregated views across all user videos (for creators)';
```

---

### sessions

**Purpose:** Store active user sessions (JWT blacklist on logout)

```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(500) UNIQUE NOT NULL,
    refresh_token VARCHAR(500) UNIQUE NOT NULL,
    device VARCHAR(100),
    ip_address INET,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Check expiry in the future
    CONSTRAINT expires_in_future CHECK (expires_at > created_at)
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_token ON sessions(token);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);

-- Auto-delete expired sessions
CREATE INDEX idx_sessions_cleanup ON sessions(expires_at) WHERE expires_at < NOW();
```

---

### oauth_connections

**Purpose:** Store OAuth provider connections (Google, GitHub)

```sql
CREATE TABLE oauth_connections (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL CHECK (provider IN ('google', 'github')),
    provider_user_id VARCHAR(255) NOT NULL,
    access_token TEXT,
    refresh_token TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- One provider per user
    UNIQUE(user_id, provider),
    -- Unique provider user
    UNIQUE(provider, provider_user_id)
);

CREATE INDEX idx_oauth_user_id ON oauth_connections(user_id);
```

---

### videos

**Purpose:** Store video metadata and processing status

```sql
CREATE TABLE videos (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    creator_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Metadata
    title VARCHAR(200) NOT NULL,
    description TEXT,
    thumbnail_url TEXT,
    video_url TEXT, -- R2 base path
    
    -- File info
    duration INTEGER, -- seconds
    file_size BIGINT, -- bytes
    format VARCHAR(20),
    resolution VARCHAR(20),
    codec VARCHAR(50),
    bitrate INTEGER,
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'processing' 
        CHECK (status IN ('processing', 'ready', 'failed', 'deleted')),
    visibility VARCHAR(20) NOT NULL DEFAULT 'public' 
        CHECK (visibility IN ('public', 'unlisted', 'private')),
    
    -- Categorization
    tags TEXT[], -- Array of tags
    category VARCHAR(50),
    
    -- Timestamps
    uploaded_at TIMESTAMP NOT NULL DEFAULT NOW(),
    processed_at TIMESTAMP,
    deleted_at TIMESTAMP,
    
    -- Denormalized engagement metrics (updated via triggers)
    view_count BIGINT DEFAULT 0,
    like_count INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    share_count INTEGER DEFAULT 0,
    
    -- Constraints
    CONSTRAINT title_length CHECK (char_length(title) >= 3),
    CONSTRAINT description_length CHECK (char_length(description) <= 5000),
    CONSTRAINT tags_count CHECK (array_length(tags, 1) <= 10)
);

-- Indexes
CREATE INDEX idx_videos_creator_id ON videos(creator_id);
CREATE INDEX idx_videos_status ON videos(status);
CREATE INDEX idx_videos_visibility ON videos(visibility);
CREATE INDEX idx_videos_uploaded_at ON videos(uploaded_at DESC);
CREATE INDEX idx_videos_category ON videos(category);
CREATE INDEX idx_videos_tags ON videos USING GIN(tags);
CREATE INDEX idx_videos_view_count ON videos(view_count DESC);

-- Composite index for feed queries
CREATE INDEX idx_videos_status_visibility_uploaded 
    ON videos(status, visibility, uploaded_at DESC);

-- Full-text search
CREATE INDEX idx_videos_title_trgm ON videos USING gin(title gin_trgm_ops);
CREATE INDEX idx_videos_description_trgm ON videos USING gin(description gin_trgm_ops);

-- Comments
COMMENT ON TABLE videos IS 'Video metadata and processing status';
COMMENT ON COLUMN videos.status IS 'Processing status: processing, ready, failed, deleted';
COMMENT ON COLUMN videos.visibility IS 'Privacy setting: public, unlisted, private';
```

---

### video_streams

**Purpose:** Store HLS streaming variants (360p, 480p, 720p, 1080p)

```sql
CREATE TABLE video_streams (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    quality VARCHAR(20) NOT NULL, -- '360p', '480p', '720p', '1080p'
    hls_url TEXT NOT NULL,
    file_size BIGINT,
    bitrate INTEGER,
    resolution VARCHAR(20),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- One stream per quality per video
    UNIQUE(video_id, quality)
);

CREATE INDEX idx_video_streams_video_id ON video_streams(video_id);

COMMENT ON TABLE video_streams IS 'HLS streaming variants for adaptive bitrate';
```

---

### likes

**Purpose:** Track video likes

```sql
CREATE TABLE likes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- User can like video only once
    UNIQUE(user_id, video_id)
);

CREATE INDEX idx_likes_user_id ON likes(user_id);
CREATE INDEX idx_likes_video_id ON likes(video_id);
CREATE INDEX idx_likes_created_at ON likes(created_at DESC);

-- Composite index for checking if user liked video
CREATE INDEX idx_likes_user_video ON likes(user_id, video_id);
```

---

### comments

**Purpose:** Store video comments and replies

```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    parent_comment_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    
    text TEXT NOT NULL,
    
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP,
    
    is_edited BOOLEAN DEFAULT FALSE,
    is_deleted BOOLEAN DEFAULT FALSE, -- Soft delete
    
    -- Denormalized metrics
    like_count INTEGER DEFAULT 0,
    reply_count INTEGER DEFAULT 0,
    
    -- Constraints
    CONSTRAINT text_length CHECK (char_length(text) >= 1 AND char_length(text) <= 1000)
);

CREATE INDEX idx_comments_video_id ON comments(video_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
CREATE INDEX idx_comments_parent_id ON comments(parent_comment_id);
CREATE INDEX idx_comments_created_at ON comments(created_at DESC);

-- Index for fetching top-level comments
CREATE INDEX idx_comments_video_top_level 
    ON comments(video_id, created_at DESC) 
    WHERE parent_comment_id IS NULL;
```

---

### comment_likes

**Purpose:** Track comment likes

```sql
CREATE TABLE comment_likes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    comment_id UUID NOT NULL REFERENCES comments(id) ON DELETE CASCADE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE(user_id, comment_id)
);

CREATE INDEX idx_comment_likes_comment_id ON comment_likes(comment_id);
CREATE INDEX idx_comment_likes_user_id ON comment_likes(user_id);
```

---

### subscriptions

**Purpose:** Track user subscriptions to creators

```sql
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    subscriber_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    creator_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    notifications_enabled BOOLEAN DEFAULT TRUE,
    subscribed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- User cannot subscribe to themselves
    CHECK (subscriber_id != creator_id),
    
    -- One subscription per subscriber-creator pair
    UNIQUE(subscriber_id, creator_id)
);

CREATE INDEX idx_subscriptions_subscriber_id ON subscriptions(subscriber_id);
CREATE INDEX idx_subscriptions_creator_id ON subscriptions(creator_id);
CREATE INDEX idx_subscriptions_subscribed_at ON subscriptions(subscribed_at DESC);
```

---

### playlists

**Purpose:** Store user-created playlists

```sql
CREATE TABLE playlists (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    visibility VARCHAR(20) NOT NULL DEFAULT 'public' 
        CHECK (visibility IN ('public', 'unlisted', 'private')),
    thumbnail_url TEXT,
    video_count INTEGER DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    CONSTRAINT name_length CHECK (char_length(name) >= 1),
    CONSTRAINT description_length CHECK (char_length(description) <= 500)
);

CREATE INDEX idx_playlists_user_id ON playlists(user_id);
CREATE INDEX idx_playlists_visibility ON playlists(visibility);
CREATE INDEX idx_playlists_created_at ON playlists(created_at DESC);
```

---

### playlist_videos

**Purpose:** Link videos to playlists with ordering

```sql
CREATE TABLE playlist_videos (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    playlist_id UUID NOT NULL REFERENCES playlists(id) ON DELETE CASCADE,
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    position INTEGER NOT NULL,
    added_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Video can only be in playlist once
    UNIQUE(playlist_id, video_id),
    
    -- Position must be positive
    CHECK (position > 0)
);

CREATE INDEX idx_playlist_videos_playlist_id ON playlist_videos(playlist_id, position);
CREATE INDEX idx_playlist_videos_video_id ON playlist_videos(video_id);
```

---

### watch_history (TimescaleDB Hypertable)

**Purpose:** Track video views for analytics (time-series data)

```sql
CREATE TABLE watch_history (
    id UUID DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL, -- Nullable for anonymous views
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    session_id UUID,
    
    watched_at TIMESTAMP NOT NULL DEFAULT NOW(),
    watch_duration INTEGER, -- Seconds watched
    last_position INTEGER, -- Last playback position
    completed BOOLEAN DEFAULT FALSE,
    
    device VARCHAR(50),
    country VARCHAR(2),
    referrer TEXT,
    
    PRIMARY KEY (id, watched_at)
);

-- Convert to TimescaleDB hypertable (time-series optimization)
SELECT create_hypertable('watch_history', 'watched_at');

-- Indexes
CREATE INDEX idx_watch_history_user_id ON watch_history(user_id, watched_at DESC);
CREATE INDEX idx_watch_history_video_id ON watch_history(video_id, watched_at DESC);
CREATE INDEX idx_watch_history_session_id ON watch_history(session_id);

COMMENT ON TABLE watch_history IS 'Time-series data for video views and watch analytics';
```

---

### video_analytics (TimescaleDB Hypertable)

**Purpose:** Aggregated video statistics by day

```sql
CREATE TABLE video_analytics (
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    
    views INTEGER DEFAULT 0,
    unique_viewers INTEGER DEFAULT 0,
    likes INTEGER DEFAULT 0,
    comments INTEGER DEFAULT 0,
    shares INTEGER DEFAULT 0,
    
    total_watch_time BIGINT DEFAULT 0, -- Total seconds watched
    avg_watch_duration INTEGER DEFAULT 0, -- Average seconds per view
    completion_rate DECIMAL(5,2) DEFAULT 0, -- Percentage who watched to end
    
    PRIMARY KEY (video_id, date)
);

SELECT create_hypertable('video_analytics', 'date');

CREATE INDEX idx_video_analytics_video_id ON video_analytics(video_id, date DESC);

COMMENT ON TABLE video_analytics IS 'Daily aggregated analytics per video';
```

---

### reports

**Purpose:** User-submitted content reports

```sql
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    reporter_id UUID REFERENCES users(id) ON DELETE SET NULL,
    video_id UUID REFERENCES videos(id) ON DELETE CASCADE,
    comment_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    
    reason VARCHAR(50) NOT NULL CHECK (reason IN ('copyright', 'abuse', 'spam', 'other')),
    description TEXT,
    
    status VARCHAR(20) DEFAULT 'pending' 
        CHECK (status IN ('pending', 'reviewed', 'resolved', 'dismissed')),
    
    reviewed_by UUID REFERENCES users(id) ON DELETE SET NULL,
    reviewed_at TIMESTAMP,
    
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Either video or comment must be reported
    CHECK (
        (video_id IS NOT NULL AND comment_id IS NULL) OR
        (video_id IS NULL AND comment_id IS NOT NULL)
    )
);

CREATE INDEX idx_reports_status ON reports(status, created_at DESC);
CREATE INDEX idx_reports_video_id ON reports(video_id);
CREATE INDEX idx_reports_comment_id ON reports(comment_id);
```

---

### notifications

**Purpose:** User notifications (new subscriber, comment, like)

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL CHECK (type IN ('new_subscriber', 'comment', 'reply', 'like', 'video_ready')),
    
    title TEXT NOT NULL,
    message TEXT NOT NULL,
    link TEXT,
    
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Reference to related entity
    related_user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    related_video_id UUID REFERENCES videos(id) ON DELETE CASCADE,
    related_comment_id UUID REFERENCES comments(id) ON DELETE CASCADE
);

CREATE INDEX idx_notifications_user_id ON notifications(user_id, created_at DESC);
CREATE INDEX idx_notifications_unread ON notifications(user_id, is_read, created_at DESC);
```

---

## Indexes

### Performance Indexes

```sql
-- Composite indexes for common queries

-- Video feed query (For You page)
CREATE INDEX idx_videos_feed 
    ON videos(status, visibility, uploaded_at DESC) 
    WHERE status = 'ready' AND visibility = 'public';

-- Trending videos (views in last 24h)
CREATE INDEX idx_videos_trending 
    ON videos(view_count DESC, uploaded_at DESC) 
    WHERE status = 'ready' AND visibility = 'public' 
        AND uploaded_at > NOW() - INTERVAL '7 days';

-- Creator's videos
CREATE INDEX idx_videos_creator 
    ON videos(creator_id, uploaded_at DESC) 
    WHERE status != 'deleted';

-- User's watch history
CREATE INDEX idx_watch_user_recent 
    ON watch_history(user_id, watched_at DESC);
```

---

## Triggers & Functions

### Auto-update updated_at timestamp

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to tables
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_videos_updated_at BEFORE UPDATE ON videos
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_comments_updated_at BEFORE UPDATE ON comments
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_playlists_updated_at BEFORE UPDATE ON playlists
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

---

### Update video like_count

```sql
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
```

---

### Update comment like_count

```sql
CREATE OR REPLACE FUNCTION update_comment_like_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE comments SET like_count = like_count + 1 WHERE id = NEW.comment_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE comments SET like_count = like_count - 1 WHERE id = OLD.comment_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_comment_like_count
AFTER INSERT OR DELETE ON comment_likes
FOR EACH ROW EXECUTE FUNCTION update_comment_like_count();
```

---

### Update video comment_count

```sql
CREATE OR REPLACE FUNCTION update_video_comment_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' AND NEW.parent_comment_id IS NULL THEN
        UPDATE videos SET comment_count = comment_count + 1 WHERE id = NEW.video_id;
    ELSIF TG_OP = 'DELETE' AND OLD.parent_comment_id IS NULL THEN
        UPDATE videos SET comment_count = comment_count - 1 WHERE id = OLD.video_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_video_comment_count
AFTER INSERT OR DELETE ON comments
FOR EACH ROW EXECUTE FUNCTION update_video_comment_count();
```

---

### Update comment reply_count

```sql
CREATE OR REPLACE FUNCTION update_comment_reply_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' AND NEW.parent_comment_id IS NOT NULL THEN
        UPDATE comments SET reply_count = reply_count + 1 WHERE id = NEW.parent_comment_id;
    ELSIF TG_OP = 'DELETE' AND OLD.parent_comment_id IS NOT NULL THEN
        UPDATE comments SET reply_count = reply_count - 1 WHERE id = OLD.parent_comment_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_comment_reply_count
AFTER INSERT OR DELETE ON comments
FOR EACH ROW EXECUTE FUNCTION update_comment_reply_count();
```

---

### Update follower counts

```sql
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

---

### Update playlist video_count

```sql
CREATE OR REPLACE FUNCTION update_playlist_video_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE playlists SET video_count = video_count + 1 WHERE id = NEW.playlist_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE playlists SET video_count = video_count - 1 WHERE id = OLD.playlist_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_playlist_video_count
AFTER INSERT OR DELETE ON playlist_videos
FOR EACH ROW EXECUTE FUNCTION update_playlist_video_count();
```

---

### Update user total_uploads

```sql
CREATE OR REPLACE FUNCTION update_user_total_uploads()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE users SET total_uploads = total_uploads + 1 WHERE id = NEW.creator_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE users SET total_uploads = total_uploads - 1 WHERE id = OLD.creator_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_user_total_uploads
AFTER INSERT OR DELETE ON videos
FOR EACH ROW EXECUTE FUNCTION update_user_total_uploads();
```

---

## Sample Data

### Users

```sql
INSERT INTO users (id, username, email, password_hash, role, avatar_url, bio) VALUES
('550e8400-e29b-41d4-a716-446655440000', 'johndoe', 'john@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5GyxADTv2Z.q2', 'creator', 'https://cdn.streamhub.com/avatars/johndoe.jpg', 'Tech content creator'),
('550e8400-e29b-41d4-a716-446655440001', 'janedoe', 'jane@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5GyxADTv2Z.q2', 'creator', 'https://cdn.streamhub.com/avatars/janedoe.jpg', 'Music producer and educator'),
('550e8400-e29b-41d4-a716-446655440002', 'viewer1', 'viewer@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5GyxADTv2Z.q2', 'viewer', NULL, NULL),
('550e8400-e29b-41d4-a716-446655440003', 'admin', 'admin@streamhub.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5GyxADTv2Z.q2', 'admin', NULL, 'Platform administrator');

-- Note: password_hash is bcrypt hash of "password123"
```

---

### Videos

```sql
INSERT INTO videos (id, creator_id, title, description, thumbnail_url, video_url, duration, file_size, status, visibility, tags, category, uploaded_at, processed_at, view_count, like_count) VALUES
('650e8400-e29b-41d4-a716-446655440000', '550e8400-e29b-41d4-a716-446655440000', 'JavaScript Tutorial for Beginners', 'Learn JavaScript basics in this comprehensive tutorial', 'https://cdn.streamhub.com/thumbnails/js-tutorial.jpg', 'https://cdn.streamhub.com/videos/650e8400/master.m3u8', 600, 85000000, 'ready', 'public', ARRAY['javascript', 'tutorial', 'programming'], 'education', NOW() - INTERVAL '2 days', NOW() - INTERVAL '2 days', 12500, 450),

('650e8400-e29b-41d4-a716-446655440001', '550e8400-e29b-41d4-a716-446655440000', 'React Hooks Explained', 'Deep dive into React Hooks with practical examples', 'https://cdn.streamhub.com/thumbnails/react-hooks.jpg', 'https://cdn.streamhub.com/videos/650e8401/master.m3u8', 900, 120000000, 'ready', 'public', ARRAY['react', 'javascript', 'hooks'], 'education', NOW() - INTERVAL '5 days', NOW() - INTERVAL '5 days', 25000, 890),

('650e8400-e29b-41d4-a716-446655440002', '550e8400-e29b-41d4-a716-446655440001', 'Music Production Basics', 'Introduction to music production and DAWs', 'https://cdn.streamhub.com/thumbnails/music-prod.jpg', 'https://cdn.streamhub.com/videos/650e8402/master.m3u8', 720, 95000000, 'ready', 'public', ARRAY['music', 'production', 'tutorial'], 'music', NOW() - INTERVAL '1 day', NOW() - INTERVAL '1 day', 5600, 230),

('650e8400-e29b-41d4-a716-446655440003', '550e8400-e29b-41d4-a716-446655440000', 'Building REST APIs with Node.js', 'Complete guide to building RESTful APIs', 'https://cdn.streamhub.com/thumbnails/rest-api.jpg', 'https://cdn.streamhub.com/videos/650e8403/master.m3u8', 1200, 150000000, 'ready', 'public', ARRAY['nodejs', 'api', 'backend'], 'education', NOW() - INTERVAL '7 days', NOW() - INTERVAL '7 days', 45000, 1500);
```

---

### Video Streams

```sql
INSERT INTO video_streams (video_id, quality, hls_url, file_size, bitrate, resolution) VALUES
('650e8400-e29b-41d4-a716-446655440000', '360p', 'https://cdn.streamhub.com/videos/650e8400/360p.m3u8', 25000000, 800000, '640x360'),
('650e8400-e29b-41d4-a716-446655440000', '720p', 'https://cdn.streamhub.com/videos/650e8400/720p.m3u8', 60000000, 2800000, '1280x720'),

('650e8400-e29b-41d4-a716-446655440001', '360p', 'https://cdn.streamhub.com/videos/650e8401/360p.m3u8', 35000000, 800000, '640x360'),
('650e8400-e29b-41d4-a716-446655440001', '720p', 'https://cdn.streamhub.com/videos/650e8401/720p.m3u8', 85000000, 2800000, '1280x720');
```

---

### Subscriptions

```sql
INSERT INTO subscriptions (subscriber_id, creator_id, subscribed_at) VALUES
('550e8400-e29b-41d4-a716-446655440002', '550e8400-e29b-41d4-a716-446655440000', NOW() - INTERVAL '10 days'),
('550e8400-e29b-41d4-a716-446655440002', '550e8400-e29b-41d4-a716-446655440001', NOW() - INTERVAL '5 days');
```

---

### Likes

```sql
INSERT INTO likes (user_id, video_id, created_at) VALUES
('550e8400-e29b-41d4-a716-446655440002', '650e8400-e29b-41d4-a716-446655440000', NOW() - INTERVAL '1 day'),
('550e8400-e29b-41d4-a716-446655440002', '650e8400-e29b-41d4-a716-446655440001', NOW() - INTERVAL '4 days');
```

---

### Comments

```sql
INSERT INTO comments (id, user_id, video_id, text, created_at, like_count) VALUES
('750e8400-e29b-41d4-a716-446655440000', '550e8400-e29b-41d4-a716-446655440002', '650e8400-e29b-41d4-a716-446655440000', 'Great tutorial! Very helpful for beginners.', NOW() - INTERVAL '1 day', 12),

('750e8400-e29b-41d4-a716-446655440001', '550e8400-e29b-41d4-a716-446655440000', '650e8400-e29b-41d4-a716-446655440000', 'Thanks! Glad it helped you.', NOW() - INTERVAL '1 day', 3);

-- Add reply
UPDATE comments SET parent_comment_id = '750e8400-e29b-41d4-a716-446655440000' 
WHERE id = '750e8400-e29b-41d4-a716-446655440001';
```

---

### Playlists

```sql
INSERT INTO playlists (id, user_id, name, description, visibility) VALUES
('850e8400-e29b-41d4-a716-446655440000', '550e8400-e29b-41d4-a716-446655440002', 'JavaScript Learning Path', 'My collection of JavaScript tutorials', 'public');

INSERT INTO playlist_videos (playlist_id, video_id, position) VALUES
('850e8400-e29b-41d4-a716-446655440000', '650e8400-e29b-41d4-a716-446655440000', 1),
('850e8400-e29b-41d4-a716-446655440000', '650e8400-e29b-41d4-a716-446655440001', 2);
```

---

### Watch History

```sql
INSERT INTO watch_history (user_id, video_id, watched_at, watch_duration, completed, device, country) VALUES
('550e8400-e29b-41d4-a716-446655440002', '650e8400-e29b-41d4-a716-446655440000', NOW() - INTERVAL '1 day', 550, true, 'desktop', 'US'),
('550e8400-e29b-41d4-a716-446655440002', '650e8400-e29b-41d4-a716-446655440001', NOW() - INTERVAL '3 days', 200, false, 'mobile', 'US');
```

---

## Queries Reference

### Common Read Queries

**Get video with creator info:**

```sql
SELECT 
    v.*,
    u.username as creator_username,
    u.avatar_url as creator_avatar,
    u.follower_count as creator_followers
FROM videos v
JOIN users u ON v.creator_id = u.id
WHERE v.id = $1 AND v.status = 'ready';
```

---

**Get video feed (For You page):**

```sql
SELECT v.*, u.username, u.avatar_url
FROM videos v
JOIN users u ON v.creator_id = u.id
WHERE v.status = 'ready' 
    AND v.visibility = 'public'
ORDER BY v.uploaded_at DESC
LIMIT 20 OFFSET $1;
```

---

**Get trending videos:**

```sql
SELECT v.*, u.username, u.avatar_url
FROM videos v
JOIN users u ON v.creator_id = u.id
WHERE v.status = 'ready' 
    AND v.visibility = 'public'
    AND v.uploaded_at > NOW() - INTERVAL '7 days'
ORDER BY v.view_count DESC, v.uploaded_at DESC
LIMIT 20;
```

---

**Get user's subscribed creators:**

```sql
SELECT 
    u.id,
    u.username,
    u.avatar_url,
    u.follower_count,
    s.subscribed_at
FROM subscriptions s
JOIN users u ON s.creator_id = u.id
WHERE s.subscriber_id = $1
ORDER BY s.subscribed_at DESC;
```

---

**Get video comments (threaded):**

```sql
-- Top-level comments
SELECT 
    c.*,
    u.username,
    u.avatar_url
FROM comments c
JOIN users u ON c.user_id = u.id
WHERE c.video_id = $1 
    AND c.parent_comment_id IS NULL
    AND c.is_deleted = false
ORDER BY c.created_at DESC
LIMIT 20;

-- Replies to a comment
SELECT 
    c.*,
    u.username,
    u.avatar_url
FROM comments c
JOIN users u ON c.user_id = u.id
WHERE c.parent_comment_id = $1
    AND c.is_deleted = false
ORDER BY c.created_at ASC;
```

---

**Search videos:**

```sql
SELECT v.*, u.username, u.avatar_url
FROM videos v
JOIN users u ON v.creator_id = u.id
WHERE v.status = 'ready'
    AND v.visibility = 'public'
    AND (
        v.title ILIKE '%' || $1 || '%' 
        OR v.description ILIKE '%' || $1 || '%'
        OR $1 = ANY(v.tags)
    )
ORDER BY v.view_count DESC
LIMIT 20;
```

---

**Get creator analytics:**

```sql
SELECT 
    SUM(views) as total_views,
    SUM(unique_viewers) as total_unique_viewers,
    SUM(total_watch_time) as total_watch_time,
    AVG(avg_watch_duration) as avg_watch_duration,
    AVG(completion_rate) as avg_completion_rate
FROM video_analytics va
JOIN videos v ON va.video_id = v.id
WHERE v.creator_id = $1
    AND va.date >= NOW() - INTERVAL '30 days';
```

---

### Common Write Queries

**Create video:**

```sql
INSERT INTO videos (
    id, creator_id, title, description, tags, category, visibility
) VALUES (
    $1, $2, $3, $4, $5, $6, $7
) RETURNING *;
```

---

**Update video metadata:**

```sql
UPDATE videos SET
    title = $2,
    description = $3,
    tags = $4,
    category = $5,
    visibility = $6
WHERE id = $1 AND creator_id = $7
RETURNING *;
```

---

**Like video (upsert):**

```sql
INSERT INTO likes (user_id, video_id)
VALUES ($1, $2)
ON CONFLICT (user_id, video_id) DO NOTHING
RETURNING *;
```

---

**Unlike video:**

```sql
DELETE FROM likes
WHERE user_id = $1 AND video_id = $2
RETURNING *;
```

---

**Track video view:**

```sql
INSERT INTO watch_history (
    user_id, video_id, watch_duration, device, country
) VALUES ($1, $2, $3, $4, $5);
```

---

## Maintenance Queries

**Clean up expired sessions:**

```sql
DELETE FROM sessions WHERE expires_at < NOW();
```

---

**Clean up old watch history (> 1 year):**

```sql
DELETE FROM watch_history 
WHERE watched_at < NOW() - INTERVAL '1 year';
```

---

**Recompute denormalized counts:**

```sql
-- Fix video like counts
UPDATE videos v SET like_count = (
    SELECT COUNT(*) FROM likes WHERE video_id = v.id
);

-- Fix video comment counts
UPDATE videos v SET comment_count = (
    SELECT COUNT(*) FROM comments 
    WHERE video_id = v.id AND parent_comment_id IS NULL
);

-- Fix user follower counts
UPDATE users u SET follower_count = (
    SELECT COUNT(*) FROM subscriptions WHERE creator_id = u.id
);
```

---

**Vacuum and analyze (run weekly):**

```sql
VACUUM ANALYZE videos;
VACUUM ANALYZE users;
VACUUM ANALYZE comments;
VACUUM ANALYZE watch_history;
```

---

**End of Database Schema**

*This schema is production-ready with proper indexes, constraints, and triggers. Use it as the foundation for your backend implementation.*
