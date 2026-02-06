# StreamHub - User Stories & Acceptance Criteria

**Version:** 1.0  
**Last Updated:** February 2026

---

## Table of Contents

1. [Authentication & User Management](#authentication--user-management)
2. [Video Upload & Management](#video-upload--management)
3. [Video Discovery & Playback](#video-discovery--playback)
4. [Engagement Features](#engagement-features)
5. [Creator Dashboard](#creator-dashboard)
6. [Monetization](#monetization)

---

## Authentication & User Management

### US-001: User Registration

**As a** new user  
**I want to** create an account  
**So that** I can upload videos and engage with content

**Acceptance Criteria:**

```gherkin
Given I am on the registration page
When I enter a valid username, email, and password
And I select my role (viewer or creator)
And I click "Sign Up"
Then my account should be created
And I should receive a verification email
And I should be redirected to the home page
And I should be logged in automatically

Validation Rules:
- Username: 3-50 characters, alphanumeric + underscore
- Email: Valid email format, unique in system
- Password: Minimum 8 characters, 1 uppercase, 1 number
```

**Edge Cases:**
- Email already exists → Show error: "Email already registered"
- Username taken → Show error: "Username not available"
- Weak password → Show specific requirements not met
- Network error → Show retry option

---

### US-002: User Login

**As a** registered user  
**I want to** log in to my account  
**So that** I can access my personalized content

**Acceptance Criteria:**

```gherkin
Given I am on the login page
When I enter my email and password
And I click "Log In"
Then I should be authenticated
And I should be redirected to the home page
And my session should be saved

Given I enter incorrect credentials
When I click "Log In"
Then I should see an error message "Invalid email or password"
And my login attempts should be rate limited after 5 tries

Given I check "Remember me"
When I log in successfully
Then my session should persist for 30 days
```

---

### US-003: OAuth Login

**As a** user  
**I want to** log in with Google or GitHub  
**So that** I don't have to create a new password

**Acceptance Criteria:**

```gherkin
Given I am on the login page
When I click "Continue with Google"
Then I should be redirected to Google OAuth
And after authorization, I should be logged in
And my account should be created if it doesn't exist

Given I have an existing account with the same email
When I use OAuth for the first time
Then my accounts should be linked
And I should be able to log in with either method
```

---

### US-004: User Profile Edit

**As a** logged-in user  
**I want to** edit my profile  
**So that** I can update my information

**Acceptance Criteria:**

```gherkin
Given I am on my profile page
When I update my username, bio, or avatar
And I click "Save Changes"
Then my profile should be updated
And I should see a success message
And the changes should be visible immediately

Validation:
- Bio: Maximum 500 characters
- Avatar: Image files only (jpg, png, max 5MB)
- Username: Must be unique
```

---

## Video Upload & Management

### US-005: Video Upload

**As a** creator  
**I want to** upload a video  
**So that** I can share content with viewers

**Acceptance Criteria:**

```gherkin
Given I am logged in as a creator
When I click "Upload Video"
Then I should see an upload modal

Given I am in the upload modal
When I drag and drop a video file
Or when I click "Select File" and choose a video
Then the file should start uploading
And I should see a progress bar
And I should be able to fill in metadata while uploading

Given the upload is complete
When I enter title, description, tags, and category
And I select visibility (public/unlisted/private)
And I click "Publish"
Then the video should be sent for processing
And I should see a success message
And I should be redirected to my dashboard

Validation:
- Title: 3-200 characters (required)
- Description: Max 5000 characters
- Tags: Max 10 tags
- File: MP4 or WebM, max 2GB
- Category: One of predefined options

Given the file is too large (>2GB)
When I try to upload
Then I should see an error: "File size exceeds 2GB limit"

Given the upload fails
When an error occurs
Then I should see an error message
And I should have the option to retry
```

---

### US-006: Video Processing Status

**As a** creator  
**I want to** see my video processing status  
**So that** I know when it's ready to watch

**Acceptance Criteria:**

```gherkin
Given I have uploaded a video
When I navigate to the video page
And the video is still processing
Then I should see "Processing..." status
And I should see an estimated time remaining
And the play button should be disabled

Given processing is complete
When I refresh the page
Then I should see "Ready" status
And I should be able to play the video
And I should receive a notification

Given processing fails
When an error occurs
Then I should see "Failed" status
And I should see an error reason
And I should have option to re-upload or contact support
```

---

### US-007: Edit Video Metadata

**As a** creator  
**I want to** edit my video metadata  
**So that** I can update information or fix mistakes

**Acceptance Criteria:**

```gherkin
Given I am viewing my own video
When I click "Edit"
Then I should see an edit form
With current title, description, tags, category, visibility

When I update any fields
And I click "Save Changes"
Then the video metadata should be updated
And I should see a success message
And the changes should be reflected immediately

Given I try to edit someone else's video
When I click "Edit"
Then I should see a "Forbidden" error
```

---

### US-008: Delete Video

**As a** creator  
**I want to** delete my videos  
**So that** I can remove content I no longer want public

**Acceptance Criteria:**

```gherkin
Given I am viewing my own video
When I click "Delete"
Then I should see a confirmation dialog
With warning: "This action cannot be undone"

When I confirm deletion
Then the video should be soft-deleted (status='deleted')
And I should be redirected to my dashboard
And the video should no longer appear in search or feeds
And the video files should be kept for 30 days (for recovery)

Given I try to delete someone else's video
Then I should see a "Forbidden" error
```

---

## Video Discovery & Playback

### US-009: Browse Video Feed

**As a** user  
**I want to** browse videos on the home page  
**So that** I can discover content

**Acceptance Criteria:**

```gherkin
Given I am on the home page
Then I should see a grid of video cards
With thumbnail, title, creator, views, and upload time

When I scroll to the bottom
Then more videos should load automatically (infinite scroll)

Given I am logged in
When I view the "For You" feed
Then I should see personalized video recommendations

Given I am not logged in
When I view the home feed
Then I should see trending/popular videos
```

---

### US-010: Search Videos

**As a** user  
**I want to** search for videos  
**So that** I can find specific content

**Acceptance Criteria:**

```gherkin
Given I am on any page
When I click the search bar
And I start typing a query
Then I should see autocomplete suggestions

When I submit the search
Then I should see relevant video results
Sorted by relevance by default

When I apply filters (category, duration, upload date)
Then results should update in real-time
Without page reload

Given no videos match my search
When I search for "nonexistent query"
Then I should see "No videos found" message
And suggested popular searches
```

---

### US-011: Watch Video

**As a** user  
**I want to** watch a video  
**So that** I can enjoy the content

**Acceptance Criteria:**

```gherkin
Given I click on a video card
Then I should be navigated to the video player page
And the video should start loading
And I should see video metadata (title, description, creator)

When the video is ready
Then it should auto-play (if user has autoplay enabled)
And I should see playback controls
And the view count should increment (once per user per hour)

When I click play/pause
Then the video should play/pause

When I seek to a timestamp
Then the video should jump to that position

When I click fullscreen
Then the video should enter fullscreen mode

When I adjust volume
Then the volume should change
And the setting should persist for next video

When the video ends
Then I should see recommended next videos
And the video should not loop by default
```

---

### US-012: Video Player Controls

**As a** viewer  
**I want to** control video playback  
**So that** I can customize my viewing experience

**Acceptance Criteria:**

```gherkin
Given I am watching a video
When I hover over the player
Then I should see the control bar

Available controls:
- Play/Pause button
- Progress bar (seekable)
- Volume control
- Quality selector (360p, 480p, 720p, 1080p)
- Playback speed (0.5x, 1x, 1.5x, 2x)
- Fullscreen toggle
- Picture-in-Picture

Keyboard shortcuts:
- Space/K: Play/pause
- F: Fullscreen
- M: Mute/unmute
- ←/→: Seek backward/forward 5s
- 0-9: Jump to percentage (0%-90%)

Given I select a quality option
When I click 720p
Then the video should switch to 720p
And playback should resume from same position
And my preference should be saved
```

---

## Engagement Features

### US-013: Like Video

**As a** logged-in user  
**I want to** like videos  
**So that** I can show appreciation

**Acceptance Criteria:**

```gherkin
Given I am watching a video
When I click the like button
Then the button should turn blue/active
And the like count should increase by 1
And my like should be saved

When I click the like button again
Then my like should be removed
And the button should return to inactive state
And the like count should decrease by 1

Given I am not logged in
When I click the like button
Then I should see a "Login required" message
And I should be prompted to sign in
```

---

### US-014: Comment on Video

**As a** logged-in user  
**I want to** comment on videos  
**So that** I can share my thoughts

**Acceptance Criteria:**

```gherkin
Given I am watching a video
When I scroll to the comments section
Then I should see existing comments
Sorted by recent or top (user selectable)

When I type a comment and click "Comment"
Then my comment should be posted
And it should appear at the top of the list
And the video's comment count should increase

Validation:
- Comment text: 1-1000 characters
- Rate limit: Max 20 comments per minute

Given I want to reply to a comment
When I click "Reply" on a comment
Then I should see a reply input box
And my reply should be threaded under the original comment

Given I am the comment author
When I click "Edit" on my comment
Then I should be able to update the text
And it should show "edited" indicator

Given I am the comment author or video creator
When I click "Delete" on a comment
Then the comment should be removed (soft delete)
And replies should remain visible
```

---

### US-015: Subscribe to Creator

**As a** viewer  
**I want to** subscribe to creators  
**So that** I can follow their content

**Acceptance Criteria:**

```gherkin
Given I am watching a creator's video
When I click "Subscribe"
Then I should be subscribed
And the button should change to "Subscribed"
And the creator's subscriber count should increase

When I click "Subscribed" again
Then I should be unsubscribed
And the button should change to "Subscribe"
And the creator's subscriber count should decrease

Given I am subscribed to a creator
When they upload a new video
Then I should receive a notification (if enabled)
And the video should appear in my "Subscriptions" feed

Given I am not logged in
When I click "Subscribe"
Then I should be prompted to log in
```

---

### US-016: Create Playlist

**As a** logged-in user  
**I want to** create playlists  
**So that** I can organize videos I like

**Acceptance Criteria:**

```gherkin
Given I am logged in
When I click "Create Playlist"
Then I should see a creation form

When I enter a name and description
And I select visibility (public/unlisted/private)
And I click "Create"
Then the playlist should be created
And I should be redirected to the empty playlist page

Given I am viewing a video
When I click "Save to playlist"
Then I should see my playlists
And I should be able to select one
When I select a playlist
Then the video should be added
And I should see a success message

Given I am viewing my playlist
When I drag videos to reorder them
Then the order should be saved
And playback should follow the new order
```

---

## Creator Dashboard

### US-017: View Analytics Overview

**As a** creator  
**I want to** see my analytics  
**So that** I can track my performance

**Acceptance Criteria:**

```gherkin
Given I am logged in as a creator
When I navigate to my dashboard
Then I should see an analytics overview:
- Total views (last 28 days)
- Total likes
- Total comments
- Subscriber count

And I should see a line chart:
- Views over time (daily breakdown)

And I should see top performing videos:
- Top 5 videos by views
- With thumbnails and view counts

When I select a different time range (7d, 30d, 90d, all)
Then the data should update accordingly
```

---

### US-018: View Video-Specific Analytics

**As a** creator  
**I want to** see analytics for individual videos  
**So that** I can understand what performs well

**Acceptance Criteria:**

```gherkin
Given I am viewing my own video
When I click "Analytics"
Then I should see detailed metrics:
- Total views
- Unique viewers
- Average watch duration
- Completion rate (% who watched to end)
- Like/dislike ratio
- Comment count

And I should see charts:
- Views over time (daily)
- Audience retention curve (drop-off points)
- Traffic sources (search, direct, recommendations)
- Viewer demographics (devices, countries)

When I hover over data points
Then I should see detailed tooltips
```

---

### US-019: Manage Videos

**As a** creator  
**I want to** manage all my videos in one place  
**So that** I can efficiently organize my content

**Acceptance Criteria:**

```gherkin
Given I am on my video manager page
Then I should see a list of all my videos
With thumbnail, title, status, views, upload date

When I search by title
Then the list should filter in real-time

When I filter by status (ready/processing/failed)
Then only matching videos should show

When I sort by (recent/views/likes)
Then the list should reorder accordingly

When I click "Edit" on a video
Then I should see the edit form

When I click "Delete" on a video
Then I should see a confirmation dialog

When I select multiple videos
Then I should see bulk actions:
- Delete selected
- Change visibility
- Add to playlist
```

---

## Monetization

### US-020: Enable Monetization (Creator)

**As a** creator  
**I want to** enable monetization on my channel  
**So that** I can earn money from my content

**Acceptance Criteria:**

```gherkin
Given I am a creator with 1000+ subscribers
When I navigate to "Monetization" settings
Then I should see eligibility status

When I click "Enable Monetization"
Then I should be prompted to connect Stripe
And provide tax information

When I complete Stripe setup
Then my account should be approved for monetization
And I should see monetization options:
- Ad revenue sharing
- Channel memberships
- Tips/donations
- Pay-per-view videos

Given I don't meet requirements (< 1000 subs)
When I try to enable monetization
Then I should see eligibility requirements
And my current progress
```

---

### US-021: Set Up Channel Membership

**As a** creator  
**I want to** offer channel memberships  
**So that** I can earn recurring revenue

**Acceptance Criteria:**

```gherkin
Given I have monetization enabled
When I navigate to "Memberships" settings
Then I should be able to create membership tiers

When I create a tier:
- Name (e.g., "Basic", "Premium", "Ultra")
- Price ($2.99, $4.99, $9.99 per month)
- Benefits (custom text)
And I click "Create"
Then the tier should be added
And it should appear on my channel page

Given a user wants to join
When they select a tier and click "Join"
Then they should be redirected to Stripe checkout
And after payment, they should become a member
And they should receive member badge on comments

Given a member wants to cancel
When they click "Cancel Membership"
Then they should remain a member until period ends
And they should not be charged next month
```

---

### US-022: Receive Tips/Donations

**As a** creator  
**I want to** receive tips from viewers  
**So that** I can be supported directly

**Acceptance Criteria:**

```gherkin
Given I have monetization enabled
When viewers watch my videos
Then they should see a "Tip" button

When a viewer clicks "Tip"
Then they should see tip amount options:
- $1, $5, $10, $25, $50, Custom
And a message input (optional)

When they complete payment
Then I should receive 95% of the amount
And the platform takes 5% fee
And I should see the tip in my earnings dashboard

When I reach minimum payout ($50)
Then I should be able to request payout to bank
And I should receive funds within 7 days
```

---

### US-023: View Earnings Dashboard

**As a** creator  
**I want to** track my earnings  
**So that** I know how much I'm making

**Acceptance Criteria:**

```gherkin
Given I have monetization enabled
When I navigate to "Earnings" dashboard
Then I should see:
- Total revenue (lifetime)
- Revenue breakdown:
  * Ad revenue
  * Membership revenue
  * Tips/donations
  * Pay-per-view
- Pending balance
- Available for payout

And I should see earnings chart:
- Revenue over time (daily/monthly)

When I click "Request Payout"
Then I should see payout form
With bank details (pre-filled from Stripe)
And expected transfer date

Validation:
- Minimum payout: $50
- Payout frequency: Weekly or Monthly
```

---

**End of User Stories**

*These user stories serve as the contract between product requirements and implementation. Each story should be translated into code that passes all acceptance criteria.*
