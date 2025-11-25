# Vertical Video Feed Performance (Jetpack Compose + ExoPlayer/Media3)

This repo discusses the requirements for a **working, optimized implementation** of a list-style media feed built with:
- Jetpack Compose
- LazyColumn
- ExoPlayer/Media3
- AndroidView interop

This applies to feeds like **Instagram, Reddit, Pinterest, YouTube** — where each row may contain **video or mixed media**, not the “one item per screen” TikTok/Reels model.

The goal:  
**Create a scalable media architecture that performs well on both mid-tier and flagship phones.**

Test devices:
- **Samsung A25 (mid-tier)**
- **Samsung S25 (flagship)**

This README covers:
- The **behavior** of the optimized feed on both devices  
- The **architecture** (pooling, preloading, thumbnail assist)
- The **tradeoffs & lessons learned** using ExoPlayer inside a `LazyColumn + AndroidView` environment

---

## 1. Device Demo Videos (Optimized Implementation)

Both videos below use the **same architecture**.

<table>
  <tr>
    <td align="center">
      <strong>Samsung A25 — Optimized</strong><br><br>
      <video src="https://github.com/user-attachments/assets/b0ee704f-b289-4ca1-9fd4-9ee58166b069"
        width="400"
        controls>
      </video> 
    </td>
    <td align="center">
      <strong>Samsung S25 — Optimized</strong><br><br>
      <video src="https://github.com/user-attachments/assets/06420aee-384d-4c64-a445-843cc276149a"
        width="400"
        controls>
      </video>
    </td>
  </tr>
</table>

---

## 2. Why a Media Feed Is Harder Than It Looks

A media feed may appear simple, but in practice it’s significantly more complex than rendering text or images.

Key challenges developers face:

### **Player/Surface Lifecycle**
- Ensuring the correct player is attached when items enter/exit view
- Managing surface reuse without flickers or black frames

### **Performance & Jank**
- Avoiding player creation/destruction during scroll
- Maintaining 60fps while preloading upcoming media

### **Media Initialization Issues**
- Black/blank frames before first frame rendered  
- Incorrect video appearing due to async surface/player attachment

### **Threading Constraints**
- ExoPlayer and surfaces must operate consistently on the **same thread**
- Incorrect threading → crashes, OOMs, undefined state

### **Lack of Public Examples**
- Few end-to-end examples exist for **Compose + ExoPlayer + LazyColumn**

---

## 3. Optimized Architecture Overview

This implementation focuses on **scalability**, **smoothness**, and **resource efficiency**:

### **3.1 Surface / View Pool**

To display video, ExoPlayer needs a `SurfaceView` or `TextureView`.

Instead of creating/destroying these for each list item:
- Maintain a pool using an `ArrayDeque`
- Acquire via `surfacePool.acquire()`
- Release via `surfacePool.releaseToPool()`
- Ensure main-thread access with `ensureMainThread()`

Lifecycle binding happens inside `AndroidView`:
- **factory** → acquire a view  
- **onReset** → recycle view safely  
- **onRelease** → return to pool  

This reduces view churn and prevents black flickers.

---

### **3.2 Player Pool**

A pool of ExoPlayers avoids repeating:
- Decoder initialization
- Renderer creation
- Track selection
- Buffer pipeline initialization

Key constraints:
- Players must stay on the **same thread** they were created on  
- Pool size max ~5 (practical due to decoder count & 60fps budget)

Best practices:
- Use `player.pause()` (not stop) to retain codecs  
- Allow PreloadManager to prepare buffers efficiently  
- Add listeners for:
  - First frame rendered
  - Player state changes
  - Errors
  - Analytics

---

### **3.3 Viewport-Based Preloading Window**

Use a **sliding window** around the active (centered) item:

Example:
- Active index: `i`
- Preload radius: `r`
- Window: `[i - r, …, i, …, i + r]`

Behavior:
- **Active item:** playing  
- **Neighbors:** prepared + buffered  
- **Others:** detached from players/surfaces  

This ensures instant playback during fast scrolls.

---

## 4. Separation of Concerns

### **ViewModel**
- Holds list of media items (URLs, thumbnails)
- Tracks:
  - Active index (center)
  - Preload window
- Provides references to:
  - `SurfacePool`
  - `PlayerPool`

### **PreloadManager**
- Builds ExoPlayers with:
  - `LoadControl`
  - `RenderersFactory`
  - `CacheDataSource`
  - `BandwidthMeter`
  - `TrackSelector`
- Implements `PreloadStatusControl` to stage buffers & prepare media

### **SurfacePool**
- Provides pooled `View` instances
- Ensures main-thread reuse
- Reduces allocation & churn

### **PlayerPool**
- Creates & manages ExoPlayers (up to max pool size)
- Handles first frame, play/pause state, analytics, and errors

### **UI (Compose + LazyColumn + AndroidView)**
- Displays the items
- Reports which index is centered
- Delegates all media-related logic to:
  - PlayerPool
  - SurfacePool
  - PreloadManager

### **Issues this solve**
- https://github.com/androidx/media/issues/2493
- https://github.com/androidx/media/issues/2018





