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

Close the current WebView.

**Send:**
```javascript
document.dispatchEvent(new CustomEvent('cap-events', {
    detail: { name: 'close' }
}))
```

**Receive:** None (portal closes)

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
| Close | `{ name: 'close' }` | None |
| Open Portal | `{ name: 'open', url, ... }` | None |
| Camera | `{ name: 'camera-blocked' }` | `{ name: 'camera-permission-result', granted }` |
| Photos | `{ name: 'photo-library', selectionLimit, filter }` | `{ success, cancelled, photos }` |
| PDF | `{ name: 'intent', type: 'application/pdf', url, title }` | None |
| Error | `{ name: 'application-error' }` | None |

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
    close() {
        this.send({ name: 'close' })
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

// Trigger events
portal.requestCamera()
portal.openPhotoPicker(5)
portal.openPDF('https://example.com/doc.pdf', 'Document')
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
- WebView: Modern browsers with CustomEvent support
