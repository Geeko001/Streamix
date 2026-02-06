# StreamHub - Code Style & Patterns Guide

**Version:** 1.0  
**Last Updated:** February 2026

---

## Table of Contents

1. [General Principles](#general-principles)
2. [TypeScript/JavaScript](#typescriptjavascript)
3. [React/Next.js](#reactnextjs)
4. [File & Folder Structure](#file--folder-structure)
5. [Naming Conventions](#naming-conventions)
6. [Code Patterns](#code-patterns)
7. [Testing Standards](#testing-standards)
8. [Git Workflow](#git-workflow)

---

## General Principles

### Code Philosophy

1. **Clarity over Cleverness** - Write code that's easy to understand
2. **DRY (Don't Repeat Yourself)** - Extract reusable logic
3. **KISS (Keep It Simple, Stupid)** - Avoid over-engineering
4. **YAGNI (You Aren't Gonna Need It)** - Don't build for hypothetical future
5. **Separation of Concerns** - Each module should have one responsibility

### Code Quality Rules

- ✅ **DO** write self-documenting code with clear variable names
- ✅ **DO** add comments for "why", not "what"
- ✅ **DO** keep functions small (<50 lines)
- ✅ **DO** handle errors explicitly
- ❌ **DON'T** use magic numbers (use named constants)
- ❌ **DON'T** nest more than 3 levels deep
- ❌ **DON'T** mutate props or state directly

---

## TypeScript/JavaScript

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "preserve"
  }
}
```

### Type Definitions

**Always define interfaces/types:**

```typescript
// ✅ GOOD - Explicit types
interface User {
  id: string;
  username: string;
  email: string;
  role: 'viewer' | 'creator' | 'admin';
  avatar_url: string | null;
}

const user: User = {
  id: '123',
  username: 'johndoe',
  email: 'john@example.com',
  role: 'creator',
  avatar_url: null,
};

// ❌ BAD - No types
const user = {
  id: '123',
  username: 'johndoe',
  // ...
};
```

**Use union types for variants:**

```typescript
// ✅ GOOD
type ButtonVariant = 'primary' | 'secondary' | 'ghost' | 'danger';

interface ButtonProps {
  variant: ButtonVariant;
}

// ❌ BAD
interface ButtonProps {
  variant: string; // Too vague
}
```

### Variable Declarations

```typescript
// ✅ GOOD - const by default
const apiUrl = 'https://api.streamhub.com';
const maxUploadSize = 2 * 1024 * 1024 * 1024; // 2GB

// ✅ GOOD - let when reassignment needed
let currentPage = 1;
let isLoading = false;

// ❌ BAD - var is forbidden
var count = 0;
```

### Function Declarations

```typescript
// ✅ GOOD - Arrow functions for consistency
const fetchVideos = async (params: FetchVideosParams): Promise<Video[]> => {
  const response = await api.get('/videos', { params });
  return response.data.data;
};

// ✅ GOOD - Named export
export const formatDuration = (seconds: number): string => {
  const minutes = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return `${minutes}:${secs.toString().padStart(2, '0')}`;
};

// ❌ BAD - Function keyword (use arrow functions)
function fetchVideos(params) {
  // ...
}
```

### Error Handling

```typescript
// ✅ GOOD - Explicit error handling
const uploadVideo = async (file: File) => {
  try {
    const response = await api.post('/videos/upload', formData);
    return response.data;
  } catch (error) {
    if (error instanceof AxiosError) {
      throw new Error(error.response?.data.error.message || 'Upload failed');
    }
    throw error;
  }
};

// ❌ BAD - Silent failures
const uploadVideo = async (file: File) => {
  try {
    const response = await api.post('/videos/upload', formData);
    return response.data;
  } catch (error) {
    // Silent failure - no handling
  }
};
```

### Async/Await

```typescript
// ✅ GOOD - Use async/await
const getData = async () => {
  const users = await fetchUsers();
  const videos = await fetchVideos();
  return { users, videos };
};

// ❌ BAD - Promise chaining
const getData = () => {
  return fetchUsers()
    .then(users => {
      return fetchVideos()
        .then(videos => ({ users, videos }));
    });
};
```

### Object/Array Operations

```typescript
// ✅ GOOD - Immutable operations
const addVideo = (videos: Video[], newVideo: Video) => {
  return [...videos, newVideo];
};

const updateVideo = (videos: Video[], id: string, updates: Partial<Video>) => {
  return videos.map(video => 
    video.id === id ? { ...video, ...updates } : video
  );
};

// ❌ BAD - Mutation
const addVideo = (videos: Video[], newVideo: Video) => {
  videos.push(newVideo); // Mutates array
  return videos;
};
```

---

## React/Next.js

### Component Structure

```tsx
// ✅ GOOD - Consistent structure
import { useState, useEffect } from 'react';
import { useRouter } from 'next/navigation';
import Button from '@/components/ui/Button';
import { useAuth } from '@/hooks/useAuth';
import { Video } from '@/types';

interface VideoCardProps {
  video: Video;
  onClick?: () => void;
}

const VideoCard: React.FC<VideoCardProps> = ({ video, onClick }) => {
  // 1. Hooks
  const router = useRouter();
  const { isAuthenticated } = useAuth();
  
  // 2. State
  const [isHovered, setIsHovered] = useState(false);
  
  // 3. Effects
  useEffect(() => {
    // Effect logic
  }, []);
  
  // 4. Event handlers
  const handleClick = () => {
    onClick?.();
    router.push(`/watch/${video.id}`);
  };
  
  // 5. Render
  return (
    <div onClick={handleClick}>
      {/* JSX */}
    </div>
  );
};

export default VideoCard;
```

### Component Naming

```typescript
// ✅ GOOD - PascalCase for components
const VideoPlayer = () => { };
const UserProfile = () => { };
const SearchBar = () => { };

// ❌ BAD
const videoPlayer = () => { };
const user_profile = () => { };
```

### Props

```typescript
// ✅ GOOD - Explicit prop types
interface ButtonProps {
  variant: 'primary' | 'secondary';
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  onClick?: () => void;
  children: React.ReactNode;
}

const Button: React.FC<ButtonProps> = ({
  variant = 'primary',
  size = 'md',
  disabled = false,
  onClick,
  children,
}) => {
  // ...
};

// ❌ BAD - Prop drilling (use context/store instead)
<Component1 user={user} video={video} settings={settings} />
  <Component2 user={user} video={video} settings={settings} />
    <Component3 user={user} video={video} settings={settings} />
```

### State Management

```typescript
// ✅ GOOD - Zustand store
import { create } from 'zustand';

interface VideoStore {
  videos: Video[];
  setVideos: (videos: Video[]) => void;
  addVideo: (video: Video) => void;
}

export const useVideoStore = create<VideoStore>((set) => ({
  videos: [],
  setVideos: (videos) => set({ videos }),
  addVideo: (video) => set((state) => ({ videos: [video, ...state.videos] })),
}));

// Usage in component
const { videos, addVideo } = useVideoStore();

// ❌ BAD - Prop drilling for global state
// Pass state through 5+ component levels
```

### Conditional Rendering

```tsx
// ✅ GOOD - Early returns
const VideoPlayer = ({ video }) => {
  if (!video) {
    return <div>Loading...</div>;
  }
  
  if (video.status === 'processing') {
    return <ProcessingState />;
  }
  
  return <VideoPlayerUI video={video} />;
};

// ❌ BAD - Nested ternaries
const VideoPlayer = ({ video }) => {
  return (
    <div>
      {!video ? (
        <div>Loading...</div>
      ) : video.status === 'processing' ? (
        <ProcessingState />
      ) : (
        <VideoPlayerUI video={video} />
      )}
    </div>
  );
};
```

### Event Handlers

```tsx
// ✅ GOOD - Prefix with "handle"
const handleClick = () => { };
const handleSubmit = (e: FormEvent) => { };
const handleChange = (e: ChangeEvent<HTMLInputElement>) => { };

// ❌ BAD
const onClick = () => { };
const submit = () => { };
```

### useEffect

```typescript
// ✅ GOOD - Clear dependencies
useEffect(() => {
  fetchVideos(userId);
}, [userId]); // Only re-run when userId changes

// ✅ GOOD - Cleanup
useEffect(() => {
  const interval = setInterval(() => {
    fetchAnalytics();
  }, 5000);
  
  return () => clearInterval(interval);
}, []);

// ❌ BAD - Missing dependencies
useEffect(() => {
  fetchVideos(userId);
}, []); // userId is missing!

// ❌ BAD - No cleanup
useEffect(() => {
  setInterval(() => {
    fetchAnalytics();
  }, 5000);
  // Missing cleanup - memory leak!
}, []);
```

---

## File & Folder Structure

### Project Structure

```
streamhub/
├── src/
│   ├── app/                    # Next.js App Router
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── (main)/
│   │   │   ├── page.tsx        # Home page
│   │   │   ├── watch/[id]/
│   │   │   └── search/
│   │   ├── api/                # API routes
│   │   ├── layout.tsx
│   │   └── globals.css
│   │
│   ├── components/
│   │   ├── ui/                 # Reusable UI components
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   └── Modal.tsx
│   │   ├── layout/             # Layout components
│   │   │   ├── Navbar.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   └── Footer.tsx
│   │   ├── video/              # Video-specific
│   │   │   ├── VideoCard.tsx
│   │   │   ├── VideoPlayer.tsx
│   │   │   └── VideoGrid.tsx
│   │   └── features/           # Feature-specific
│   │       ├── auth/
│   │       ├── comments/
│   │       └── dashboard/
│   │
│   ├── hooks/                  # Custom hooks
│   │   ├── useAuth.ts
│   │   ├── useVideo.ts
│   │   └── useToast.ts
│   │
│   ├── store/                  # Zustand stores
│   │   ├── authStore.ts
│   │   ├── videoStore.ts
│   │   └── uiStore.ts
│   │
│   ├── lib/                    # Utilities
│   │   ├── api.ts              # API client
│   │   ├── utils.ts            # Helper functions
│   │   └── constants.ts
│   │
│   ├── types/                  # TypeScript types
│   │   ├── video.ts
│   │   ├── user.ts
│   │   └── index.ts
│   │
│   └── styles/                 # Global styles
│       └── globals.css
│
├── public/                     # Static assets
│   ├── images/
│   └── icons/
│
├── tests/                      # Tests
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
└── package.json
```

### File Naming

```
✅ GOOD:
components/ui/Button.tsx          (PascalCase for components)
hooks/useAuth.ts                  (camelCase, prefix with "use")
store/authStore.ts                (camelCase, suffix with "Store")
lib/utils.ts                      (camelCase for utilities)
types/video.ts                    (camelCase for type files)

❌ BAD:
components/ui/button.tsx          (lowercase)
hooks/Auth.ts                     (no "use" prefix)
store/auth_store.ts               (snake_case)
```

---

## Naming Conventions

### Variables

```typescript
// ✅ GOOD - Descriptive camelCase
const isAuthenticated = true;
const userVideos = [];
const maxUploadSize = 2147483648;
const apiBaseUrl = 'https://api.streamhub.com';

// ❌ BAD
const auth = true;                // Too vague
const videos = [];                // Unclear whose videos
const MAX = 2147483648;           // Not descriptive
const url = 'https://...';        // Too generic
```

### Constants

```typescript
// ✅ GOOD - UPPERCASE for true constants
const MAX_FILE_SIZE = 2 * 1024 * 1024 * 1024;
const API_BASE_URL = 'https://api.streamhub.com';
const VIDEO_QUALITIES = ['360p', '480p', '720p', '1080p'] as const;

// ✅ GOOD - Config object
const CONFIG = {
  MAX_UPLOAD_SIZE: 2147483648,
  MAX_VIDEO_DURATION: 3600,
  SUPPORTED_FORMATS: ['video/mp4', 'video/webm'],
} as const;
```

### Functions

```typescript
// ✅ GOOD - Verbs, camelCase
const fetchVideos = () => { };
const uploadVideo = () => { };
const formatDuration = () => { };
const isValidEmail = () => { };
const hasPermission = () => { };

// ❌ BAD
const videos = () => { };         // Noun, not verb
const UploadVideo = () => { };    // PascalCase (that's for components)
const check_email = () => { };    // snake_case
```

### Components

```typescript
// ✅ GOOD - PascalCase, descriptive
const VideoCard = () => { };
const UserProfile = () => { };
const SearchBar = () => { };
const LoadingSpinner = () => { };

// ❌ BAD
const videocard = () => { };      // lowercase
const Card = () => { };           // Too generic
const UC = () => { };             // Abbreviation
```

### Booleans

```typescript
// ✅ GOOD - is/has/can prefix
const isLoading = true;
const hasError = false;
const canEdit = true;
const shouldAutoPlay = false;

// ❌ BAD
const loading = true;             // Not clear it's boolean
const error = false;              // Could be error object
```

---

## Code Patterns

### API Calls

```typescript
// ✅ GOOD - Centralized API client
// lib/api.ts
import axios from 'axios';
import { useAuthStore } from '@/store/authStore';

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
});

// Request interceptor (add auth token)
api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor (handle errors)
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout();
    }
    return Promise.reject(error);
  }
);

export default api;

// Usage in components/hooks
import api from '@/lib/api';

const fetchVideos = async () => {
  const response = await api.get('/videos');
  return response.data.data;
};
```

### Form Handling

```typescript
// ✅ GOOD - React Hook Form + Zod
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const videoSchema = z.object({
  title: z.string().min(3).max(200),
  description: z.string().max(5000).optional(),
  tags: z.array(z.string()).max(10).optional(),
  visibility: z.enum(['public', 'unlisted', 'private']),
});

type VideoFormData = z.infer<typeof videoSchema>;

const VideoForm = () => {
  const { register, handleSubmit, formState: { errors } } = useForm<VideoFormData>({
    resolver: zodResolver(videoSchema),
  });

  const onSubmit = (data: VideoFormData) => {
    // Handle form submission
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('title')} />
      {errors.title && <span>{errors.title.message}</span>}
      {/* ... */}
    </form>
  );
};
```

### Loading States

```typescript
// ✅ GOOD - Explicit loading states
const VideoList = () => {
  const [videos, setVideos] = useState<Video[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadVideos = async () => {
      try {
        setLoading(true);
        const data = await fetchVideos();
        setVideos(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };
    
    loadVideos();
  }, []);

  if (loading) return <Skeleton />;
  if (error) return <Error message={error} />;
  if (videos.length === 0) return <EmptyState />;
  
  return <VideoGrid videos={videos} />;
};
```

### Utility Functions

```typescript
// ✅ GOOD - Pure functions in lib/utils.ts
export const formatDuration = (seconds: number): string => {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = seconds % 60;
  
  if (hours > 0) {
    return `${hours}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  }
  return `${minutes}:${secs.toString().padStart(2, '0')}`;
};

export const formatViews = (views: number): string => {
  if (views >= 1000000) {
    return `${(views / 1000000).toFixed(1)}M`;
  }
  if (views >= 1000) {
    return `${(views / 1000).toFixed(1)}K`;
  }
  return views.toString();
};

export const formatTimeAgo = (date: string): string => {
  const now = new Date();
  const past = new Date(date);
  const diffInSeconds = Math.floor((now.getTime() - past.getTime()) / 1000);
  
  if (diffInSeconds < 60) return 'Just now';
  if (diffInSeconds < 3600) return `${Math.floor(diffInSeconds / 60)} minutes ago`;
  if (diffInSeconds < 86400) return `${Math.floor(diffInSeconds / 3600)} hours ago`;
  if (diffInSeconds < 604800) return `${Math.floor(diffInSeconds / 86400)} days ago`;
  return past.toLocaleDateString();
};
```

---

## Testing Standards

### Unit Tests

```typescript
// ✅ GOOD - Jest + React Testing Library
import { render, screen, fireEvent } from '@testing-library/react';
import Button from './Button';

describe('Button', () => {
  it('renders children correctly', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('calls onClick when clicked', () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);
    
    fireEvent.click(screen.getByText('Click me'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click me</Button>);
    expect(screen.getByText('Click me')).toBeDisabled();
  });
});
```

### E2E Tests

```typescript
// ✅ GOOD - Playwright
import { test, expect } from '@playwright/test';

test.describe('Video Upload', () => {
  test('should upload video successfully', async ({ page }) => {
    await page.goto('/');
    await page.click('text=Upload Video');
    
    await page.setInputFiles('input[type="file"]', 'test-video.mp4');
    await page.fill('input[name="title"]', 'Test Video');
    await page.click('button:has-text("Publish")');
    
    await expect(page.locator('text=Video uploaded successfully')).toBeVisible();
  });
});
```

---

## Git Workflow

### Commit Messages

```bash
# ✅ GOOD - Conventional Commits
feat: add video upload functionality
fix: resolve video player buffering issue
docs: update API documentation
style: format code with prettier
refactor: extract video player controls to separate component
test: add unit tests for video service
chore: update dependencies

# With scope
feat(auth): implement OAuth login
fix(player): fix fullscreen mode on iOS

# With body
feat: add video analytics dashboard

Implemented comprehensive analytics dashboard with:
- View count over time
- Audience demographics
- Traffic sources
- Retention curve

Closes #123

# ❌ BAD
Updated stuff
Fixed bug
WIP
asdf
```

### Branch Naming

```bash
# ✅ GOOD
feature/video-upload
feature/auth-oauth
fix/player-buffering
fix/login-validation
refactor/video-service
docs/api-documentation

# ❌ BAD
new-feature
bug-fix
john-changes
temp
```

### Pull Request Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests added/updated
- [ ] E2E tests added/updated
- [ ] Manually tested

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No new warnings
```

---

**End of Code Style Guide**

*Follow these conventions to maintain a consistent, high-quality codebase that's easy for AI assistants to understand and extend.*
