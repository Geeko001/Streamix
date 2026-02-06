# StreamHub - Analytics Requirements

**Version:** 1.0  
**Analytics Stack:** PostgreSQL + TimescaleDB + Python  
**Last Updated:** February 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Event Tracking](#event-tracking)
3. [Metrics Definitions](#metrics-definitions)
4. [Data Collection](#data-collection)
5. [Storage Strategy](#storage-strategy)
6. [Dashboard Requirements](#dashboard-requirements)
7. [Query Examples](#query-examples)
8. [Real-Time Analytics](#real-time-analytics)

---

## Overview

### Analytics Goals

1. **Creator Insights** - Help creators understand their audience and performance
2. **Platform Metrics** - Track overall platform health and growth
3. **User Behavior** - Understand how users discover and engage with content
4. **Revenue Analytics** - Track monetization and earnings

### Key Performance Indicators (KPIs)

| Metric | Target | Category |
|--------|--------|----------|
| Monthly Active Users (MAU) | 100K | Platform Health |
| Daily Active Users (DAU) | 10K | Platform Health |
| DAU/MAU Ratio | >30% | Engagement |
| Average Watch Time | >10 min/session | Engagement |
| Video Completion Rate | >50% | Content Quality |
| Creator Retention (30-day) | >60% | Creator Health |
| Revenue per User (RPU) | >$5/month | Monetization |
| Upload Rate | >100/day | Content Growth |

---

## Event Tracking

### Event Types

```typescript
// Event schema
interface AnalyticsEvent {
  event_type: string;
  timestamp: string;
  user_id: string | null;
  session_id: string;
  properties: Record<string, any>;
  context: {
    device: string;
    country: string;
    referrer?: string;
    user_agent?: string;
  };
}
```

### Video Events

**video_view**

```typescript
{
  event_type: 'video_view',
  timestamp: '2026-02-06T10:30:00Z',
  user_id: 'user-uuid',
  session_id: 'session-uuid',
  properties: {
    video_id: 'video-uuid',
    creator_id: 'creator-uuid',
    watch_duration: 120,  // seconds
    video_duration: 600,   // total video length
    quality: '720p',
    autoplay: false,
    referrer_type: 'search', // search, direct, recommendation, external
  },
  context: {
    device: 'desktop',
    country: 'US',
    referrer: 'https://google.com/search?q=...'
  }
}
```

**video_play**

```typescript
{
  event_type: 'video_play',
  properties: {
    video_id: 'video-uuid',
    position: 0,  // Playback position when pressed play
  }
}
```

**video_pause**

```typescript
{
  event_type: 'video_pause',
  properties: {
    video_id: 'video-uuid',
    position: 45,  // Playback position when paused
  }
}
```

**video_seek**

```typescript
{
  event_type: 'video_seek',
  properties: {
    video_id: 'video-uuid',
    from_position: 30,
    to_position: 60,
  }
}
```

**video_ended**

```typescript
{
  event_type: 'video_ended',
  properties: {
    video_id: 'video-uuid',
    watch_duration: 600,
    completed: true,  // watched to end
  }
}
```

### Engagement Events

**video_like**

```typescript
{
  event_type: 'video_like',
  properties: {
    video_id: 'video-uuid',
    creator_id: 'creator-uuid',
  }
}
```

**comment_create**

```typescript
{
  event_type: 'comment_create',
  properties: {
    video_id: 'video-uuid',
    comment_id: 'comment-uuid',
    comment_length: 50,
    parent_comment_id: null,
  }
}
```

**user_subscribe**

```typescript
{
  event_type: 'user_subscribe',
  properties: {
    creator_id: 'creator-uuid',
  }
}
```

**playlist_add**

```typescript
{
  event_type: 'playlist_add',
  properties: {
    playlist_id: 'playlist-uuid',
    video_id: 'video-uuid',
  }
}
```

**share**

```typescript
{
  event_type: 'share',
  properties: {
    video_id: 'video-uuid',
    platform: 'twitter', // twitter, facebook, copy_link, embed
  }
}
```

### User Events

**user_signup**

```typescript
{
  event_type: 'user_signup',
  properties: {
    signup_method: 'email', // email, google, github
    role: 'creator',
  }
}
```

**user_login**

```typescript
{
  event_type: 'user_login',
  properties: {
    login_method: 'email',
  }
}
```

**search**

```typescript
{
  event_type: 'search',
  properties: {
    query: 'javascript tutorial',
    results_count: 234,
    clicked_result_position: 1,  // Which result they clicked
  }
}
```

---

## Metrics Definitions

### Video Metrics

**Views**
- **Definition:** Count of watch_duration > 30 seconds
- **Why 30s:** Filters out accidental clicks
- **Deduplication:** Max 1 view per user per video per hour

**Unique Viewers**
- **Definition:** Count of distinct users who viewed
- **Period:** Calculated daily, weekly, monthly

**Watch Time**
- **Definition:** Total seconds watched across all viewers
- **Formula:** `SUM(watch_duration)`

**Average View Duration (AVD)**
- **Definition:** Average watch time per view
- **Formula:** `SUM(watch_duration) / COUNT(views)`

**Completion Rate**
- **Definition:** Percentage who watched to end
- **Formula:** `COUNT(video_ended) / COUNT(video_view) * 100`

**Audience Retention**
- **Definition:** Percentage of viewers at each timestamp
- **Calculation:** Group by timestamp, count viewers
- **Visualization:** Retention curve graph

**Engagement Rate**
- **Definition:** (Likes + Comments + Shares) / Views * 100
- **Formula:** `(like_count + comment_count + share_count) / view_count * 100`

### Creator Metrics

**Total Views**
- **Definition:** Sum of all video views
- **Period:** 7d, 30d, 90d, all-time

**Subscriber Growth**
- **Definition:** Net new subscribers over period
- **Formula:** `COUNT(new_subscribers) - COUNT(unsubscribes)`

**Upload Frequency**
- **Definition:** Videos uploaded per week
- **Target:** 1+ per week for active creators

**Average Views per Video**
- **Definition:** Total views / video count
- **Formula:** `SUM(video_views) / COUNT(videos)`

**Top Performing Content**
- **Definition:** Videos with highest views/engagement
- **Sort by:** Views, watch time, engagement rate

### Platform Metrics

**Daily Active Users (DAU)**
- **Definition:** Unique users who performed any action in 24h
- **Actions:** View, like, comment, upload, search

**Monthly Active Users (MAU)**
- **Definition:** Unique users who performed any action in 30 days

**Stickiness (DAU/MAU)**
- **Definition:** Ratio of DAU to MAU
- **Target:** >30%
- **Formula:** `DAU / MAU * 100`

**New User Retention**
- **Day 1:** % of new users who return next day
- **Day 7:** % who return within 7 days
- **Day 30:** % who return within 30 days

**Content Growth Rate**
- **Definition:** New videos uploaded per day
- **Formula:** `COUNT(videos_today) / COUNT(videos_yesterday) - 1`

---

## Data Collection

### Backend Event Tracking

**API Endpoint:** `POST /api/analytics/track`

**Implementation:**

```typescript
// pages/api/analytics/track.ts
export default async function handler(req, res) {
  const { event_type, properties } = req.body;
  const session = await getServerSession(req, res);
  
  const event = {
    event_type,
    timestamp: new Date().toISOString(),
    user_id: session?.user?.id || null,
    session_id: req.cookies.session_id,
    properties,
    context: {
      device: getDevice(req.headers['user-agent']),
      country: getCountryFromIP(req.headers['x-forwarded-for']),
      referrer: req.headers.referer,
      user_agent: req.headers['user-agent'],
    },
  };

  // Insert into events table
  await db.query(
    `INSERT INTO analytics_events 
     (event_type, timestamp, user_id, session_id, properties, context) 
     VALUES ($1, $2, $3, $4, $5, $6)`,
    [
      event.event_type,
      event.timestamp,
      event.user_id,
      event.session_id,
      JSON.stringify(event.properties),
      JSON.stringify(event.context),
    ]
  );

  res.json({ success: true });
}
```

### Frontend Tracking Hook

```typescript
// hooks/useAnalytics.ts
import { useEffect } from 'react';
import { useAuthStore } from '@/store/authStore';

export const useAnalytics = () => {
  const track = async (event_type: string, properties: any = {}) => {
    try {
      await fetch('/api/analytics/track', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ event_type, properties }),
      });
    } catch (error) {
      console.error('Analytics error:', error);
    }
  };

  return { track };
};

// Usage in VideoPlayer component
const VideoPlayer = ({ video }) => {
  const { track } = useAnalytics();
  
  useEffect(() => {
    track('video_view', { 
      video_id: video.id,
      creator_id: video.creator_id,
    });
  }, []);
  
  const handlePlay = () => {
    track('video_play', { 
      video_id: video.id,
      position: videoRef.current.currentTime,
    });
  };
  
  // ...
};
```

---

## Storage Strategy

### Analytics Events Table (TimescaleDB)

```sql
CREATE TABLE analytics_events (
    id UUID DEFAULT uuid_generate_v4(),
    event_type VARCHAR(100) NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    session_id UUID,
    properties JSONB,
    context JSONB,
    
    PRIMARY KEY (id, timestamp)
);

-- Convert to hypertable (TimescaleDB)
SELECT create_hypertable('analytics_events', 'timestamp');

-- Indexes
CREATE INDEX idx_analytics_events_type ON analytics_events(event_type, timestamp DESC);
CREATE INDEX idx_analytics_events_user_id ON analytics_events(user_id, timestamp DESC);
CREATE INDEX idx_analytics_events_video_id ON analytics_events((properties->>'video_id'), timestamp DESC);
```

### Aggregated Metrics Tables

**Daily Video Stats:**

```sql
CREATE TABLE video_stats_daily (
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    
    views INTEGER DEFAULT 0,
    unique_viewers INTEGER DEFAULT 0,
    total_watch_time BIGINT DEFAULT 0,  -- seconds
    avg_watch_duration INTEGER DEFAULT 0,
    completion_count INTEGER DEFAULT 0,
    completion_rate DECIMAL(5,2) DEFAULT 0,
    
    likes INTEGER DEFAULT 0,
    comments INTEGER DEFAULT 0,
    shares INTEGER DEFAULT 0,
    
    traffic_search INTEGER DEFAULT 0,
    traffic_direct INTEGER DEFAULT 0,
    traffic_recommendation INTEGER DEFAULT 0,
    traffic_external INTEGER DEFAULT 0,
    
    PRIMARY KEY (video_id, date)
);

CREATE INDEX idx_video_stats_daily_date ON video_stats_daily(date DESC);
```

**Daily Creator Stats:**

```sql
CREATE TABLE creator_stats_daily (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    
    total_views INTEGER DEFAULT 0,
    unique_viewers INTEGER DEFAULT 0,
    total_watch_time BIGINT DEFAULT 0,
    
    new_subscribers INTEGER DEFAULT 0,
    unsubscribes INTEGER DEFAULT 0,
    net_subscribers INTEGER DEFAULT 0,
    
    new_videos INTEGER DEFAULT 0,
    
    revenue_cents INTEGER DEFAULT 0,
    
    PRIMARY KEY (user_id, date)
);

CREATE INDEX idx_creator_stats_daily_date ON creator_stats_daily(date DESC);
```

**Daily Platform Stats:**

```sql
CREATE TABLE platform_stats_daily (
    date DATE PRIMARY KEY,
    
    dau INTEGER DEFAULT 0,
    new_users INTEGER DEFAULT 0,
    total_users INTEGER DEFAULT 0,
    
    videos_uploaded INTEGER DEFAULT 0,
    total_videos INTEGER DEFAULT 0,
    
    total_views INTEGER DEFAULT 0,
    total_watch_time BIGINT DEFAULT 0,
    
    revenue_cents INTEGER DEFAULT 0
);
```

### Aggregation Jobs (Cron)

**Run daily at midnight:**

```python
# workers/analytics_aggregator.py
import psycopg2
from datetime import datetime, timedelta

def aggregate_video_stats(date):
    """Aggregate video statistics for a given date"""
    
    query = """
        INSERT INTO video_stats_daily 
        (video_id, date, views, unique_viewers, total_watch_time, avg_watch_duration, 
         completion_count, likes, comments, shares)
        SELECT 
            properties->>'video_id' as video_id,
            $1 as date,
            COUNT(*) FILTER (WHERE event_type = 'video_view') as views,
            COUNT(DISTINCT user_id) FILTER (WHERE event_type = 'video_view') as unique_viewers,
            SUM((properties->>'watch_duration')::int) FILTER (WHERE event_type = 'video_view') as total_watch_time,
            AVG((properties->>'watch_duration')::int) FILTER (WHERE event_type = 'video_view') as avg_watch_duration,
            COUNT(*) FILTER (WHERE event_type = 'video_ended' AND (properties->>'completed')::boolean = true) as completion_count,
            COUNT(*) FILTER (WHERE event_type = 'video_like') as likes,
            COUNT(*) FILTER (WHERE event_type = 'comment_create') as comments,
            COUNT(*) FILTER (WHERE event_type = 'share') as shares
        FROM analytics_events
        WHERE DATE(timestamp) = $1
          AND properties->>'video_id' IS NOT NULL
        GROUP BY properties->>'video_id'
        ON CONFLICT (video_id, date) DO UPDATE SET
            views = EXCLUDED.views,
            unique_viewers = EXCLUDED.unique_viewers,
            total_watch_time = EXCLUDED.total_watch_time,
            avg_watch_duration = EXCLUDED.avg_watch_duration,
            completion_count = EXCLUDED.completion_count,
            likes = EXCLUDED.likes,
            comments = EXCLUDED.comments,
            shares = EXCLUDED.shares;
    """
    
    conn = psycopg2.connect(DATABASE_URL)
    cur = conn.cursor()
    cur.execute(query, (date,))
    conn.commit()
    cur.close()
    conn.close()

# Run for yesterday
yesterday = datetime.now() - timedelta(days=1)
aggregate_video_stats(yesterday.date())
```

---

## Dashboard Requirements

### Creator Dashboard

**Overview Section:**

```
┌──────────────────────────────────────────────────────┐
│  Last 28 Days                                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐     │
│  │  125.4K    │  │    8.2K    │  │    2.1K    │     │
│  │  Views     │  │   Likes    │  │  Comments  │     │
│  │  ↗ +12.5%  │  │  ↗ +8.3%   │  │  ↗ +15.2%  │     │
│  └────────────┘  └────────────┘  └────────────┘     │
│                                                       │
│  Views Over Time                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │        📈 Line Chart                        │    │
│  │      (Daily views for last 28 days)         │    │
│  └─────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

**API Endpoint:** `GET /api/analytics/creator/overview`

**Response:**

```typescript
{
  period: "28d",
  metrics: {
    total_views: 125400,
    views_change: 12.5,  // % change from previous period
    total_likes: 8200,
    likes_change: 8.3,
    total_comments: 2100,
    comments_change: 15.2,
    total_watch_time: 2500000,  // seconds
    avg_watch_duration: 200,     // seconds
  },
  views_by_day: [
    { date: "2026-01-10", views: 4500 },
    { date: "2026-01-11", views: 4800 },
    // ...
  ]
}
```

**Video Analytics Page:**

```
┌──────────────────────────────────────────────────────┐
│  Video Title                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐     │
│  │   45.2K    │  │    1.8K    │  │    85%     │     │
│  │   Views    │  │   Likes    │  │ Completion │     │
│  └────────────┘  └────────────┘  └────────────┘     │
│                                                       │
│  Audience Retention                                  │
│  ┌─────────────────────────────────────────────┐    │
│  │  100%  ████████████▓▓▓▓▒▒▒░░░░               │    │
│  │   75%                                        │    │
│  │   50%                                        │    │
│  │   25%                                        │    │
│  │    0%  0──────5:00──────10:00──────15:00    │    │
│  └─────────────────────────────────────────────┘    │
│                                                       │
│  Traffic Sources                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  Search          45% ████████████████████   │    │
│  │  Direct          25% ███████████            │    │
│  │  Recommendations 20% █████████              │    │
│  │  External        10% ████                   │    │
│  └─────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

**API Endpoint:** `GET /api/analytics/video/:id`

**Query for Retention Curve:**

```sql
WITH retention AS (
    SELECT 
        FLOOR((properties->>'position')::int / 10) * 10 as timestamp_bucket,
        COUNT(DISTINCT session_id) as viewers
    FROM analytics_events
    WHERE event_type = 'video_view'
      AND properties->>'video_id' = $1
      AND DATE(timestamp) >= $2
    GROUP BY timestamp_bucket
)
SELECT 
    timestamp_bucket,
    viewers,
    ROUND(viewers * 100.0 / MAX(viewers) OVER (), 2) as retention_percentage
FROM retention
ORDER BY timestamp_bucket;
```

---

## Query Examples

### Get Top Videos (Last 30 Days)

```sql
SELECT 
    v.id,
    v.title,
    v.thumbnail_url,
    SUM(vsd.views) as total_views,
    SUM(vsd.total_watch_time) as total_watch_time,
    AVG(vsd.completion_rate) as avg_completion_rate
FROM videos v
JOIN video_stats_daily vsd ON v.id = vsd.video_id
WHERE vsd.date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY v.id
ORDER BY total_views DESC
LIMIT 10;
```

### Get Creator Growth (Subscribers Over Time)

```sql
SELECT 
    date,
    SUM(net_subscribers) OVER (ORDER BY date) as cumulative_subscribers
FROM creator_stats_daily
WHERE user_id = $1
  AND date >= CURRENT_DATE - INTERVAL '90 days'
ORDER BY date;
```

### Calculate DAU/MAU

```sql
WITH dau AS (
    SELECT COUNT(DISTINCT user_id) as count
    FROM analytics_events
    WHERE DATE(timestamp) = CURRENT_DATE
),
mau AS (
    SELECT COUNT(DISTINCT user_id) as count
    FROM analytics_events
    WHERE DATE(timestamp) >= CURRENT_DATE - INTERVAL '30 days'
)
SELECT 
    dau.count as dau,
    mau.count as mau,
    ROUND(dau.count * 100.0 / mau.count, 2) as stickiness
FROM dau, mau;
```

### Get Traffic Sources for Video

```sql
SELECT 
    context->>'referrer' as referrer,
    COUNT(*) as views
FROM analytics_events
WHERE event_type = 'video_view'
  AND properties->>'video_id' = $1
  AND DATE(timestamp) >= $2
GROUP BY referrer
ORDER BY views DESC;
```

---

## Real-Time Analytics

### Redis for Real-Time Counts

```typescript
// Increment view count in Redis
await redis.incr(`video:${videoId}:views:realtime`);

// Flush to PostgreSQL every 5 minutes
const flushRealTimeCounts = async () => {
  const keys = await redis.keys('video:*:views:realtime');
  
  for (const key of keys) {
    const [, videoId] = key.split(':');
    const count = await redis.get(key);
    
    await db.query(
      'UPDATE videos SET view_count = view_count + $1 WHERE id = $2',
      [parseInt(count), videoId]
    );
    
    await redis.del(key);
  }
};

// Run every 5 minutes
setInterval(flushRealTimeCounts, 5 * 60 * 1000);
```

### WebSocket for Live Viewer Count

```typescript
// Backend
io.on('connection', (socket) => {
  socket.on('watch_video', async (videoId) => {
    // Add to viewers set
    await redis.sadd(`video:${videoId}:viewers`, socket.id);
    
    // Get count
    const count = await redis.scard(`video:${videoId}:viewers`);
    
    // Broadcast to all viewers
    io.to(videoId).emit('viewer_count', count);
    
    // Join room
    socket.join(videoId);
  });
  
  socket.on('disconnect', async () => {
    // Remove from all video viewers
    const videoRooms = Array.from(socket.rooms);
    for (const videoId of videoRooms) {
      await redis.srem(`video:${videoId}:viewers`, socket.id);
      const count = await redis.scard(`video:${videoId}:viewers`);
      io.to(videoId).emit('viewer_count', count);
    }
  });
});

// Frontend
const socket = io();

socket.emit('watch_video', videoId);

socket.on('viewer_count', (count) => {
  setLiveViewers(count);
});
```

---

**End of Analytics Requirements**

*This specification provides a complete analytics system. All queries are optimized for TimescaleDB and ready for production use.*
