# StreamHub - Component Specification

**Version:** 1.0  
**Framework:** React 18+ (Next.js 14)  
**State Management:** Zustand  
**Styling:** Tailwind CSS  
**Last Updated:** February 2026

---

## Table of Contents

1. [Component Architecture](#component-architecture)
2. [Core Components](#core-components)
3. [Layout Components](#layout-components)
4. [Video Components](#video-components)
5. [Form Components](#form-components)
6. [Feedback Components](#feedback-components)
7. [State Management](#state-management)
8. [Hooks](#hooks)

---

## Component Architecture

### File Structure

```
src/
├── components/
│   ├── layout/
│   │   ├── Navbar.tsx
│   │   ├── Sidebar.tsx
│   │   └── Footer.tsx
│   ├── video/
│   │   ├── VideoCard.tsx
│   │   ├── VideoPlayer.tsx
│   │   ├── VideoGrid.tsx
│   │   └── VideoUploader.tsx
│   ├── ui/
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Modal.tsx
│   │   └── Toast.tsx
│   └── features/
│       ├── auth/
│       ├── comments/
│       └── dashboard/
├── hooks/
│   ├── useAuth.ts
│   ├── useVideo.ts
│   └── useToast.ts
├── store/
│   ├── authStore.ts
│   ├── videoStore.ts
│   └── uiStore.ts
└── lib/
    ├── api.ts
    └── utils.ts
```

---

## Core Components

### Button

**Purpose:** Reusable button component with variants

**File:** `src/components/ui/Button.tsx`

**Props Interface:**

```typescript
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  loading?: boolean;
  fullWidth?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  onClick?: () => void;
  type?: 'button' | 'submit' | 'reset';
  children: React.ReactNode;
  className?: string;
}
```

**Implementation:**

```tsx
import { ButtonHTMLAttributes, forwardRef } from 'react';
import { cn } from '@/lib/utils';
import { Loader2 } from 'lucide-react';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
  fullWidth?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
}

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ 
    variant = 'primary',
    size = 'md',
    loading = false,
    fullWidth = false,
    leftIcon,
    rightIcon,
    disabled,
    className,
    children,
    ...props
  }, ref) => {
    const baseStyles = 'inline-flex items-center justify-center font-semibold rounded-lg transition-all focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed';
    
    const variants = {
      primary: 'bg-indigo-600 text-white hover:bg-indigo-700 focus:ring-indigo-500',
      secondary: 'bg-transparent text-gray-300 border border-gray-600 hover:bg-gray-800 focus:ring-gray-500',
      ghost: 'bg-transparent text-gray-300 hover:bg-gray-800 focus:ring-gray-500',
      danger: 'bg-red-600 text-white hover:bg-red-700 focus:ring-red-500',
    };
    
    const sizes = {
      sm: 'px-3 py-1.5 text-sm',
      md: 'px-4 py-2 text-base',
      lg: 'px-6 py-3 text-lg',
    };
    
    return (
      <button
        ref={ref}
        disabled={disabled || loading}
        className={cn(
          baseStyles,
          variants[variant],
          sizes[size],
          fullWidth && 'w-full',
          className
        )}
        {...props}
      >
        {loading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
        {!loading && leftIcon && <span className="mr-2">{leftIcon}</span>}
        {children}
        {!loading && rightIcon && <span className="ml-2">{rightIcon}</span>}
      </button>
    );
  }
);

Button.displayName = 'Button';

export default Button;
```

**Usage:**

```tsx
<Button variant="primary" onClick={handleSubmit}>
  Submit
</Button>

<Button variant="secondary" size="sm" leftIcon={<PlusIcon />}>
  Add Video
</Button>

<Button loading={isLoading} disabled={!isValid}>
  Upload
</Button>
```

---

### Input

**Purpose:** Text input field with label and error state

**Props Interface:**

```typescript
interface InputProps {
  label?: string;
  placeholder?: string;
  type?: 'text' | 'email' | 'password' | 'number' | 'tel';
  value: string;
  onChange: (e: React.ChangeEvent<HTMLInputElement>) => void;
  error?: string;
  required?: boolean;
  disabled?: boolean;
  maxLength?: number;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  className?: string;
}
```

**Implementation:**

```tsx
import { InputHTMLAttributes, forwardRef } from 'react';
import { cn } from '@/lib/utils';

interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
}

const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ 
    label,
    error,
    leftIcon,
    rightIcon,
    className,
    ...props
  }, ref) => {
    return (
      <div className="w-full">
        {label && (
          <label className="block text-sm font-medium text-gray-300 mb-1">
            {label}
            {props.required && <span className="text-red-500 ml-1">*</span>}
          </label>
        )}
        
        <div className="relative">
          {leftIcon && (
            <div className="absolute left-3 top-1/2 -translate-y-1/2 text-gray-500">
              {leftIcon}
            </div>
          )}
          
          <input
            ref={ref}
            className={cn(
              'w-full px-4 py-2 bg-gray-800 border rounded-lg text-white placeholder-gray-500 transition-all',
              'focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-transparent',
              'disabled:opacity-50 disabled:cursor-not-allowed',
              error && 'border-red-500 focus:ring-red-500',
              !error && 'border-gray-600',
              leftIcon && 'pl-10',
              rightIcon && 'pr-10',
              className
            )}
            {...props}
          />
          
          {rightIcon && (
            <div className="absolute right-3 top-1/2 -translate-y-1/2 text-gray-500">
              {rightIcon}
            </div>
          )}
        </div>
        
        {error && (
          <p className="mt-1 text-sm text-red-500">{error}</p>
        )}
      </div>
    );
  }
);

Input.displayName = 'Input';

export default Input;
```

**Usage:**

```tsx
<Input
  label="Email"
  type="email"
  placeholder="Enter your email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  error={errors.email}
  required
/>
```

---

### Modal

**Purpose:** Reusable modal/dialog component

**Props Interface:**

```typescript
interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title?: string;
  size?: 'sm' | 'md' | 'lg' | 'xl';
  children: React.ReactNode;
  footer?: React.ReactNode;
  closeOnClickOutside?: boolean;
  showCloseButton?: boolean;
}
```

**Implementation:**

```tsx
import { Fragment } from 'react';
import { Dialog, Transition } from '@headlessui/react';
import { X } from 'lucide-react';
import { cn } from '@/lib/utils';

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title?: string;
  size?: 'sm' | 'md' | 'lg' | 'xl';
  children: React.ReactNode;
  footer?: React.ReactNode;
  closeOnClickOutside?: boolean;
  showCloseButton?: boolean;
}

const Modal: React.FC<ModalProps> = ({
  isOpen,
  onClose,
  title,
  size = 'md',
  children,
  footer,
  closeOnClickOutside = true,
  showCloseButton = true,
}) => {
  const sizes = {
    sm: 'max-w-md',
    md: 'max-w-lg',
    lg: 'max-w-2xl',
    xl: 'max-w-4xl',
  };

  return (
    <Transition show={isOpen} as={Fragment}>
      <Dialog
        onClose={closeOnClickOutside ? onClose : () => {}}
        className="relative z-50"
      >
        {/* Backdrop */}
        <Transition.Child
          as={Fragment}
          enter="ease-out duration-300"
          enterFrom="opacity-0"
          enterTo="opacity-100"
          leave="ease-in duration-200"
          leaveFrom="opacity-100"
          leaveTo="opacity-0"
        >
          <div className="fixed inset-0 bg-black/70" aria-hidden="true" />
        </Transition.Child>

        {/* Modal */}
        <div className="fixed inset-0 flex items-center justify-center p-4">
          <Transition.Child
            as={Fragment}
            enter="ease-out duration-300"
            enterFrom="opacity-0 scale-95"
            enterTo="opacity-100 scale-100"
            leave="ease-in duration-200"
            leaveFrom="opacity-100 scale-100"
            leaveTo="opacity-0 scale-95"
          >
            <Dialog.Panel
              className={cn(
                'w-full bg-gray-900 rounded-lg shadow-xl',
                sizes[size]
              )}
            >
              {/* Header */}
              {(title || showCloseButton) && (
                <div className="flex items-center justify-between p-6 border-b border-gray-800">
                  {title && (
                    <Dialog.Title className="text-xl font-semibold text-white">
                      {title}
                    </Dialog.Title>
                  )}
                  {showCloseButton && (
                    <button
                      onClick={onClose}
                      className="text-gray-400 hover:text-white transition-colors"
                    >
                      <X size={24} />
                    </button>
                  )}
                </div>
              )}

              {/* Content */}
              <div className="p-6">{children}</div>

              {/* Footer */}
              {footer && (
                <div className="p-6 border-t border-gray-800">{footer}</div>
              )}
            </Dialog.Panel>
          </Transition.Child>
        </div>
      </Dialog>
    </Transition>
  );
};

export default Modal;
```

**Usage:**

```tsx
<Modal
  isOpen={isOpen}
  onClose={() => setIsOpen(false)}
  title="Upload Video"
  size="lg"
  footer={
    <>
      <Button variant="secondary" onClick={() => setIsOpen(false)}>
        Cancel
      </Button>
      <Button variant="primary" onClick={handleUpload}>
        Upload
      </Button>
    </>
  }
>
  <VideoUploadForm />
</Modal>
```

---

## Video Components

### VideoCard

**Purpose:** Display video thumbnail with metadata

**Props Interface:**

```typescript
interface VideoCardProps {
  video: {
    id: string;
    title: string;
    thumbnail_url: string;
    duration: number;
    view_count: number;
    uploaded_at: string;
    creator: {
      id: string;
      username: string;
      avatar_url: string | null;
    };
  };
  onClick?: () => void;
  showCreator?: boolean;
}
```

**Implementation:**

```tsx
import Image from 'next/image';
import Link from 'next/link';
import { formatDuration, formatViews, formatTimeAgo } from '@/lib/utils';

interface VideoCardProps {
  video: {
    id: string;
    title: string;
    thumbnail_url: string;
    duration: number;
    view_count: number;
    uploaded_at: string;
    creator: {
      id: string;
      username: string;
      avatar_url: string | null;
    };
  };
  onClick?: () => void;
  showCreator?: boolean;
}

const VideoCard: React.FC<VideoCardProps> = ({ 
  video, 
  onClick,
  showCreator = true 
}) => {
  return (
    <Link
      href={`/watch/${video.id}`}
      onClick={onClick}
      className="group cursor-pointer"
    >
      {/* Thumbnail */}
      <div className="relative aspect-video bg-gray-800 rounded-lg overflow-hidden">
        <Image
          src={video.thumbnail_url}
          alt={video.title}
          fill
          className="object-cover group-hover:scale-105 transition-transform duration-300"
        />
        
        {/* Duration badge */}
        <div className="absolute bottom-2 right-2 px-2 py-1 bg-black/80 text-white text-xs font-semibold rounded">
          {formatDuration(video.duration)}
        </div>
        
        {/* Hover overlay */}
        <div className="absolute inset-0 bg-black/0 group-hover:bg-black/20 transition-colors" />
      </div>
      
      {/* Info */}
      <div className="mt-3">
        {showCreator && (
          <div className="flex gap-3">
            {/* Creator avatar */}
            <div className="flex-shrink-0">
              <div className="w-9 h-9 rounded-full bg-gray-700 overflow-hidden">
                {video.creator.avatar_url && (
                  <Image
                    src={video.creator.avatar_url}
                    alt={video.creator.username}
                    width={36}
                    height={36}
                  />
                )}
              </div>
            </div>
            
            <div className="flex-1 min-w-0">
              {/* Title */}
              <h3 className="text-white font-medium line-clamp-2 group-hover:text-indigo-400 transition-colors">
                {video.title}
              </h3>
              
              {/* Creator name */}
              <p className="text-sm text-gray-400 mt-1">
                {video.creator.username}
              </p>
              
              {/* Metadata */}
              <p className="text-sm text-gray-400">
                {formatViews(video.view_count)} views · {formatTimeAgo(video.uploaded_at)}
              </p>
            </div>
          </div>
        )}
        
        {!showCreator && (
          <>
            <h3 className="text-white font-medium line-clamp-2">
              {video.title}
            </h3>
            <p className="text-sm text-gray-400 mt-1">
              {formatViews(video.view_count)} views · {formatTimeAgo(video.uploaded_at)}
            </p>
          </>
        )}
      </div>
    </Link>
  );
};

export default VideoCard;
```

**Usage:**

```tsx
<VideoCard video={video} showCreator={true} />
```

---

### VideoPlayer

**Purpose:** Custom video player with controls

**Props Interface:**

```typescript
interface VideoPlayerProps {
  videoUrl: string;
  thumbnailUrl?: string;
  title: string;
  onTimeUpdate?: (currentTime: number) => void;
  onEnded?: () => void;
  autoPlay?: boolean;
}
```

**Implementation:**

```tsx
'use client';

import { useRef, useState, useEffect } from 'react';
import { Play, Pause, Volume2, VolumeX, Maximize, Settings } from 'lucide-react';
import { formatDuration } from '@/lib/utils';

interface VideoPlayerProps {
  videoUrl: string;
  thumbnailUrl?: string;
  title: string;
  onTimeUpdate?: (currentTime: number) => void;
  onEnded?: () => void;
  autoPlay?: boolean;
}

const VideoPlayer: React.FC<VideoPlayerProps> = ({
  videoUrl,
  thumbnailUrl,
  title,
  onTimeUpdate,
  onEnded,
  autoPlay = false,
}) => {
  const videoRef = useRef<HTMLVideoElement>(null);
  const [isPlaying, setIsPlaying] = useState(autoPlay);
  const [currentTime, setCurrentTime] = useState(0);
  const [duration, setDuration] = useState(0);
  const [volume, setVolume] = useState(1);
  const [isMuted, setIsMuted] = useState(false);
  const [showControls, setShowControls] = useState(true);

  const togglePlay = () => {
    if (videoRef.current) {
      if (isPlaying) {
        videoRef.current.pause();
      } else {
        videoRef.current.play();
      }
      setIsPlaying(!isPlaying);
    }
  };

  const toggleMute = () => {
    if (videoRef.current) {
      videoRef.current.muted = !isMuted;
      setIsMuted(!isMuted);
    }
  };

  const handleSeek = (e: React.ChangeEvent<HTMLInputElement>) => {
    const time = parseFloat(e.target.value);
    if (videoRef.current) {
      videoRef.current.currentTime = time;
      setCurrentTime(time);
    }
  };

  const handleVolumeChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const vol = parseFloat(e.target.value);
    if (videoRef.current) {
      videoRef.current.volume = vol;
      setVolume(vol);
      setIsMuted(vol === 0);
    }
  };

  const toggleFullscreen = () => {
    if (videoRef.current) {
      if (document.fullscreenElement) {
        document.exitFullscreen();
      } else {
        videoRef.current.requestFullscreen();
      }
    }
  };

  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    const handleLoadedMetadata = () => {
      setDuration(video.duration);
    };

    const handleTimeUpdate = () => {
      setCurrentTime(video.currentTime);
      onTimeUpdate?.(video.currentTime);
    };

    const handleEnded = () => {
      setIsPlaying(false);
      onEnded?.();
    };

    video.addEventListener('loadedmetadata', handleLoadedMetadata);
    video.addEventListener('timeupdate', handleTimeUpdate);
    video.addEventListener('ended', handleEnded);

    return () => {
      video.removeEventListener('loadedmetadata', handleLoadedMetadata);
      video.removeEventListener('timeupdate', handleTimeUpdate);
      video.removeEventListener('ended', handleEnded);
    };
  }, [onTimeUpdate, onEnded]);

  return (
    <div 
      className="relative bg-black aspect-video rounded-lg overflow-hidden group"
      onMouseEnter={() => setShowControls(true)}
      onMouseLeave={() => setShowControls(false)}
    >
      <video
        ref={videoRef}
        src={videoUrl}
        poster={thumbnailUrl}
        className="w-full h-full"
        onClick={togglePlay}
        autoPlay={autoPlay}
      />

      {/* Controls */}
      <div 
        className={`absolute bottom-0 left-0 right-0 bg-gradient-to-t from-black/80 to-transparent p-4 transition-opacity ${
          showControls ? 'opacity-100' : 'opacity-0'
        }`}
      >
        {/* Progress bar */}
        <input
          type="range"
          min="0"
          max={duration || 0}
          value={currentTime}
          onChange={handleSeek}
          className="w-full mb-3 h-1 bg-gray-600 rounded-lg appearance-none cursor-pointer"
        />

        {/* Controls row */}
        <div className="flex items-center justify-between text-white">
          <div className="flex items-center gap-4">
            {/* Play/Pause */}
            <button onClick={togglePlay} className="hover:text-indigo-400 transition-colors">
              {isPlaying ? <Pause size={24} /> : <Play size={24} />}
            </button>

            {/* Volume */}
            <div className="flex items-center gap-2 group/volume">
              <button onClick={toggleMute} className="hover:text-indigo-400 transition-colors">
                {isMuted ? <VolumeX size={20} /> : <Volume2 size={20} />}
              </button>
              <input
                type="range"
                min="0"
                max="1"
                step="0.1"
                value={volume}
                onChange={handleVolumeChange}
                className="w-20 h-1 bg-gray-600 rounded-lg appearance-none cursor-pointer opacity-0 group-hover/volume:opacity-100 transition-opacity"
              />
            </div>

            {/* Time */}
            <span className="text-sm">
              {formatDuration(currentTime)} / {formatDuration(duration)}
            </span>
          </div>

          <div className="flex items-center gap-4">
            {/* Settings */}
            <button className="hover:text-indigo-400 transition-colors">
              <Settings size={20} />
            </button>

            {/* Fullscreen */}
            <button onClick={toggleFullscreen} className="hover:text-indigo-400 transition-colors">
              <Maximize size={20} />
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};

export default VideoPlayer;
```

**Usage:**

```tsx
<VideoPlayer
  videoUrl={video.video_url}
  thumbnailUrl={video.thumbnail_url}
  title={video.title}
  autoPlay={true}
  onTimeUpdate={(time) => console.log('Current time:', time)}
  onEnded={() => console.log('Video ended')}
/>
```

---

### VideoGrid

**Purpose:** Responsive grid of video cards

**Props Interface:**

```typescript
interface VideoGridProps {
  videos: Video[];
  loading?: boolean;
  columns?: 1 | 2 | 3 | 4;
  gap?: number;
  onVideoClick?: (videoId: string) => void;
}
```

**Implementation:**

```tsx
import VideoCard from './VideoCard';
import { Video } from '@/types';

interface VideoGridProps {
  videos: Video[];
  loading?: boolean;
  columns?: 1 | 2 | 3 | 4;
  gap?: number;
  onVideoClick?: (videoId: string) => void;
}

const VideoGrid: React.FC<VideoGridProps> = ({
  videos,
  loading = false,
  columns = 3,
  gap = 6,
  onVideoClick,
}) => {
  const gridCols = {
    1: 'grid-cols-1',
    2: 'grid-cols-1 md:grid-cols-2',
    3: 'grid-cols-1 md:grid-cols-2 lg:grid-cols-3',
    4: 'grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4',
  };

  if (loading) {
    return (
      <div className={`grid ${gridCols[columns]} gap-${gap}`}>
        {Array.from({ length: 12 }).map((_, i) => (
          <VideoCardSkeleton key={i} />
        ))}
      </div>
    );
  }

  if (videos.length === 0) {
    return (
      <div className="text-center py-12">
        <p className="text-gray-400">No videos found</p>
      </div>
    );
  }

  return (
    <div className={`grid ${gridCols[columns]} gap-${gap}`}>
      {videos.map((video) => (
        <VideoCard
          key={video.id}
          video={video}
          onClick={() => onVideoClick?.(video.id)}
        />
      ))}
    </div>
  );
};

const VideoCardSkeleton = () => (
  <div className="animate-pulse">
    <div className="aspect-video bg-gray-800 rounded-lg" />
    <div className="mt-3 flex gap-3">
      <div className="w-9 h-9 bg-gray-800 rounded-full" />
      <div className="flex-1 space-y-2">
        <div className="h-4 bg-gray-800 rounded w-3/4" />
        <div className="h-3 bg-gray-800 rounded w-1/2" />
      </div>
    </div>
  </div>
);

export default VideoGrid;
```

---

## State Management

### Auth Store (Zustand)

**File:** `src/store/authStore.ts`

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface User {
  id: string;
  username: string;
  email: string;
  role: 'viewer' | 'creator' | 'admin';
  avatar_url: string | null;
}

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  
  // Actions
  login: (token: string, user: User) => void;
  logout: () => void;
  updateUser: (user: Partial<User>) => void;
  setLoading: (loading: boolean) => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      isAuthenticated: false,
      isLoading: false,
      
      login: (token, user) => set({ 
        token, 
        user, 
        isAuthenticated: true 
      }),
      
      logout: () => set({ 
        token: null, 
        user: null, 
        isAuthenticated: false 
      }),
      
      updateUser: (updatedUser) => set((state) => ({
        user: state.user ? { ...state.user, ...updatedUser } : null
      })),
      
      setLoading: (loading) => set({ isLoading: loading }),
    }),
    {
      name: 'auth-storage',
    }
  )
);
```

---

### Video Store

**File:** `src/store/videoStore.ts`

```typescript
import { create } from 'zustand';
import { Video } from '@/types';

interface VideoState {
  videos: Video[];
  currentVideo: Video | null;
  loading: boolean;
  error: string | null;
  
  // Actions
  setVideos: (videos: Video[]) => void;
  addVideo: (video: Video) => void;
  setCurrentVideo: (video: Video | null) => void;
  updateVideo: (id: string, updates: Partial<Video>) => void;
  removeVideo: (id: string) => void;
  setLoading: (loading: boolean) => void;
  setError: (error: string | null) => void;
}

export const useVideoStore = create<VideoState>((set) => ({
  videos: [],
  currentVideo: null,
  loading: false,
  error: null,
  
  setVideos: (videos) => set({ videos }),
  
  addVideo: (video) => set((state) => ({
    videos: [video, ...state.videos]
  })),
  
  setCurrentVideo: (video) => set({ currentVideo: video }),
  
  updateVideo: (id, updates) => set((state) => ({
    videos: state.videos.map(v => 
      v.id === id ? { ...v, ...updates } : v
    ),
    currentVideo: state.currentVideo?.id === id 
      ? { ...state.currentVideo, ...updates }
      : state.currentVideo
  })),
  
  removeVideo: (id) => set((state) => ({
    videos: state.videos.filter(v => v.id !== id),
    currentVideo: state.currentVideo?.id === id ? null : state.currentVideo
  })),
  
  setLoading: (loading) => set({ loading }),
  setError: (error) => set({ error }),
}));
```

---

## Hooks

### useAuth

**File:** `src/hooks/useAuth.ts`

```typescript
import { useAuthStore } from '@/store/authStore';
import { useRouter } from 'next/navigation';
import api from '@/lib/api';

export const useAuth = () => {
  const router = useRouter();
  const { user, token, isAuthenticated, login, logout, setLoading } = useAuthStore();

  const loginUser = async (email: string, password: string) => {
    setLoading(true);
    try {
      const response = await api.post('/auth/login', { email, password });
      const { token, user } = response.data.data;
      login(token, user);
      router.push('/');
    } catch (error) {
      throw error;
    } finally {
      setLoading(false);
    }
  };

  const logoutUser = async () => {
    try {
      await api.post('/auth/logout');
    } catch (error) {
      console.error('Logout error:', error);
    } finally {
      logout();
      router.push('/login');
    }
  };

  const requireAuth = () => {
    if (!isAuthenticated) {
      router.push('/login');
      return false;
    }
    return true;
  };

  return {
    user,
    token,
    isAuthenticated,
    loginUser,
    logoutUser,
    requireAuth,
  };
};
```

---

### useVideo

**File:** `src/hooks/useVideo.ts`

```typescript
import { useState } from 'react';
import api from '@/lib/api';
import { Video } from '@/types';
import { useVideoStore } from '@/store/videoStore';

export const useVideo = () => {
  const { setVideos, setCurrentVideo, setLoading, setError } = useVideoStore();
  const [uploading, setUploading] = useState(false);

  const fetchVideos = async (params?: { 
    page?: number; 
    limit?: number; 
    category?: string;
    sort?: string;
  }) => {
    setLoading(true);
    try {
      const response = await api.get('/videos', { params });
      setVideos(response.data.data);
      return response.data;
    } catch (error) {
      setError('Failed to fetch videos');
      throw error;
    } finally {
      setLoading(false);
    }
  };

  const fetchVideo = async (id: string) => {
    setLoading(true);
    try {
      const response = await api.get(`/videos/${id}`);
      setCurrentVideo(response.data.data);
      return response.data.data;
    } catch (error) {
      setError('Failed to fetch video');
      throw error;
    } finally {
      setLoading(false);
    }
  };

  const uploadVideo = async (file: File, metadata: {
    title: string;
    description?: string;
    tags?: string[];
    category?: string;
    visibility: string;
  }) => {
    setUploading(true);
    try {
      // Get upload URL
      const uploadUrlResponse = await api.post('/videos/upload-url', {
        filename: file.name,
        file_size: file.size,
        content_type: file.type,
      });
      
      const { upload_url, video_id } = uploadUrlResponse.data.data;

      // Upload to R2
      await fetch(upload_url, {
        method: 'PUT',
        body: file,
        headers: {
          'Content-Type': file.type,
        },
      });

      // Create video metadata
      const videoResponse = await api.post('/videos', {
        video_id,
        ...metadata,
      });

      return videoResponse.data.data;
    } catch (error) {
      throw error;
    } finally {
      setUploading(false);
    }
  };

  const likeVideo = async (videoId: string) => {
    try {
      await api.post(`/videos/${videoId}/like`);
    } catch (error) {
      throw error;
    }
  };

  const unlikeVideo = async (videoId: string) => {
    try {
      await api.delete(`/videos/${videoId}/like`);
    } catch (error) {
      throw error;
    }
  };

  return {
    fetchVideos,
    fetchVideo,
    uploadVideo,
    likeVideo,
    unlikeVideo,
    uploading,
  };
};
```

---

### useToast

**File:** `src/hooks/useToast.ts`

```typescript
import { create } from 'zustand';

interface Toast {
  id: string;
  type: 'success' | 'error' | 'warning' | 'info';
  message: string;
  duration?: number;
}

interface ToastStore {
  toasts: Toast[];
  addToast: (toast: Omit<Toast, 'id'>) => void;
  removeToast: (id: string) => void;
}

const useToastStore = create<ToastStore>((set) => ({
  toasts: [],
  
  addToast: (toast) => {
    const id = Math.random().toString(36).substring(7);
    set((state) => ({
      toasts: [...state.toasts, { ...toast, id }]
    }));
    
    // Auto-remove after duration
    setTimeout(() => {
      set((state) => ({
        toasts: state.toasts.filter((t) => t.id !== id)
      }));
    }, toast.duration || 3000);
  },
  
  removeToast: (id) => set((state) => ({
    toasts: state.toasts.filter((t) => t.id !== id)
  })),
}));

export const useToast = () => {
  const { addToast } = useToastStore();

  const toast = {
    success: (message: string) => addToast({ type: 'success', message }),
    error: (message: string) => addToast({ type: 'error', message }),
    warning: (message: string) => addToast({ type: 'warning', message }),
    info: (message: string) => addToast({ type: 'info', message }),
  };

  return toast;
};

export const ToastContainer = () => {
  const { toasts, removeToast } = useToastStore();

  return (
    <div className="fixed top-4 right-4 z-50 space-y-2">
      {toasts.map((toast) => (
        <div
          key={toast.id}
          className={`px-4 py-3 rounded-lg shadow-lg text-white ${
            toast.type === 'success' ? 'bg-green-600' :
            toast.type === 'error' ? 'bg-red-600' :
            toast.type === 'warning' ? 'bg-yellow-600' :
            'bg-blue-600'
          }`}
          onClick={() => removeToast(toast.id)}
        >
          {toast.message}
        </div>
      ))}
    </div>
  );
};
```

---

**End of Component Specification**

*Use these component specs to generate consistent, type-safe React components. All props are fully typed with TypeScript.*
