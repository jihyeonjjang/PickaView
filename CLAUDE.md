# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PickaView is an iOS video discovery app that fetches videos from the Pixabay API and provides personalized recommendations based on user watch behavior. The app features video playback with support for multiple device orientations (iPhone/iPad, portrait/landscape), user statistics, and a tag-based preference learning system.

**Tech Stack:** Swift, UIKit, CoreData, AVFoundation, URLSession (async/await), Combine
**Dependencies:** DGCharts (charts), SkeletonView (loading states) - managed via Swift Package Manager

## Build and Run Commands

### Building the Project
```bash
# Build for any iOS device
xcodebuild -scheme PickaView -configuration Debug -destination 'generic/platform=iOS'

# Build for iOS simulator
xcodebuild -scheme PickaView -configuration Debug -destination 'platform=iOS Simulator,name=iPhone 15'

# Clean build folder
xcodebuild clean -scheme PickaView
```

### Running in Simulator
Open the project in Xcode and run:
```bash
open PickaView.xcodeproj
# Then press Cmd+R in Xcode, or use xcodebuild with -destination for specific simulator
```

### API Key Configuration
The app requires a Pixabay API key configured as an environment variable:
- Add `PIXABAY_API_KEY` to the Xcode scheme's environment variables
- The key is read from `Info.plist` at runtime via `$(PIXABAY_API_KEY)` substitution
- Without a valid key, the app will fail to fetch videos (handled by NetworkError.missingAPIKey)

### Testing
Currently, there are no unit tests in the project. When adding tests:
```bash
# Run all tests
xcodebuild test -scheme PickaView -destination 'platform=iOS Simulator,name=iPhone 15'

# Run tests and show output
xcodebuild test -scheme PickaView -destination 'platform=iOS Simulator,name=iPhone 15' | xcpretty
```

## Architecture

### High-Level Pattern: MVVM
- **Models:** CoreData entities (Video, Tag, TimeStamp, History) + Network DTOs (PixabayVideo)
- **Views:** ViewControllers + Storyboards (.storyboard files)
- **ViewModels:** Business logic, data transformation, CoreData/Network interactions
- **Dependency Injection:** CoreDataManager singleton is created in TabBarViewModel and passed down

### Key Architectural Layers

#### 1. Data Layer (`PickaView/Data/`)

**Persistence (`Data/Persistence/CoreDataManager.swift`):**
- Singleton managing NSPersistentContainer with model name "Model" (Model.xcdatamodeld)
- Main context operations on `mainContext` (viewContext)
- Core entities:
  - **Video:** id, username, userImageURL, pageURL, videoURL, tags (relationship), duration, views, downloads, likes, comments, isLiked, imageURL
  - **Tag:** name, score (Float, decays over 7 days), videos (relationship)
  - **TimeStamp:** startTime (Date), video (relationship) - tracks when user started watching
  - **History:** watchedDate (Date), duration (Int32) - daily watch time aggregation

**Smart Sync Logic:**
- `saveVideos(_:)` updates existing entities, inserts new ones, and deletes removed videos
- Only updates entities when data actually changes (guard clauses prevent unnecessary saves)
- Tag management: parses comma-separated tag strings, creates/reuses Tag entities, removes orphaned tags

**Network (`Data/Network/`):**
- **APIClient.swift:** Generic HTTP client with async/await and Decodable support
- **PixabayVideoService.swift:** Pixabay API integration using Endpoint pattern
  - `fetchVideos(page:perPage:)` returns `[PixabayVideo]`
  - Pagination support (default 20 per page)
- **NetworkError.swift:** Typed errors (invalidURL, requestFailed, invalidResponse, noData, decodingFailed, missingAPIKey)

**UserDefaults (`Data/UserDefault/ThemeManager.swift`):**
- Theme enum: `.light`, `.dark`, `.system`
- Singleton pattern, persists user preference

#### 2. View Layer (`PickaView/Views/`)

Organized by tab/feature:

**Home Tab (`Views/Home/`):**
- **HomeViewController:** Main feed with infinite scroll, search, pull-to-refresh
  - Grid layout adapts to device/orientation (uses `UICollectionViewFlowLayout`)
  - Skeleton loading states (SkeletonView)
  - Search bar filters by tag (tag name matching)
- **HomeViewModel:** Fetches videos, sorts by recommendation score, manages pagination
  - Calls `VideoRecommender.sortVideosByRecommendationScore()` before displaying

**Like Tab (`Views/Like/`):**
- **LikeViewController:** Liked videos grid using `UICollectionViewDiffableDataSource`
  - NSFetchedResultsController (FRC) for real-time CoreData updates
  - Predicate: `isLiked == true`
  - Pagination support (20 per page)
- **LikeViewModel:** FRC-based fetching, toggle like/unlike

**MyPage Tab (`Views/MyPage/`):**
- **MyPageViewController:** User stats and settings (100% programmatic UI with UIStackView)
  - DGCharts bar chart showing last 7 days watch time
  - Today/weekly watch statistics (formatted as "Xh Ym")
  - Recently watched videos horizontal scroll (last 10 videos)
  - Theme switcher (UISegmentedControl)
- **MyPageViewModel:** Dual FRC setup (videos + history), Combine publishers for reactive updates

**Player Module (`Views/Player/`):**
This is a sophisticated multi-screen architecture with shared components:

**Shared Base Class:**
- **BasePlayerViewController:** Abstract base class for all player screens
  - Common UI controls (play/pause button, seek slider, labels)
  - Gesture handling integration (delegates to PlayerGestureHandler)
  - AVPlayer lifecycle (setup, cleanup, observers)
  - Control auto-hide logic (hides after 3 seconds of inactivity)
  - Watch time tracking (Timer-based, pauses on seek/background)

**Shared Components:**
- **PlayerViewModel:** Business logic (like status, watch time tracking, save history on exit)
- **PlayerManager:** AVPlayer wrapper (playback control, time observers, seek, KVO for status)
- **PlayerGestureHandler:** Gesture recognition
  - Single tap: toggle controls
  - Double tap left/right: seek ±10 seconds
  - Long press: 2x speed playback
  - Swipe up: enter fullscreen
  - Swipe down: dismiss player

**Concrete Player Implementations:**
- **PlayerViewController** (`Views/Player/Default/`): Standard iPhone portrait player
  - Recommendation carousel at bottom
  - Auto-switches to fullscreen on landscape rotation

- **IPadLandscapeViewController** (`Views/Player/IPad/`): iPad landscape layout
  - Split view: video player on left, recommended videos list on right
  - User info display (avatar, username, views, likes)
  - Auto-switches to portrait player on rotation

- **FullscreenPlayerViewController** (`Views/Player/FullScreen/`): Landscape fullscreen
  - Minimal UI, full-screen video
  - Force landscape orientation
  - Returns to previous screen on portrait rotation

**Player Transitions:**
- Device orientation notifications trigger screen transitions
- PlayerLayer and PlayerManager instances are transferred between view controllers
- Maintains playback state during transitions

#### 3. Recommendation System (`PickaView/Recommendation/`)

**VideoRecommender.swift:**
- `sortVideosByRecommendationScore(videos:allTags:likedVideoIDs:)` → sorted videos
- For each video, calls `RecommendationScorer.calculateRecommendationScore()`

**RecommendationScorer.swift:**
Algorithm combines three factors:
1. **Tag Score (60% weight):**
   - Average score of video's tags from user's tag preferences
   - Tags have scores that increase with watch time (must watch >15%) and likes
   - Time decay: `score * exp(-daysSinceLastUpdate / 7)` (7-day half-life)
2. **Like Boost (10% weight):** +5 if video is liked
3. **Popularity (30% weight):** Normalized from views, downloads, comments

Formula: `(tagScore * 0.6) + (likeBoost * 0.1) + (popularity * 0.3)`

**Tag Score Updates:**
- Triggered in `PlayerViewModel.stopAndSaveWatching()`
- Only updates if user watched >15% of video duration
- Increments tag scores based on watch percentage
- Liking a video adds +1.0 to all its tags' scores

## Key Data Flows

### Video Playback Flow
1. User taps video → HomeViewController creates PlayerViewModel
2. PlayerViewController receives viewModel, playerManager, gestureHandler (via init)
3. PlayerManager sets up AVPlayer with video URL
4. PlayerViewModel.updateStartTime() saves timestamp to CoreData (TimeStamp entity)
5. Timer tracks watch duration (pauses on seek/background/pause)
6. Gestures handled by PlayerGestureHandler → delegate methods in BasePlayerViewController
7. On exit/dismiss: PlayerViewModel.stopAndSaveWatching()
   - Updates tag scores if watched >15%
   - Updates/creates History entity for today's watch time
8. CoreDataManager persists changes
9. FRCs in other views automatically update UI

### Network to CoreData Sync
1. HomeViewModel calls `PixabayVideoService.fetchVideos(page:perPage:)`
2. APIClient sends URLRequest, decodes JSON to `[PixabayVideo]`
3. CoreDataManager.saveVideos() performs smart sync:
   - Fetches existing videos by ID
   - Updates changed entities, inserts new ones
   - Deletes videos not in new response (keeps liked ones)
   - Parses tag strings, creates/reuses Tag entities
4. FRCs trigger UI updates in Like/MyPage tabs

### Recommendation Calculation
1. HomeViewModel calls `VideoRecommender.sortVideosByRecommendationScore()`
2. For each video: `RecommendationScorer.calculateRecommendationScore()`
3. Fetches all user tag scores from CoreData (with time decay applied)
4. Combines tag scores, like boost, popularity → final score
5. Videos sorted by score (descending), pagination applied
6. HomeViewController displays sorted results

## Common Patterns

### NSFetchedResultsController (FRC) Usage
- Used in LikeViewController and MyPageViewController for real-time updates
- Created via `FRCFactory.createVideoFRC()` or manually
- Delegate methods (`controllerDidChangeContent`) trigger UI updates
- Example predicates:
  - Liked videos: `isLiked == true`
  - Videos with timestamps: `timeStamp != nil` sorted by `timeStamp.startTime`

### UICollectionViewDiffableDataSource
- Modern approach in LikeViewController
- Snapshot-based updates (no manual insertions/deletions)
- Works seamlessly with FRC via `performBatchUpdates` → `apply(snapshot)`

### Dependency Injection
- CoreDataManager singleton created in TabBarViewModel
- Passed to each tab's ViewController/ViewModel via initializers
- Ensures all views share same CoreData context

### Image Caching
- ImageCacheManager (singleton) uses NSCache
- Download images asynchronously, cache by URL string key
- Set images in cells: `imageView.image = await ImageCacheManager.shared.loadImage(from: url)`

### Storyboard Instantiation
- Multiple storyboards: Main, Player, IPadLandscape, FullScreen, etc.
- Instantiate by identifier: `UIStoryboard(name: "Player", bundle: nil).instantiateViewController(withIdentifier: "PlayerViewController")`
- Pass dependencies after instantiation (before presenting)

## Important Implementation Notes

### CoreData Context Safety
- All CoreData operations use `mainContext` (main queue)
- Some operations wrapped in `mainContext.perform {}` or `withCheckedContinuation`
- Always save context after mutations: `try mainContext.save()`

### Watch Time Tracking
- Timer-based (not AVPlayer time) for accuracy
- Only counts actual watched time (pauses on seek/background/pause)
- Minimum 15% watch threshold to affect tag scores
- History entity aggregates by date (one entry per day)

### Tag Score Decay
- Applied during recommendation calculation (not stored)
- Formula: `currentScore * exp(-daysSinceLastUpdate / 7)`
- Prevents old preferences from dominating recommendations
- Scores never go to zero (exponential decay asymptotes)

### Orientation Handling
- NotificationCenter observers for `UIDevice.orientationDidChangeNotification`
- BasePlayerViewController checks orientation → switches view controller
- PlayerLayer and PlayerManager transferred during transitions
- Specific orientations forced in viewWillAppear (e.g., FullscreenPlayerViewController locks landscape)

### Pagination
- Home/Like tabs load 20 items per page
- Tracks current page, increments on scroll to bottom
- Skeleton loading states shown while fetching

### Memory Management
- ImageCacheManager uses NSCache (automatic memory pressure handling)
- FRCs deallocate automatically with view controllers
- PlayerManager cleaned up in deinit (removes observers, pauses player)

## File Locations Reference

- **CoreData Model:** `PickaView/Model.xcdatamodeld/Model.xcdatamodel/`
- **CoreDataManager:** `PickaView/Data/Persistence/CoreDataManager.swift`
- **Networking:** `PickaView/Data/Network/`
- **Player Shared Components:** `PickaView/Views/Player/Shared/`, `PickaView/Views/Player/Common/`
- **Recommendation Algorithm:** `PickaView/Recommendation/RecommendationScorer.swift`
- **Tab Bar Root:** `PickaView/TabBarViewController.swift`
- **Theme Manager:** `PickaView/Data/UserDefault/ThemeManager.swift`
- **Extensions:** `PickaView/Extentions/` (note: "Extentions" is misspelled in directory name)

## Development Workflow

### Adding a New Feature
1. Determine if it's a new tab/screen or enhancement to existing
2. If new screen: create ViewModel first (business logic), then ViewController (UI)
3. Pass CoreDataManager via dependency injection
4. Use FRC for CoreData-backed lists (automatic updates)
5. Follow MVVM pattern: View ← ViewModel → Model

### Modifying CoreData Schema
1. Edit `Model.xcdatamodeld` in Xcode Data Model Editor
2. Create new model version if needed (Editor → Add Model Version)
3. Update CoreDataManager fetch/save methods
4. Consider migration strategy for existing users

### Modifying Player Behavior
- Common logic: Edit `BasePlayerViewController`
- Screen-specific: Edit concrete implementations (PlayerViewController, etc.)
- Gesture behavior: Edit `PlayerGestureHandler`
- AVPlayer operations: Edit `PlayerManager`
- Business logic: Edit `PlayerViewModel`

### Modifying Recommendation Algorithm
- Weights/formula: Edit `RecommendationScorer.calculateRecommendationScore()`
- Decay rate: Change `7` in `exp(-days/7)` to different half-life
- Tag score updates: Edit `PlayerViewModel.stopAndSaveWatching()`
- Minimum watch threshold: Change `0.15` in `watchedPercentage > 0.15` check
