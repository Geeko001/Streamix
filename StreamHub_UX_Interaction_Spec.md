# StreamHub - UX & Interaction Specification

**Version:** 1.0  
**Last Updated:** February 2026  
**Status:** Draft

---

## Table of Contents

1. [Design Principles](#design-principles)
2. [User Flows](#user-flows)
3. [Screen Specifications](#screen-specifications)
4. [Component Library](#component-library)
5. [Interaction Patterns](#interaction-patterns)
6. [Responsive Design](#responsive-design)
7. [Accessibility](#accessibility)
8. [Micro-interactions](#micro-interactions)
9. [States & Feedback](#states--feedback)
10. [Content Guidelines](#content-guidelines)

---

## 1. Design Principles

### 1.1 Core Values

**Creator-First**
- Empower creators with powerful tools
- Make uploading & managing content effortless
- Provide transparent, actionable analytics

**Discovery-Focused**
- AI helps users find content they'll love
- Serendipity over algorithmic echo chambers
- Surface diverse, quality content

**Clean & Fast**
- Minimal UI, maximum functionality
- Fast page loads (<2s)
- Smooth, 60fps animations

**Accessible to All**
- WCAG 2.1 AA compliance
- Keyboard navigation support
- Screen reader friendly

### 1.2 Design Language

**Visual Style:**
- Modern, clean, minimalist
- Dark mode default (light mode available)
- High contrast for accessibility
- Consistent spacing system (4px grid)

**Typography:**
- Headings: Inter (700 weight)
- Body: Inter (400 weight)
- Monospace: JetBrains Mono (code/numbers)

**Color Palette:**

```
Primary:
- Brand: #6366F1 (Indigo)
- Brand Hover: #4F46E5
- Brand Light: #818CF8

Neutrals (Dark Mode):
- Background: #0F172A
- Surface: #1E293B
- Border: #334155
- Text Primary: #F1F5F9
- Text Secondary: #94A3B8

Semantic:
- Success: #10B981
- Warning: #F59E0B
- Error: #EF4444
- Info: #3B82F6
```

**Spacing System:**

```
xs: 4px
sm: 8px
md: 16px
lg: 24px
xl: 32px
2xl: 48px
3xl: 64px
```

**Border Radius:**

```
sm: 4px (buttons, inputs)
md: 8px (cards)
lg: 12px (modals)
xl: 16px (hero sections)
full: 9999px (avatars, pills)
```

---

## 2. User Flows

### 2.1 New User Onboarding

```
Landing Page
    │
    ├─► Sign Up (Email/OAuth)
    │     │
    │     ├─► Enter email/password
    │     ├─► Email verification
    │     └─► Choose role (Creator/Viewer)
    │           │
    │           ├─► Creator: Upload first video prompt
    │           └─► Viewer: Pick 5 interests
    │                 │
    │                 └─► Personalized feed
    │
    └─► Browse as Guest
          │
          └─► Watch video → Prompt to sign up
```

**Screens:**
1. Landing page
2. Sign up modal
3. Email verification
4. Role selection
5. Interest picker (viewers)
6. Upload prompt (creators)
7. Home feed

### 2.2 Video Upload Flow

```
Creator Dashboard
    │
    └─► Click "Upload Video"
          │
          ├─► Drag & drop or select file
          │     │
          │     ├─► Upload progress (0-100%)
          │     └─► Processing animation
          │
          ├─► Enter metadata (while uploading)
          │     ├─► Title (required)
          │     ├─► Description
          │     ├─► Tags (max 10)
          │     ├─► Category
          │     ├─► Thumbnail (auto or custom)
          │     └─► Visibility (public/unlisted/private)
          │
          └─► Submit
                │
                ├─► Success: "Video uploaded! Processing..."
                └─► Redirect to video page (shows processing state)
```

**Screens:**
1. Upload modal
2. File picker
3. Upload progress
4. Metadata form
5. Success confirmation
6. Video page (processing state)

### 2.3 Video Discovery Flow

```
Home Page
    │
    ├─► For You (Personalized)
    │     │
    │     ├─► Infinite scroll
    │     ├─► Click video thumbnail
    │     └─► Watch video
    │
    ├─► Search
    │     │
    │     ├─► Type query
    │     ├─► See autocomplete suggestions
    │     ├─► Filter by category/duration/date
    │     └─► Click result
    │
    ├─► Trending
    │     │
    │     └─► Top videos (hour/day/week)
    │
    └─► Browse Categories
          │
          └─► Grid of categories
                │
                └─► Videos in category
```

**Screens:**
1. Home feed
2. Search page
3. Trending page
4. Category page
5. Video player page

### 2.4 Video Watching Flow

```
Video Player Page
    │
    ├─► Video player (auto-play)
    │     │
    │     ├─► Play/pause
    │     ├─► Seek
    │     ├─► Volume
    │     ├─► Quality selector
    │     ├─► Fullscreen
    │     └─► Picture-in-picture
    │
    ├─► Engagement
    │     ├─► Like/unlike
    │     ├─► Share
    │     ├─► Save to playlist
    │     └─► Report
    │
    ├─► Video info
    │     ├─► Title, description
    │     ├─► Creator profile (click → channel)
    │     ├─► Subscribe button
    │     └─► Tags
    │
    ├─► Comments
    │     ├─► Read comments
    │     ├─► Write comment
    │     ├─► Reply to comment
    │     └─► Like comment
    │
    └─► Related videos
          │
          └─► Click → new video
```

**Screens:**
1. Video player page
2. Full-screen player
3. Comment thread
4. Share modal
5. Playlist modal

### 2.5 Creator Dashboard Flow

```
Creator Dashboard
    │
    ├─► Analytics Overview
    │     ├─► Total views, likes, comments
    │     ├─► Line chart (views over time)
    │     ├─► Top videos
    │     └─► Audience demographics
    │
    ├─► Video Manager
    │     ├─► List all videos
    │     ├─► Edit video
    │     ├─► Delete video
    │     └─► View analytics per video
    │
    ├─► Comments
    │     ├─► All comments across videos
    │     ├─► Reply
    │     └─► Moderate (hide/delete)
    │
    └─► Settings
          ├─► Profile settings
          ├─► Channel customization
          └─► Notifications
```

**Screens:**
1. Dashboard home
2. Video manager
3. Video editor
4. Comment manager
5. Settings page

---

## 3. Screen Specifications

### 3.1 Landing Page

**Layout:**

```
┌─────────────────────────────────────────────────────────┐
│  [Logo] StreamHub          [Search]  [Login] [Sign Up]  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│          HERO SECTION (80vh)                            │
│                                                         │
│      Discover & Share                                   │
│      Creative Content                                   │
│                                                         │
│      [Get Started] [Browse Videos]                      │
│                                                         │
│      Background: Gradient + subtle video grid          │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│      TRENDING SECTION                                   │
│      "What's Hot Right Now"                             │
│                                                         │
│      [Video Card] [Video Card] [Video Card]             │
│      [Video Card] [Video Card] [Video Card]             │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│      FEATURES SECTION                                   │
│                                                         │
│      [Icon] For Creators      [Icon] For Viewers        │
│      Upload unlimited         Discover unique           │
│      Earn fairly              AI recommendations        │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│      FOOTER                                             │
│      About | Blog | Careers | Terms | Privacy          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Components:**
- Navigation bar (sticky)
- Hero section with CTA buttons
- Video card grid (6 trending videos)
- Feature cards
- Footer

**Interactions:**
- Smooth scroll to sections
- Video card hover: Scale up 1.05x, show play icon
- CTA buttons: Hover lift effect
- Auto-play muted video previews on hover (optional)

---

### 3.2 Home Feed (Authenticated)

**Layout:**

```
┌─────────────────────────────────────────────────────────┐
│  [Logo]  [Search........................]  [Upload] [👤]│
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Sidebar (240px)       Main Feed (flex-1)               │
│  ┌──────────────┐      ┌─────────────────────────────┐ │
│  │ For You      │      │  [Video Card]               │ │
│  │ Trending     │      │  ┌───────────┐              │ │
│  │ Subscriptions│      │  │ Thumbnail │              │ │
│  │              │      │  │  (16:9)   │              │ │
│  │ Categories   │      │  └───────────┘              │ │
│  │ - Music      │      │  Title                      │ │
│  │ - Gaming     │      │  Creator • Views • 2h ago   │ │
│  │ - Education  │      └─────────────────────────────┘ │
│  │ - Tech       │                                      │
│  │              │      [Video Card] [Video Card]       │
│  │ My Stuff     │      [Video Card] [Video Card]       │
│  │ - Playlists  │      [Video Card] [Video Card]       │
│  │ - History    │                                      │
│  │ - Liked      │      Infinite scroll...              │
│  └──────────────┘                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Components:**
- Top navigation (search, upload, profile)
- Sidebar navigation (collapsible on mobile)
- Video card grid (responsive: 1-4 columns)
- Infinite scroll loader

**Video Card Spec:**

```
┌─────────────────────────┐
│                         │
│     Thumbnail           │ ← 16:9 ratio
│     (Image + Duration)  │ ← Duration badge bottom-right
│                         │
├─────────────────────────┤
│ [Avatar] Title (2 lines)│ ← Truncate with ellipsis
│         Creator name    │ ← Gray text
│         Views • Time    │ ← Metadata
└─────────────────────────┘
```

**Interactions:**
- Sidebar: Click category → filter feed
- Video card: Hover → show preview (2s delay)
- Video card: Click → navigate to player
- Infinite scroll: Load more on scroll bottom
- Search: Focus → show recent searches

---

### 3.3 Video Player Page

**Layout:**

```
┌─────────────────────────────────────────────────────────┐
│  [Logo]  [Search........................]  [Upload] [👤]│
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Video Player Container (max-width: 1280px)             │
│  ┌─────────────────────────────────────────────────┐   │
│  │                                                 │   │
│  │                                                 │   │
│  │           VIDEO PLAYER (16:9)                   │   │
│  │                                                 │   │
│  │                                                 │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────┐  ┌───────────────────────┐   │
│  │  Main Content        │  │  Related Videos       │   │
│  │  ┌────────────────┐  │  │  [Video Card]         │   │
│  │  │ Title          │  │  │  [Video Card]         │   │
│  │  │ 1.2M views     │  │  │  [Video Card]         │   │
│  │  └────────────────┘  │  │  [Video Card]         │   │
│  │                      │  │  ...                  │   │
│  │  [👍 123K] [💬 4.5K]│  │                       │   │
│  │  [Share] [Save]     │  │                       │   │
│  │                      │  │                       │   │
│  │  Creator Section     │  │                       │   │
│  │  [Avatar] Name       │  │                       │   │
│  │  500K subscribers    │  │                       │   │
│  │  [Subscribe]         │  │                       │   │
│  │                      │  │                       │   │
│  │  Description         │  │                       │   │
│  │  (Expandable)        │  │                       │   │
│  │                      │  │                       │   │
│  │  Comments Section    │  │                       │   │
│  │  4.5K comments       │  │                       │   │
│  │  [Sort by]           │  │                       │   │
│  │                      │  │                       │   │
│  │  [Write comment...]  │  │                       │   │
│  │                      │  │                       │   │
│  │  Comment Thread      │  │                       │   │
│  │  Comment Thread      │  │                       │   │
│  │  ...                 │  │                       │   │
│  └──────────────────────┘  └───────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Video Player Controls:**

```
Bottom Control Bar:
┌────────────────────────────────────────────────────┐
│ [▶️] ─────●─────────────── 12:34 / 45:00          │
│ [🔊] [⚙️] [CC] [PIP] [Fullscreen]                 │
└────────────────────────────────────────────────────┘

Hover States:
- Progress bar: Show thumbnail preview on hover
- Volume: Vertical slider on hover
- Settings: Dropdown menu (quality, speed)
```

**Engagement Buttons:**

```
[👍 Like] [👎]  |  [💬 Comment]  |  [↗️ Share]  |  [➕ Save]

States:
- Default: Outline icon, gray
- Active: Filled icon, brand color
- Hover: Scale 1.1x, show tooltip
```

**Comment Component:**

```
┌──────────────────────────────────────────┐
│ [Avatar] @username • 2 hours ago         │
│                                          │
│ Comment text goes here. Can be           │
│ multiple lines with proper wrapping.     │
│                                          │
│ [👍 12] [Reply]                          │
│                                          │
│   └─► [Avatar] @reply • 1 hour ago      │
│       Reply text...                      │
│       [👍 3] [Reply]                     │
└──────────────────────────────────────────┘
```

**Interactions:**
- Video: Click → play/pause
- Video: Double-click → fullscreen
- Video: Keyboard shortcuts (Space, F, M, ←→)
- Like button: Click → toggle, animate count
- Comment: Click reply → expand reply box
- Description: Click "Show more" → expand
- Related videos: Click → navigate (keep playlist queue)

---

### 3.4 Upload Modal

**Layout:**

```
┌─────────────────────────────────────────────────┐
│  Upload Video                              [✕]  │
├─────────────────────────────────────────────────┤
│                                                 │
│  Step 1: Select File                            │
│  ┌───────────────────────────────────────────┐ │
│  │                                           │ │
│  │       📹                                  │ │
│  │                                           │ │
│  │   Drag & drop video file here             │ │
│  │   or [Select File]                        │ │
│  │                                           │ │
│  │   Supported: MP4, WebM (max 2GB)          │ │
│  │                                           │ │
│  └───────────────────────────────────────────┘ │
│                                                 │
└─────────────────────────────────────────────────┘

After file selected:
┌─────────────────────────────────────────────────┐
│  Upload Video                              [✕]  │
├─────────────────────────────────────────────────┤
│                                                 │
│  Step 2: Upload & Details                       │
│                                                 │
│  my-video.mp4 (450 MB)                          │
│  ████████████████░░░░ 80%                       │
│  Uploading... 2min remaining                    │
│                                                 │
│  ─────────────────────────────────────────────  │
│                                                 │
│  Video Details (fill while uploading)           │
│                                                 │
│  Title *                                        │
│  [...................................]          │
│                                                 │
│  Description                                    │
│  [...................................]          │
│  [...................................]          │
│                                                 │
│  Tags (max 10)                                  │
│  [Tag1] [Tag2] [Tag3] [+ Add tag]              │
│                                                 │
│  Category                                       │
│  [Dropdown: Select category ▼]                 │
│                                                 │
│  Thumbnail                                      │
│  [Preview] [Upload custom]                     │
│                                                 │
│  Visibility                                     │
│  ⚫ Public  ⚪ Unlisted  ⚪ Private              │
│                                                 │
│  [Cancel]                    [Publish] ←disabled│
│                              until upload done  │
└─────────────────────────────────────────────────┘

After publish:
┌─────────────────────────────────────────────────┐
│  ✅ Video Uploaded Successfully!                │
├─────────────────────────────────────────────────┤
│                                                 │
│  Your video is processing...                    │
│  You'll be notified when it's ready to watch.   │
│                                                 │
│  [Go to Dashboard]  [Upload Another]            │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Interactions:**
- Drag & drop: Highlight drop zone on drag over
- File selected: Show preview, start upload
- Upload progress: Real-time percentage updates
- Form: Validate on blur, show errors inline
- Publish: Disabled until upload complete & required fields filled
- Success: Show checkmark animation, auto-close after 3s

---

### 3.5 Creator Dashboard

**Layout:**

```
┌─────────────────────────────────────────────────────────┐
│  [Logo]  Creator Studio         [Notifications] [👤]     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Sidebar           Analytics Overview                   │
│  ┌────────────┐    ┌─────────────────────────────────┐ │
│  │ Dashboard  │    │  Last 28 Days                   │ │
│  │ Videos     │    │                                 │ │
│  │ Comments   │    │  ┌──────────┐  ┌──────────┐    │ │
│  │ Analytics  │    │  │ 125.4K   │  │  8.2K    │    │ │
│  │ Settings   │    │  │ Views    │  │  Likes   │    │ │
│  └────────────┘    │  └──────────┘  └──────────┘    │ │
│                    │                                 │ │
│                    │  ┌──────────┐  ┌──────────┐    │ │
│                    │  │  2.1K    │  │  45      │    │ │
│                    │  │ Comments │  │  Videos  │    │ │
│                    │  └──────────┘  └──────────┘    │ │
│                    └─────────────────────────────────┘ │
│                                                         │
│                    Chart: Views Over Time               │
│                    ┌─────────────────────────────────┐ │
│                    │  📈 Line chart with sparkline    │ │
│                    │      showing daily views         │ │
│                    └─────────────────────────────────┘ │
│                                                         │
│                    Top Performing Videos                │
│                    ┌─────────────────────────────────┐ │
│                    │ 1. Video Title     45K views    │ │
│                    │ 2. Video Title     32K views    │ │
│                    │ 3. Video Title     28K views    │ │
│                    └─────────────────────────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Video Manager View:**

```
┌─────────────────────────────────────────────────────────┐
│  Your Videos (45)                    [+ Upload Video]    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [Search videos...]  [Filter: All ▼]  [Sort: Recent ▼]  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ [Thumbnail] Title                               │   │
│  │             Description preview...              │   │
│  │             ⚫ Public • 12K views • 2 days ago  │   │
│  │             [Edit] [Analytics] [Delete]         │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  [Video Row]                                            │
│  [Video Row]                                            │
│  ...                                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Interactions:**
- Stats cards: Hover → show % change from previous period
- Chart: Hover → show tooltip with exact values
- Video rows: Hover → highlight, show action buttons
- Edit: Open inline editor or modal
- Delete: Confirmation dialog

---

### 3.6 Search Results Page

**Layout:**

```
┌─────────────────────────────────────────────────────────┐
│  [Logo]  [Search: "javascript tutorial"......] [👤]     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Filters                Results (1,234)                 │
│  ┌──────────────┐      ┌─────────────────────────────┐ │
│  │ Type         │      │ [Video Result Card]         │ │
│  │ ☑️ Videos    │      │ ┌────┐                      │ │
│  │ ☑️ Music     │      │ │Thum│ Title               │ │
│  │              │      │ │nail│ Creator • Views     │ │
│  │ Duration     │      │ └────┘ Description...      │ │
│  │ ◉ Any        │      └─────────────────────────────┘ │
│  │ ⚪ < 5min    │                                      │
│  │ ⚪ 5-20min   │      [Video Result Card]             │
│  │ ⚪ > 20min   │      [Video Result Card]             │
│  │              │      [Video Result Card]             │
│  │ Upload Date  │      ...                             │
│  │ ◉ Any        │                                      │
│  │ ⚪ Today     │      [Load More]                     │
│  │ ⚪ This Week │                                      │
│  │              │                                      │
│  │ Category     │                                      │
│  │ ☑️ Education│                                      │
│  │ ☐ Gaming    │                                      │
│  │ ...         │                                      │
│  └──────────────┘                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Search Result Card:**

```
┌───────────────────────────────────────────────┐
│ [Thumbnail 120x68] Video Title                │
│ (16:9 aspect)      Creator Name • 45K views   │
│                    2 days ago                 │
│                                               │
│                    Description preview text   │
│                    that shows first 2 lines   │
│                    with ellipsis...           │
└───────────────────────────────────────────────┘
```

**Interactions:**
- Filters: Click → toggle, instant results update
- Search input: Type → auto-save to URL params
- Result card: Click → navigate to video
- Sort dropdown: Change → re-fetch results
- Load more: Paginated infinite scroll

---

## 4. Component Library

### 4.1 Buttons

**Primary Button:**

```tsx
<Button variant="primary" size="md">
  Upload Video
</Button>

Styles:
- Background: #6366F1
- Hover: #4F46E5
- Active: scale(0.98)
- Disabled: opacity 0.5, cursor not-allowed
- Padding: 12px 24px
- Border-radius: 8px
- Font: 600 weight
```

**Secondary Button:**

```tsx
<Button variant="secondary">
  Cancel
</Button>

Styles:
- Background: transparent
- Border: 1px solid #334155
- Hover: bg #1E293B
```

**Icon Button:**

```tsx
<IconButton icon={<ShareIcon />} label="Share" />

Styles:
- Size: 40x40px
- Border-radius: 50%
- Hover: bg #1E293B
- Active: scale(0.95)
```

### 4.2 Input Fields

**Text Input:**

```tsx
<Input 
  label="Title"
  placeholder="Enter video title"
  required
  error="Title is required"
/>

States:
- Default: border #334155
- Focus: border #6366F1, ring
- Error: border #EF4444, red text
- Disabled: opacity 0.6
```

**Textarea:**

```tsx
<Textarea 
  label="Description"
  rows={4}
  maxLength={5000}
/>

Features:
- Auto-resize on input
- Character counter (dynamic)
- Markdown preview (optional)
```

**Select Dropdown:**

```tsx
<Select 
  label="Category"
  options={categories}
  placeholder="Select category"
/>

Interactions:
- Click → expand dropdown
- Keyboard: Arrow keys to navigate
- Type → filter options
```

### 4.3 Cards

**Video Card:**

```tsx
<VideoCard
  thumbnail="..."
  title="Video Title"
  creator="Creator Name"
  views={12500}
  uploadedAt="2 days ago"
  duration={345} // seconds
/>

Layout:
┌─────────────┐
│ Thumbnail   │ ← 16:9 ratio, lazy load
│   [5:45]    │ ← Duration badge
├─────────────┤
│ [Avatar]    │
│ Title...    │ ← 2 lines max
│ Creator     │ ← Gray text
│ Views•Time  │
└─────────────┘

Hover:
- Scale: 1.03
- Shadow: elevate
- Show quick actions (Save, Add to playlist)
```

**Creator Card:**

```tsx
<CreatorCard
  avatar="..."
  name="Creator Name"
  subscribers={500000}
  isSubscribed={false}
/>

Layout:
┌─────────────────┐
│  [  Avatar  ]   │ ← Large circle
│                 │
│  Creator Name   │
│  500K subs      │
│                 │
│  [Subscribe]    │
└─────────────────┘
```

### 4.4 Modals

**Base Modal:**

```tsx
<Modal 
  isOpen={true}
  onClose={handleClose}
  title="Modal Title"
>
  {children}
</Modal>

Features:
- Backdrop: semi-transparent black
- Close on: ESC key, backdrop click, X button
- Animation: Fade in + scale up
- Focus trap: Tab cycles within modal
- Max width: 600px (sm), 800px (md), 1000px (lg)
```

**Confirmation Dialog:**

```tsx
<ConfirmDialog
  title="Delete Video"
  message="Are you sure you want to delete this video? This action cannot be undone."
  confirmText="Delete"
  confirmVariant="danger"
  onConfirm={handleDelete}
/>

Layout:
┌──────────────────────┐
│ ⚠️ Delete Video      │
├──────────────────────┤
│ Message text...      │
│                      │
│ [Cancel] [Delete]    │
└──────────────────────┘
```

### 4.5 Navigation

**Top Navigation Bar:**

```tsx
<Navbar>
  <Logo />
  <SearchBar />
  <NavItems>
    <UploadButton />
    <NotificationBell />
    <ProfileDropdown />
  </NavItems>
</Navbar>

Sticky: Yes (position: sticky, top: 0)
Height: 64px
Z-index: 1000
Background: #0F172A with blur backdrop
```

**Sidebar Navigation:**

```tsx
<Sidebar collapsed={false}>
  <NavGroup title="Discover">
    <NavItem icon={<HomeIcon />} label="For You" active />
    <NavItem icon={<TrendingIcon />} label="Trending" />
  </NavGroup>
  <NavGroup title="Categories">
    <NavItem label="Music" />
    <NavItem label="Gaming" />
  </NavGroup>
</Sidebar>

Interactions:
- Collapse/expand toggle
- Active state: blue accent bar
- Hover: highlight background
- Mobile: overlay drawer
```

### 4.6 Feedback Components

**Toast Notifications:**

```tsx
<Toast type="success" duration={3000}>
  Video uploaded successfully!
</Toast>

Positions: top-right, top-center, bottom-right
Types: success, error, warning, info
Auto-dismiss: Yes (configurable)
Animation: Slide in from right, fade out
```

**Loading Spinners:**

```tsx
<Spinner size="md" />

Sizes: sm (16px), md (24px), lg (32px)
Animation: Rotate 360deg, 1s linear infinite
```

**Progress Bars:**

```tsx
<ProgressBar value={75} max={100} />

States:
- Determinate: Show percentage
- Indeterminate: Animated pulse
- Color: Brand blue (#6366F1)
```

### 4.7 Data Display

**Stats Card:**

```tsx
<StatCard
  label="Total Views"
  value="125.4K"
  change={+12.5}
  period="vs last month"
/>

Layout:
┌─────────────┐
│ Total Views │
│   125.4K    │ ← Large font
│ ↗️ +12.5%   │ ← Green if positive
└─────────────┘
```

**Table:**

```tsx
<Table>
  <TableHeader>
    <TableColumn>Title</TableColumn>
    <TableColumn>Views</TableColumn>
    <TableColumn>Date</TableColumn>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell>Video Title</TableCell>
      <TableCell>12.5K</TableCell>
      <TableCell>2 days ago</TableCell>
    </TableRow>
  </TableBody>
</Table>

Features:
- Sortable columns (click header)
- Hover row highlight
- Responsive: stack on mobile
```

---

## 5. Interaction Patterns

### 5.1 Navigation Patterns

**Deep Linking:**
- All pages have unique URLs
- State preserved in URL params
- Browser back/forward supported

**Breadcrumbs:**

```
Home > Category > Video Title
Click any: Navigate to that level
```

**Keyboard Shortcuts:**

```
Global:
/ → Focus search
? → Show keyboard shortcuts help
ESC → Close modal/drawer

Video Player:
Space → Play/pause
K → Play/pause
F → Fullscreen
M → Mute/unmute
← → → Seek backward/forward 5s
0-9 → Seek to percentage (0% - 90%)
```

### 5.2 Form Interactions

**Auto-save:**
- Draft state saved to localStorage
- Restore on page revisit
- Show "Draft saved" indicator

**Validation:**
- Inline validation on blur
- Real-time for critical fields (username)
- Error messages below field
- Form-level error summary at top

**Multi-step Forms:**

```
Step indicator:
● ──── ○ ──── ○
1. File  2. Details  3. Publish

- Show progress
- Allow back/next navigation
- Save state between steps
```

### 5.3 Data Loading

**Skeleton Screens:**

```
While loading video feed:
┌─────────────┐
│ ▓▓▓▓▓▓▓▓▓▓  │ ← Pulsing gray boxes
├─────────────┤
│ ▓▓▓ ▓▓▓▓    │
│ ▓▓▓▓        │
└─────────────┘

Better than: Spinners or blank screen
```

**Optimistic Updates:**

```
User clicks Like:
1. Immediately update UI (count +1, button active)
2. Send API request in background
3. If fails: Revert UI, show error toast
```

**Infinite Scroll:**

```
Scroll position → Load more threshold (200px from bottom)
  ↓
Load next page (20 items)
  ↓
Append to existing list
  ↓
Show loading indicator at bottom
```

### 5.4 Error Handling

**Error States:**

```
Network Error:
┌──────────────────────┐
│  😕                  │
│  Couldn't load       │
│  videos              │
│                      │
│  [Try Again]         │
└──────────────────────┘

404 Not Found:
┌──────────────────────┐
│  🔍                  │
│  Video not found     │
│                      │
│  [Go Home]           │
└──────────────────────┘

Empty State:
┌──────────────────────┐
│  📹                  │
│  No videos yet       │
│                      │
│  [Upload First Video]│
└──────────────────────┘
```

**Error Recovery:**

```
Failed upload:
1. Show error message
2. Offer retry with same file
3. Allow user to check connection
4. Resume from last checkpoint (if supported)
```

---

## 6. Responsive Design

### 6.1 Breakpoints

```
xs: 0px       (Mobile portrait)
sm: 640px     (Mobile landscape)
md: 768px     (Tablet)
lg: 1024px    (Desktop)
xl: 1280px    (Large desktop)
2xl: 1536px   (Ultra-wide)
```

### 6.2 Layout Adaptations

**Video Grid:**

```
Mobile (< 640px):   1 column
Tablet (640-1024):  2 columns
Desktop (> 1024):   3-4 columns
```

**Sidebar:**

```
Mobile: Hidden by default, overlay drawer
Tablet: Collapsed icons only
Desktop: Full sidebar with labels
```

**Video Player:**

```
Mobile: Full width, aspect ratio preserved
Desktop: Max width 1280px, centered
```

### 6.3 Touch Interactions

**Mobile Gestures:**

```
Video Player:
- Swipe left/right: Seek ±10s
- Swipe up/down (left side): Brightness
- Swipe up/down (right side): Volume
- Double-tap left/right: Seek ±10s
- Pinch to zoom: Fullscreen mode

Feed:
- Pull to refresh
- Swipe left on video card: Quick actions
```

**Button Sizes:**

```
Desktop: min 40x40px
Mobile: min 48x48px (WCAG touch target)
```

---

## 7. Accessibility

### 7.1 WCAG 2.1 AA Compliance

**Color Contrast:**

```
Text on background: min 4.5:1
Large text: min 3:1
UI components: min 3:1

Check:
- White text (#F1F5F9) on dark bg (#0F172A) ✅ 15.4:1
- Blue button (#6366F1) on dark bg ✅ 4.8:1
```

**Focus Indicators:**

```css
*:focus-visible {
  outline: 2px solid #6366F1;
  outline-offset: 2px;
}

/* High contrast mode */
@media (prefers-contrast: high) {
  *:focus-visible {
    outline-width: 3px;
  }
}
```

### 7.2 Keyboard Navigation

**Tab Order:**

```
Logical flow: Logo → Search → Upload → Profile
Video cards: Tab to navigate, Enter to select
Modal: Focus trap, ESC to close
```

**Skip Links:**

```html
<a href="#main-content" class="skip-link">
  Skip to main content
</a>

<!-- Hidden unless focused -->
.skip-link:not(:focus) {
  position: absolute;
  left: -9999px;
}
```

### 7.3 Screen Reader Support

**ARIA Labels:**

```html
<button aria-label="Like video">
  <HeartIcon />
</button>

<nav aria-label="Main navigation">
  ...
</nav>

<input 
  type="text"
  aria-describedby="error-message"
  aria-invalid="true"
/>
```

**Live Regions:**

```html
<!-- Announce dynamic updates -->
<div aria-live="polite" aria-atomic="true">
  Video uploaded successfully
</div>

<!-- For critical alerts -->
<div aria-live="assertive">
  Upload failed. Please try again.
</div>
```

**Video Player Accessibility:**

```html
<video aria-label="Video title">
  <track kind="captions" src="captions.vtt" />
</video>

<!-- Controls -->
<button aria-label="Play video">▶️</button>
<button aria-label="Mute">🔊</button>
```

### 7.4 Alternative Content

**Image Alt Text:**

```html
<img 
  src="thumbnail.jpg"
  alt="JavaScript tutorial: Variables and data types"
/>

<!-- Decorative images -->
<img src="decoration.png" alt="" role="presentation" />
```

**Video Captions:**

```
Required:
- Closed captions (subtitles)
- Transcript available on demand

Optional (Phase 2):
- Audio descriptions for visual content
- Sign language interpretation
```

---

## 8. Micro-interactions

### 8.1 Button Interactions

**Hover:**

```css
button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(99, 102, 241, 0.3);
  transition: all 0.2s ease;
}
```

**Click:**

```css
button:active {
  transform: scale(0.98);
}
```

**Loading State:**

```
[Button Text] → [Spinner] Loading...
Disable button during loading
Show success checkmark on complete
```

### 8.2 Like Animation

**Click Like Button:**

```
1. Button scales up 1.3x
2. Heart fills with color (red)
3. Particle burst animation (small hearts)
4. Count increments with slide up animation
5. Button returns to normal size

Duration: 400ms
Easing: cubic-bezier(0.34, 1.56, 0.64, 1)
```

### 8.3 Card Hover Effects

**Video Card:**

```
Default → Hover:
- Translate Y: -4px
- Shadow: increase elevation
- Thumbnail: slight zoom (1.05x)
- Show preview after 1s delay
- Play icon appears
```

### 8.4 Page Transitions

**Navigation:**

```
Page A → Page B:
1. Fade out current page (200ms)
2. Update URL
3. Fade in new page (300ms)
4. Scroll to top (smooth)

Or use View Transitions API:
document.startViewTransition(() => {
  // Update DOM
});
```

### 8.5 Notification Animations

**Toast Enter:**

```
1. Slide in from right (300ms)
2. Ease-out timing
3. Slight bounce on land

Exit:
1. Fade out (200ms)
2. Slide out to right
```

---

## 9. States & Feedback

### 9.1 Loading States

**Button:**

```
<button disabled>
  <Spinner size="sm" /> Uploading...
</button>
```

**Page:**

```
Skeleton screen for initial load
Shimmer effect on skeleton boxes
```

**Inline:**

```
"Loading more videos..." with small spinner
```

### 9.2 Success States

**Toast:**

```
✅ Video uploaded successfully!
[Auto-dismiss in 3s]
```

**Inline:**

```
✅ Changes saved
[Fade out after 2s]
```

**Page:**

```
Success screen with:
- Large checkmark icon
- Success message
- Next action CTA
```

### 9.3 Error States

**Form Field:**

```
Input with red border
❌ Error message below
Icon in input (right side)
```

**Page:**

```
Error illustration
Error message
Suggested action or retry button
```

**Toast:**

```
❌ Upload failed. Please try again.
[Dismiss] [Retry]
```

### 9.4 Empty States

**No Results:**

```
🔍 No videos found for "query"
Try different keywords or browse categories
[Browse Categories]
```

**No Content:**

```
📹 You haven't uploaded any videos yet
Share your creativity with the world!
[Upload Video]
```

### 9.5 Disabled States

**Button:**

```css
button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  /* No hover effects */
}
```

**Input:**

```css
input:disabled {
  background: #1E293B;
  color: #64748B;
  cursor: not-allowed;
}
```

---

## 10. Content Guidelines

### 10.1 Copywriting

**Tone:**
- Friendly and encouraging
- Clear and concise
- Avoid jargon
- Action-oriented

**Examples:**

```
❌ Bad: "An error occurred during the upload process"
✅ Good: "Oops! Upload failed. Try again?"

❌ Bad: "Insufficient permissions to perform action"
✅ Good: "You don't have access to this. Try logging in."

❌ Bad: "No results returned from database query"
✅ Good: "No videos found. Try different keywords?"
```

### 10.2 Button Labels

**Action-oriented:**

```
✅ Upload Video (not "Submit")
✅ Save Changes (not "OK")
✅ Delete Forever (not "Confirm")
✅ Try Again (not "Retry")
```

### 10.3 Error Messages

**Structure:**

```
1. What happened (brief)
2. Why it happened (if helpful)
3. What to do (actionable)

Example:
"Upload failed. File size exceeds 2GB limit.
Try compressing your video or choose a smaller file."
```

### 10.4 Placeholder Text

**Helpful hints:**

```
Input: "Search videos, creators, tags..."
Textarea: "Tell viewers what your video is about..."
Tags: "Add tags like 'tutorial', 'beginner'..."
```

### 10.5 Time Formats

```
< 1 min: "Just now"
< 1 hour: "X minutes ago"
< 24 hours: "X hours ago"
< 7 days: "X days ago"
< 30 days: "X weeks ago"
> 30 days: "Month DD, YYYY"
```

---

## 11. Animation Guidelines

### 11.1 Timing

```
Instant: 0ms (state toggles)
Fast: 100-200ms (micro-interactions)
Normal: 200-300ms (most animations)
Slow: 300-500ms (page transitions)
Very Slow: 500ms+ (special effects)
```

### 11.2 Easing Functions

```
Default: ease-in-out (most animations)
Enter: ease-out (elements appearing)
Exit: ease-in (elements leaving)
Bounce: cubic-bezier(0.34, 1.56, 0.64, 1)
```

### 11.3 Performance

**Hardware Acceleration:**

```css
/* Use transform & opacity (GPU-accelerated) */
.animate {
  transform: translateX(0);
  opacity: 1;
  transition: transform 0.3s, opacity 0.3s;
}

/* Avoid animating: */
❌ width, height, top, left (causes reflow)
✅ transform, opacity (GPU-accelerated)
```

**Reduce Motion:**

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## 12. Platform-Specific Considerations

### 12.1 Mobile App (React Native)

**Native Gestures:**
- Swipe back to navigate
- Pull to refresh
- Long press for context menu

**Components:**
- Use native video player
- Native share sheet
- Platform-specific navigation

### 12.2 Desktop App

**Window Controls:**
- Minimize, maximize, close
- Resizable window
- Remember window size/position

**Keyboard:**
- Full keyboard shortcut support
- Menu bar shortcuts

### 12.3 Progressive Web App (PWA)

**Features:**
- Install prompt
- Offline support (service worker)
- Background sync for uploads
- Push notifications

---

## Appendix

### A. Component Specifications Summary

| Component | Variants | States | Props |
|-----------|----------|--------|-------|
| Button | primary, secondary, ghost, danger | default, hover, active, disabled, loading | variant, size, disabled, loading, onClick |
| Input | text, email, password, number | default, focus, error, disabled | label, placeholder, error, required, disabled |
| Card | video, creator, stats | default, hover, loading | data, onClick, actions |
| Modal | small, medium, large | open, closed | isOpen, title, onClose, children |
| Toast | success, error, warning, info | visible, hidden | type, message, duration, onDismiss |

### B. Figma File Structure

```
StreamHub Design System/
├── 🎨 Foundations/
│   ├── Colors
│   ├── Typography
│   ├── Spacing
│   └── Shadows
├── 🧩 Components/
│   ├── Buttons
│   ├── Inputs
│   ├── Cards
│   ├── Modals
│   └── Navigation
├── 📱 Screens/
│   ├── Landing
│   ├── Home Feed
│   ├── Video Player
│   ├── Upload
│   └── Dashboard
└── 📋 Templates/
    ├── Mobile
    ├── Tablet
    └── Desktop
```

### C. Accessibility Checklist

- [ ] Color contrast meets WCAG AA (4.5:1)
- [ ] All interactive elements keyboard accessible
- [ ] Focus indicators visible
- [ ] ARIA labels on all icon-only buttons
- [ ] Form inputs have associated labels
- [ ] Error messages announced to screen readers
- [ ] Skip navigation link present
- [ ] Video player has captions
- [ ] Images have alt text
- [ ] Touch targets min 48x48px on mobile
- [ ] Reduced motion preference respected
- [ ] Dark/light mode both accessible

---

**End of UX & Interaction Specification**

*Version: 1.0 | Last Updated: February 2026*
