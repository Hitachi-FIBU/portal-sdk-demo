# HASPortal Web Integration Guide

A complete guide for integrating web applications with the HASPortal SDK's event bridge system.

## Table of Contents

- [Quick Start](#quick-start)
- [Event System Overview](#event-system-overview)
- [Available Events](#available-events)
- [Usage Examples](#usage-examples)
- [Best Practices](#best-practices)
- [Debugging](#debugging)
- [Quick Reference](#quick-reference)

---

## Quick Start

### Basic Setup

```javascript
// 1. Listen for responses from native
document.addEventListener('cap-events-response', (event) => {
    const response = event.detail

    // Handle camera permission
    if (response.name === 'camera-permission-result') {
        console.log('Camera access:', response.granted)
    }

    // Handle photos
    else if (response.photos) {
        console.log('Photos:', response.photos)
    }
})

// 2. Send events to native
function closePortal() {
    document.dispatchEvent(new CustomEvent('cap-events', {
        detail: { name: 'close' }
    }))
}
```

---

## Event System Overview

The HASPortal SDK uses **CustomEvents** for bidirectional communication between your web app and native features.

### Two Event Channels

**Send to Native: `cap-events`**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: {
        name: 'event-name',
        // ...parameters
    }
}))
```

**Receive from Native: `cap-events-response`**
```javascript
document.addEventListener('cap-events-response', (e) => {
    console.log('Response:', e.detail)
})
```

### Key Concept

- Your web app **dispatches** `cap-events` to trigger native functionality
- Your web app **listens** to `cap-events-response` to receive results

---

## Available Events

### 1. Close Portal

Close the current WebView. May optionally include an `activityPerformed` array that names server-confirmed activities the user completed — useful for letting parent pages (in stacked-portal scenarios) refresh selectively instead of blindly.

**Send (no activity — browsed and closed):**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: { name: 'close' }
}))
```

**Send (with activity — user completed one or more server-confirmed actions):**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: {
        name: 'close',
        activityPerformed: ['points-redeemed']   // single activity
    }
}))

// Or multiple (e.g. redemption that crossed a tier threshold)
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: {
        name: 'close',
        activityPerformed: ['points-redeemed', 'tier-upgraded']
    }
}))
```

**Helper:**
```javascript
const closePortal = (activityPerformed = []) => {
    document.dispatchEvent(new CustomEvent('cap-events', {
        detail: { name: 'close', activityPerformed }
    }))
}
```

**Receive on this WebView:** None (portal closes)

**Receive on parent WebView (stacked portals only):** `cap-events-response { name: 'child-portal-closed', url, detail? }` — see [Event 8: Child Portal Closed](#8-child-portal-closed-parent-webview).

**Payload contract:**
- Only list server-confirmed activities — `'points-redeemed'` means the redemption API returned 2xx, not "user tapped Redeem".
- No duplicates in the array.
- Keep the enum closed; define valid strings centrally with the team.
- Never put PII, tokens, or PAN in `detail`.

---

### 2. Open Stacked Portal

Open a new portal on top of the current one. Event listeners are automatically inherited.

**Send:**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: {
        name: 'open',
        url: 'https://example.com/page',
        toolbarType: 'navigation',  // 'blank' | 'navigation' | 'activity' | 'progress'
        showHeader: true,
        title: 'Page Title'
    }
}))
```

**Receive:** None (new portal opens)

**Available Options:**
- `url` (required) - URL to load
- `toolbarType` - Toolbar style
- `showHeader` - Show/hide header bar
- `title` - Portal title text
- `toolbarColor` - Hex color (e.g., `'#007AFF'`)
- `showReloadButton` - Show reload button
- `closeModal` - Show confirmation on close

---

### 3. Camera Permission

Request camera permission when blocked.

**Send:**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: {
        name: 'camera-blocked',
        reason: 'permission-denied'  // optional
    }
}))
```

**Receive:**
```javascript
{
    name: 'camera-permission-result',
    granted: true  // or false
}
```

**Example:**
```javascript
document.addEventListener('cap-events-response', (e) => {
    if (e.detail.name === 'camera-permission-result') {
        if (e.detail.granted) {
            startCamera()
        } else {
            showError('Camera permission denied')
        }
    }
})
```

---

### 4. Photo Library

Open native photo picker.

**Send:**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: {
        name: 'photo-library',
        selectionLimit: 5,     // Max number of photos
        filter: 'images'       // Filter type
    }
}))
```

**Receive (Success):**
```javascript
{
    success: true,
    cancelled: false,
    photos: [
        {
            dataUrl: 'data:image/jpeg;base64,...',  // Android
            data: 'data:image/jpeg;base64,...',     // iOS
            format: 'jpeg',                         // Android
            type: 'image/jpeg',                     // iOS
            filename: 'photo.jpg',                  // iOS only
            size: 45678                             // iOS only
        }
    ]
}
```

**Receive (Cancelled):**
```javascript
{
    success: false,
    cancelled: true,
    photos: []
}
```

---

### 5. PDF Viewer

Open PDF in native viewer.

**Send:**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: {
        name: 'intent',
        type: 'application/pdf',
        url: 'https://example.com/document.pdf',
        title: 'Document Title'
    }
}))
```

**Receive:** None (PDF viewer opens)

---

### 6. Application Error

Show native error view with retry button.

**Send:**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: { name: 'application-error' }
}))
```

**Receive:** None

---

### 7. Get Viewport Info (Safe Area)

Request current safe area insets and viewport information on-demand. Useful for responsive layouts that need to account for the notch, Dynamic Island, or home indicator.

**Send:**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: { name: 'get-viewport-info' }
}))
```

**Receive:**
```javascript
{
    event: 'viewport-info-response',
    safeAreaInsets: {
        top: 59,        // Notch / Dynamic Island area
        bottom: 34,     // Home indicator area
        left: 0,
        right: 0
    },
    viewportInfo: {
        availableHeight: 719,       // Total height minus safe areas
        contentHeight: 666.5,       // Available height minus custom header
        customHeaderHeight: 52.5,   // Height of custom header (if present)
        safeAreaInsets: {            // Same as top-level safeAreaInsets
            top: 47,
            bottom: 34,
            left: 0,
            right: 0
        },
        hasCustomHeader: true       // Whether a custom header is displayed
    }
}
```

**Platform Support:** iOS only

**Example:**
```javascript
function getViewportInfo() {
    return new Promise((resolve) => {
        const handler = (event) => {
            if (event.detail.event === 'viewport-info-response') {
                document.removeEventListener('cap-events-response', handler)
                resolve(event.detail)
            }
        }

        document.addEventListener('cap-events-response', handler)

        document.dispatchEvent(new CustomEvent('cap-events', {
            detail: { name: 'get-viewport-info' }
        }))
    })
}

// Usage
getViewportInfo().then(({ safeAreaInsets, viewportInfo }) => {
    console.log('Safe area:', safeAreaInsets)
    document.body.style.paddingTop = `${safeAreaInsets.top}px`
    document.body.style.paddingBottom = `${safeAreaInsets.bottom}px`
})
```

**Notes:**
- Returns current values even after orientation changes or dynamic content updates
- Safe area insets reflect device-specific areas (notch, Dynamic Island, home indicator)

---

### 8. Child Portal Closed (parent WebView)

When a stacked child portal dismisses, the SDK dispatches a `cap-events-response` event into the **parent** portal's WebView. Lets the parent page react to activity completed in the child — e.g. refresh loyalty points after a voucher redemption.

**Send:** None — fired automatically by the SDK when a child portal dismisses (any close path: web `close`, hardware back, close modal, nav close button).

**Receive (on parent WebView):**
```javascript
{
    name: 'child-portal-closed',
    url: 'https://example.com/child',       // child's final URL
    detail: {                                // echoes the child's original close detail
        name: 'close',
        activityPerformed: ['points-redeemed']
        // ...any other keys the child attached to detail
    }
}
```

**`detail` presence:**
- Present for **web-initiated** closes (child dispatched `cap-events { name: 'close', ... }`).
- **Absent** for native-initiated closes (hardware back, close modal, nav close button) — only `name` and `url` are included.

**Example — selective refresh by activity:**
```javascript
const ACTIVITY_TO_SCOPES = {
    'points-redeemed': ['points'],
    'points-earned':   ['points'],
    'tier-upgraded':   ['points', 'tier']
}

const FETCHERS = {
    points: fetchLoyaltyInfo,
    tier:   fetchTierInfo
}

document.addEventListener('cap-events-response', (e) => {
    if (e?.detail?.name !== 'child-portal-closed') return

    const performed = e.detail.detail?.activityPerformed ?? []
    if (performed.length === 0) return   // browsed-and-closed, nothing to refresh

    // Flatten activities → scopes, dedupe, fetch once each
    const scopes = new Set()
    performed.forEach(a => ACTIVITY_TO_SCOPES[a]?.forEach(s => scopes.add(s)))
    Promise.all([...scopes].map(s => FETCHERS[s]?.())).catch(console.error)
})
```

**Platform Support:**
- **Android:** fully supported from SDK v0.7.
- **iOS:** fully supported from SDK v0.6.

---

## Usage Examples

### Photo Upload with Progress

```javascript
// Setup listener
document.addEventListener('cap-events-response', (e) => {
    if (e.detail.photos !== undefined) {
        if (e.detail.cancelled) {
            console.log('User cancelled')
            return
        }

        if (e.detail.success) {
            uploadPhotos(e.detail.photos)
        } else {
            showError('Failed to select photos')
        }
    }
})

// Trigger photo picker
function selectPhotos(maxPhotos = 5) {
    document.dispatchEvent(new CustomEvent('cap-events', {
        detail: {
            name: 'photo-library',
            selectionLimit: maxPhotos,
            filter: 'images'
        }
    }))
}

// Upload function with progress
async function uploadPhotos(photos) {
    const total = photos.length
    let uploaded = 0

    for (const photo of photos) {
        const imageData = photo.dataUrl || photo.data

        try {
            await uploadToServer(imageData)
            uploaded++
            updateProgress(uploaded, total)
        } catch (error) {
            console.error('Upload failed:', error)
        }
    }

    console.log(`Uploaded ${uploaded}/${total} photos`)
}

async function uploadToServer(base64Image) {
    const response = await fetch('/api/upload', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ image: base64Image })
    })

    if (!response.ok) {
        throw new Error('Upload failed')
    }

    return response.json()
}

function updateProgress(current, total) {
    const percent = Math.round((current / total) * 100)
    console.log(`Upload progress: ${percent}%`)
    // Update your UI here
}
```

---

### Stacked Portal Navigation

```javascript
// Navigate to detail page
function openDetailPage(itemId) {
    const url = `https://example.com/details/${itemId}`

    document.dispatchEvent(new CustomEvent('cap-events', {
        detail: {
            name: 'open',
            url: url,
            toolbarType: 'navigation',
            toolbarColor: '#007AFF',
            showReloadButton: true,
            showHeader: true,
            title: 'Item Details',
            closeModal: true
        }
    }))
}

// Close current portal
function goBack() {
    document.dispatchEvent(new CustomEvent('cap-events', {
        detail: { name: 'close' }
    }))
}

// Navigate with confirmation
function openWithConfirmation(url, title) {
    if (confirm(`Open ${title}?`)) {
        document.dispatchEvent(new CustomEvent('cap-events', {
            detail: {
                name: 'open',
                url: url,
                title: title,
                toolbarType: 'navigation',
                closeModal: true
            }
        }))
    }
}
```

---

### Camera Permission Handler

```javascript
// Setup listener
document.addEventListener('cap-events-response', (e) => {
    if (e.detail.name === 'camera-permission-result') {
        if (e.detail.granted) {
            initializeCamera()
        } else {
            showCameraUnavailable()
        }
    }
})

// Request camera permission
function requestCameraPermission() {
    document.dispatchEvent(new CustomEvent('cap-events', {
        detail: {
            name: 'camera-blocked',
            reason: 'permission-denied'
        }
    }))
}

// Initialize camera
function initializeCamera() {
    navigator.mediaDevices.getUserMedia({ video: true })
        .then(stream => {
            document.querySelector('video').srcObject = stream
        })
        .catch(err => {
            console.error('Camera error:', err)
            // If blocked, request permission
            requestCameraPermission()
        })
}

function showCameraUnavailable() {
    alert('Camera access is required for this feature. Please enable it in your device settings.')
}
```

---

### Open PDF Document

```javascript
function openPDFDocument(pdfUrl, documentTitle) {
    document.dispatchEvent(new CustomEvent('cap-events', {
        detail: {
            name: 'intent',
            type: 'application/pdf',
            url: pdfUrl,
            title: documentTitle
        }
    }))
}

// Usage
document.querySelector('.view-pdf-button').addEventListener('click', () => {
    openPDFDocument(
        'https://example.com/documents/report.pdf',
        'Monthly Report'
    )
})
```

---

## Best Practices

### 1. Setup Listeners Before Triggering Events

Always register event listeners **before** dispatching events.

```javascript
// ✅ CORRECT
document.addEventListener('cap-events-response', handleResponse)
triggerEvent()

// ❌ WRONG - may miss response
triggerEvent()
document.addEventListener('cap-events-response', handleResponse)
```

---

### 2. Always Check Event Type

Verify the event type before processing responses.

```javascript
document.addEventListener('cap-events-response', (e) => {
    // Check by 'name' field
    if (e.detail.name === 'camera-permission-result') {
        handleCamera(e.detail.granted)
    }

    // Check by data presence
    else if (e.detail.photos !== undefined) {
        handlePhotos(e.detail.photos)
    }
})
```

---

### 3. Handle Success and Failure

Always handle both success and error cases.

```javascript
document.addEventListener('cap-events-response', (e) => {
    if (e.detail.photos !== undefined) {
        if (e.detail.success) {
            processPhotos(e.detail.photos)
        } else {
            showError('Failed to select photos')
            logEvent('photo_selection_failed')
        }
    }
})
```

---

### 4. Check for User Cancellation

Users can cancel pickers - always check the `cancelled` field.

```javascript
document.addEventListener('cap-events-response', (e) => {
    if (e.detail.photos !== undefined) {
        if (e.detail.cancelled) {
            console.log('User cancelled')
            return
        }

        if (e.detail.success) {
            processPhotos(e.detail.photos)
        } else {
            showError('Failed to select photos')
        }
    }
})
```

---

### 5. Clean Up Event Listeners

Remove listeners when components unmount to prevent memory leaks.

```javascript
// Vanilla JavaScript
const handler = (e) => { /* ... */ }
document.addEventListener('cap-events-response', handler)
// Later...
document.removeEventListener('cap-events-response', handler)

// React
useEffect(() => {
    const handler = (e) => { /* ... */ }
    document.addEventListener('cap-events-response', handler)

    return () => {
        document.removeEventListener('cap-events-response', handler)
    }
}, [])

// Vue
mounted() {
    this.handler = (e) => { /* ... */ }
    document.addEventListener('cap-events-response', this.handler)
}
beforeUnmount() {
    document.removeEventListener('cap-events-response', this.handler)
}
```

---

### 6. Handle Platform Differences

Photo responses vary slightly between Android and iOS.

```javascript
document.addEventListener('cap-events-response', (e) => {
    if (e.detail.photos) {
        e.detail.photos.forEach(photo => {
            // Handle both platforms
            const imageData = photo.dataUrl || photo.data
            const format = photo.format || photo.type

            processImage(imageData, format)
        })
    }
})
```

---

### 7. Use Single Global Listener

For better performance, use one global listener instead of multiple.

```javascript
// ✅ BETTER - Single global listener
document.addEventListener('cap-events-response', (e) => {
    if (e.detail.name === 'camera-permission-result') {
        handleCamera(e.detail)
    }
    else if (e.detail.photos !== undefined) {
        handlePhotos(e.detail)
    }
})

// ❌ WORSE - Multiple listeners for same event
document.addEventListener('cap-events-response', handleCamera)
document.addEventListener('cap-events-response', handlePhotos)
```

---

## Debugging

### Log All Events

```javascript
// Log all incoming events
document.addEventListener('cap-events-response', (e) => {
    console.log('📥 Received from native:', e.detail)
})

// Log all outgoing events
const originalDispatch = document.dispatchEvent.bind(document)
document.dispatchEvent = function(event) {
    if (event.type === 'cap-events') {
        console.log('📤 Sending to native:', event.detail)
    }
    return originalDispatch(event)
}
```

---

### Verify Event Structure

```javascript
document.addEventListener('cap-events-response', (e) => {
    console.group('Event Debug')
    console.log('Type:', e.type)
    console.log('Detail:', e.detail)
    console.log('Keys:', Object.keys(e.detail))
    console.groupEnd()

    // Verify expected fields
    if (e.detail.photos !== undefined) {
        console.assert(e.detail.success !== undefined, 'Missing success field')
        console.assert(e.detail.cancelled !== undefined, 'Missing cancelled field')
    }
})
```

---

### Common Issues

**Listener not firing:**
- Ensure listener is set up before dispatching event
- Verify you're listening to `cap-events-response`, not `cap-events`
- Check browser console for errors

**Missing response:**
- Verify event name is correct
- Check that native app has required permissions

**Unexpected response format:**
- Different events use different response structures
- Platform differences exist between Android and iOS
- Always log responses to inspect actual structure

---

## Quick Reference

### Event Summary

| Event | Send To Native | Receive From Native |
|-------|---------------|---------------------|
| Close | `{ name: 'close', activityPerformed?: string[] }` | None (on this WebView) / `child-portal-closed` on parent WebView for stacked portals |
| Open Portal | `{ name: 'open', url, ... }` | None |
| Camera | `{ name: 'camera-blocked' }` | `{ name: 'camera-permission-result', granted }` |
| Photos | `{ name: 'photo-library', selectionLimit, filter }` | `{ success, cancelled, photos }` |
| PDF | `{ name: 'intent', type: 'application/pdf', url, title }` | None |
| Error | `{ name: 'application-error' }` | None |
| Viewport Info | `{ name: 'get-viewport-info' }` | `{ event: 'viewport-info-response', safeAreaInsets, viewportInfo }` |
| Child Portal Closed | *(fired by SDK)* | `{ name: 'child-portal-closed', url, detail? }` — parent WebView only (Android SDK v0.7+, iOS SDK v0.6+) |

---

### Response Structure Patterns

**Camera Permission:**
```javascript
{ name: 'camera-permission-result', granted: boolean }
```

**Photos:**
```javascript
{
    success: boolean,
    cancelled: boolean,
    photos: [{ dataUrl/data, format/type }]
}
```

**Viewport Info (iOS only):**
```javascript
{
    event: 'viewport-info-response',
    safeAreaInsets: { top, bottom, left, right },
    viewportInfo: { availableHeight, contentHeight, customHeaderHeight, safeAreaInsets, hasCustomHeader }
}
```

**Child Portal Closed (parent WebView, Android SDK v0.7+ / iOS SDK v0.6+):**
```javascript
{
    name: 'child-portal-closed',
    url: '<child final URL>',
    detail: { name: 'close', activityPerformed: ['points-redeemed'] /* echoed from child */ }
    // `detail` is absent for native-initiated closes (hardware back, close modal, nav close)
}
```

---

### Helper Utility

```javascript
// Portal utility class
class PortalBridge {
    constructor() {
        this.listeners = new Map()
        this.setupGlobalListener()
    }

    setupGlobalListener() {
        document.addEventListener('cap-events-response', (e) => {
            // Camera permission
            if (e.detail.name === 'camera-permission-result') {
                this.trigger('camera', e.detail.granted)
            }
            // Photos
            else if (e.detail.photos !== undefined) {
                this.trigger('photos', e.detail)
            }
            // Viewport info
            else if (e.detail.event === 'viewport-info-response') {
                this.trigger('viewport', e.detail)
            }
            // Child portal closed (parent WebView only, Android SDK v0.7+ / iOS SDK v0.6+)
            else if (e.detail.name === 'child-portal-closed') {
                this.trigger('childClosed', e.detail)
            }
        })
    }

    on(event, callback) {
        if (!this.listeners.has(event)) {
            this.listeners.set(event, [])
        }
        this.listeners.get(event).push(callback)
    }

    off(event, callback) {
        if (!this.listeners.has(event)) return
        const callbacks = this.listeners.get(event)
        const index = callbacks.indexOf(callback)
        if (index > -1) callbacks.splice(index, 1)
    }

    trigger(event, data) {
        if (!this.listeners.has(event)) return
        this.listeners.get(event).forEach(callback => callback(data))
    }

    send(detail) {
        document.dispatchEvent(new CustomEvent('cap-events', { detail }))
    }

    // Convenience methods
    close(activityPerformed = []) {
        this.send({ name: 'close', activityPerformed })
    }

    openPortal(url, options = {}) {
        this.send({ name: 'open', url, ...options })
    }

    requestCamera() {
        this.send({ name: 'camera-blocked' })
    }

    openPhotoPicker(selectionLimit = 1) {
        this.send({ name: 'photo-library', selectionLimit, filter: 'images' })
    }

    openPDF(url, title) {
        this.send({ name: 'intent', type: 'application/pdf', url, title })
    }

    showError() {
        this.send({ name: 'application-error' })
    }

    getViewportInfo() {
        return new Promise((resolve) => {
            this.on('viewport', (data) => {
                resolve(data)
            })
            this.send({ name: 'get-viewport-info' })
        })
    }
}

// Usage
const portal = new PortalBridge()

portal.on('camera', (granted) => {
    console.log('Camera access:', granted)
})

portal.on('photos', (result) => {
    if (!result.cancelled && result.success) {
        console.log('Photos:', result.photos)
    }
})

// Parent-only: react when a stacked child portal closes
portal.on('childClosed', ({ url, detail }) => {
    const performed = detail?.activityPerformed ?? []
    if (performed.length === 0) return
    console.log(`Child at ${url} completed:`, performed)
    // e.g. refresh loyalty points, tier info, etc.
})

// Trigger events
portal.requestCamera()
portal.openPhotoPicker(5)
portal.openPDF('https://example.com/doc.pdf', 'Document')
portal.close(['points-redeemed'])   // close with activity signal
```

---

## Additional Resources

- Contact your mobile development team for:
  - Platform-specific features
  - Native debugging support
  - Permission configurations

**SDK Compatibility:**
- Android: Min SDK 24 (Android 7.0+)
- iOS: 13.0+
