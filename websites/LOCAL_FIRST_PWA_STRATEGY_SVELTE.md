# PWA Local-First Strategy Document
## SvelteKit + Sanity CMS + Tailwind CSS

### Executive Summary
This document provides a technical strategy for building a local-first Progressive Web App (PWA) using SvelteKit, Sanity CMS, Tailwind CSS, and modern caching patterns. The architecture prioritizes offline-first functionality with intelligent cache revalidation and user-controlled update mechanisms.

---

## 1. Core Technology Stack

### Framework & Build Tools
- **SvelteKit 2+**: Meta-framework with SSR/SSG capabilities
- **Vite**: Fast build tool and development server
- **Tailwind CSS**: Utility-first CSS framework
- **@vite-pwa/sveltekit**: Zero-config PWA plugin for SvelteKit

### CMS Integration
- **Sanity.io**: Headless CMS with real-time capabilities
- **@sanity/client**: Official JavaScript client for data fetching
- **GROQ**: Query language for structured content

### Service Worker & Caching
- **Workbox 7+**: Google's service worker library (via vite-plugin-pwa)
- Built-in SvelteKit service worker support (`src/service-worker.js`)

---

## 2. Architecture Overview

### 2.1 Local-First Principles
The application should be designed with an **offline-first mindset**:
1. **Cache as primary data source**: Always attempt to serve from cache first
2. **Network as enhancement**: Treat network as sync mechanism, not prerequisite
3. **Background revalidation**: Update cache in background without blocking user
4. **Graceful degradation**: Full functionality offline with sync when online

### 2.2 Data Flow Pattern
```
User Request → Service Worker Intercept → Cache Check → 
  ├─ Cache Hit: Serve immediately + Background revalidate
  └─ Cache Miss: Network fetch → Cache → Serve
```

---

## 3. Implementation Strategy

### 3.1 PWA Setup with @vite-pwa/sveltekit

**Installation:**
```bash
npm install @vite-pwa/sveltekit -D
```

**Configuration (vite.config.ts):**
```typescript
import { sveltekit } from '@sveltejs/kit/vite'
import { SvelteKitPWA } from '@vite-pwa/sveltekit'

export default {
  plugins: [
    sveltekit(),
    SvelteKitPWA({
      registerType: 'autoUpdate',
      strategies: 'generateSW', // or 'injectManifest' for custom SW
      kit: {
        includeVersionFile: true
      },
      manifest: {
        name: 'App Name',
        short_name: 'App',
        theme_color: '#ffffff',
        background_color: '#ffffff',
        display: 'standalone',
        icons: [
          {
            src: '/icon-192.png',
            sizes: '192x192',
            type: 'image/png'
          },
          {
            src: '/icon-512.png',
            sizes: '512x512',
            type: 'image/png'
          }
        ]
      },
      workbox: {
        globPatterns: ['client/**/*.{js,css,ico,png,svg,webp,webmanifest}'],
        runtimeCaching: [
          // Caching strategies defined below
        ]
      }
    })
  ]
}
```

### 3.2 Caching Strategies by Resource Type

**A. Static App Shell (Cache-First)**
- HTML, CSS, JS bundles
- Fonts, icons, manifest
- Pre-cached on service worker installation

**B. CMS Content & Images (Stale-While-Revalidate)**
- Sanity API responses
- Content images from Sanity CDN
- Serves cached version immediately, updates in background

**C. Critical API Data (Network-First with Cache Fallback)**
- Real-time data requiring freshness
- Falls back to cache on network failure

**Workbox Configuration Example:**
```typescript
workbox: {
  runtimeCaching: [
    // Sanity API responses
    {
      urlPattern: /^https:\/\/.*\.api\.sanity\.io\/.*/i,
      handler: 'StaleWhileRevalidate',
      options: {
        cacheName: 'sanity-api-cache',
        expiration: {
          maxEntries: 50,
          maxAgeSeconds: 60 * 60 * 24 * 7 // 7 days
        },
        cacheableResponse: {
          statuses: [0, 200]
        }
      }
    },
    // Sanity CDN images
    {
      urlPattern: /^https:\/\/cdn\.sanity\.io\/.*/i,
      handler: 'StaleWhileRevalidate',
      options: {
        cacheName: 'sanity-images-cache',
        expiration: {
          maxEntries: 100,
          maxAgeSeconds: 60 * 60 * 24 * 30 // 30 days
        },
        cacheableResponse: {
          statuses: [0, 200]
        }
      }
    },
    // External images (if needed)
    {
      urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp)$/,
      handler: 'StaleWhileRevalidate',
      options: {
        cacheName: 'images-cache',
        expiration: {
          maxEntries: 60,
          maxAgeSeconds: 60 * 60 * 24 * 30 // 30 days
        }
      }
    }
  ]
}
```

### 3.3 Sanity CMS Integration

**Client Setup (lib/sanityClient.ts):**
```typescript
import { createClient } from '@sanity/client'

export const sanityClient = createClient({
  projectId: 'YOUR_PROJECT_ID',
  dataset: 'production',
  apiVersion: '2024-01-01', // Use current date
  useCdn: true, // Enable CDN for better caching
  perspective: 'published' // Only fetch published content
})
```

**Data Fetching Pattern (routes/+page.ts):**
```typescript
import { sanityClient } from '$lib/sanityClient'
import type { PageLoad } from './$types'

export const load: PageLoad = async () => {
  const data = await sanityClient.fetch(
    `*[_type == "post"][0...10]{
      _id,
      title,
      "imageUrl": image.asset->url,
      publishedAt
    }`
  )
  
  return { posts: data }
}
```

**Key Considerations:**
- Use Sanity's CDN (`useCdn: true`) for automatic edge caching
- GROQ queries return optimized data structures
- Image URLs from Sanity CDN work seamlessly with image caching strategies
- Consider using `stega` encoding for draft content if implementing live preview

---

## 4. Update Notification System

### 4.1 User-Controlled Updates

Users should be notified when a new version is available and given control over when to update.

**Implementation (routes/+layout.svelte):**
```svelte
<script lang="ts">
  import { onMount } from 'svelte'
  
  let updateAvailable = false
  let registration: ServiceWorkerRegistration | null = null
  
  onMount(async () => {
    if ('serviceWorker' in navigator) {
      registration = await navigator.serviceWorker.register('/service-worker.js')
      
      // Check for updates periodically
      setInterval(() => {
        registration?.update()
      }, 60 * 60 * 1000) // Check every hour
      
      // Listen for waiting service worker
      registration.addEventListener('updatefound', () => {
        const newWorker = registration?.installing
        
        newWorker?.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            updateAvailable = true
          }
        })
      })
    }
  })
  
  function refreshApp() {
    if (registration?.waiting) {
      registration.waiting.postMessage({ type: 'SKIP_WAITING' })
      window.location.reload()
    }
  }
</script>

{#if updateAvailable}
  <div class="fixed bottom-4 right-4 bg-blue-600 text-white p-4 rounded-lg shadow-lg z-50">
    <p class="mb-2 font-semibold">A new version is available!</p>
    <button 
      on:click={refreshApp}
      class="bg-white text-blue-600 px-4 py-2 rounded hover:bg-gray-100 transition-colors"
    >
      Update Now
    </button>
  </div>
{/if}

<slot />
```

**Service Worker Side (src/service-worker.js or custom SW):**
```javascript
self.addEventListener('message', (event) => {
  if (event.data && event.data.type === 'SKIP_WAITING') {
    self.skipWaiting()
  }
})
```

### 4.2 Update Notification Component (Alternative)

Create a reusable component for better organization:

**lib/components/UpdateNotification.svelte:**
```svelte
<script lang="ts">
  import { onMount } from 'svelte'
  
  let updateAvailable = false
  let registration: ServiceWorkerRegistration | null = null
  
  export let position: 'bottom-right' | 'top-right' | 'bottom-left' | 'top-left' = 'bottom-right'
  
  const positionClasses = {
    'bottom-right': 'bottom-4 right-4',
    'top-right': 'top-4 right-4',
    'bottom-left': 'bottom-4 left-4',
    'top-left': 'top-4 left-4'
  }
  
  onMount(async () => {
    if (!('serviceWorker' in navigator)) return
    
    try {
      registration = await navigator.serviceWorker.getRegistration()
      
      if (!registration) return
      
      // Check for updates periodically
      const updateInterval = setInterval(() => {
        registration?.update()
      }, 60 * 60 * 1000)
      
      // Listen for new service worker
      registration.addEventListener('updatefound', () => {
        const newWorker = registration?.installing
        
        newWorker?.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            updateAvailable = true
          }
        })
      })
      
      return () => clearInterval(updateInterval)
    } catch (error) {
      console.error('Service worker registration error:', error)
    }
  })
  
  function refreshApp() {
    if (registration?.waiting) {
      registration.waiting.postMessage({ type: 'SKIP_WAITING' })
      window.location.reload()
    }
  }
  
  function dismissUpdate() {
    updateAvailable = false
  }
</script>

{#if updateAvailable}
  <div 
    class="fixed {positionClasses[position]} bg-blue-600 text-white p-4 rounded-lg shadow-lg z-50 max-w-sm"
    role="alert"
  >
    <div class="flex items-start gap-3">
      <div class="flex-1">
        <p class="font-semibold mb-1">Update Available</p>
        <p class="text-sm text-blue-100">A new version of this app is ready to install.</p>
      </div>
      <button 
        on:click={dismissUpdate}
        class="text-blue-100 hover:text-white"
        aria-label="Dismiss"
      >
        ✕
      </button>
    </div>
    <div class="mt-3 flex gap-2">
      <button 
        on:click={refreshApp}
        class="flex-1 bg-white text-blue-600 px-4 py-2 rounded font-medium hover:bg-gray-100 transition-colors"
      >
        Update Now
      </button>
      <button 
        on:click={dismissUpdate}
        class="px-4 py-2 rounded border border-blue-400 text-white hover:bg-blue-700 transition-colors"
      >
        Later
      </button>
    </div>
  </div>
{/if}
```

**Usage in layout:**
```svelte
<script>
  import UpdateNotification from '$lib/components/UpdateNotification.svelte'
</script>

<UpdateNotification position="bottom-right" />
<slot />
```

---

## 5. Development Workflow

### 5.1 Local Development
```bash
npm run dev
```

**Development Configuration:**
- Enable PWA in development: `devOptions: { enabled: true }`
- Service worker updates on file changes
- Use browser DevTools → Application tab to inspect caches
- Test offline mode by toggling network in DevTools

### 5.2 Production Build
```bash
npm run build
npm run preview
```

**Build Process:**
- Generates optimized service worker with precache manifest
- Creates versioned assets with content hashes
- Compresses and optimizes all static assets
- Test PWA features in preview mode before deployment

### 5.3 Testing Checklist

**Installation:**
- [ ] App installs on desktop (Chrome, Edge)
- [ ] App installs on mobile (iOS Safari, Android Chrome)
- [ ] Install prompt appears correctly
- [ ] App icon displays properly on home screen

**Offline Functionality:**
- [ ] App loads when offline (after first visit)
- [ ] Cached content displays correctly
- [ ] Images load from cache
- [ ] Graceful error handling for uncached content

**Caching:**
- [ ] Static assets cache on first load
- [ ] API responses cache appropriately
- [ ] Images cache with correct strategy
- [ ] Cache updates on content changes

**Updates:**
- [ ] Update notification appears on new deployment
- [ ] "Update Now" button triggers reload
- [ ] New version loads after update
- [ ] No broken state after update

**Performance:**
- [ ] Lighthouse PWA audit passes (score 90+)
- [ ] First Contentful Paint < 1.5s
- [ ] Time to Interactive < 3s
- [ ] Minimal layout shift

---

## 6. Performance Optimization

### 6.1 Cache Size Management
```typescript
expiration: {
  maxEntries: 50, // Limit number of cached items
  maxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
  purgeOnQuotaError: true // Auto-cleanup on storage quota
}
```

### 6.2 Image Optimization

**Sanity Image Pipeline:**
```typescript
// Use Sanity's image URL builder
import imageUrlBuilder from '@sanity/image-url'
import { sanityClient } from '$lib/sanityClient'

const builder = imageUrlBuilder(sanityClient)

export function urlFor(source: any) {
  return builder.image(source)
}

// Usage in components
const imageUrl = urlFor(image)
  .width(800)
  .height(600)
  .format('webp')
  .quality(80)
  .url()
```

**Best Practices:**
- Lazy load images below the fold using `loading="lazy"`
- Use responsive images with `srcset`
- Serve WebP format with fallbacks
- Cache optimized image variants separately

### 6.3 Bundle Optimization
- Code splitting with SvelteKit's automatic route-based splitting
- Use `prerender: true` for static pages
- Preload critical resources in `app.html`
- Tree-shake unused dependencies
- Analyze bundle size with `vite-bundle-visualizer`

---

## 7. Critical Considerations

### 7.1 Cache Invalidation

**Service Worker Update Triggers:**
- Browser checks for updates every 24 hours (default)
- Manual check on app launch with `registration.update()`
- Any change to service worker file triggers update

**Force Update Strategy:**
```typescript
// In service worker or config
const CACHE_VERSION = 'v1.0.5' // Increment on each deploy
const CACHE_NAME = `app-cache-${CACHE_VERSION}`
```

### 7.2 iOS Safari Compatibility

**app.html additions:**
```html
<head>
  <!-- PWA Meta Tags -->
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="default">
  <meta name="apple-mobile-web-app-title" content="App Name">
  
  <!-- Apple Touch Icon -->
  <link rel="apple-touch-icon" href="/apple-touch-icon.png">
  
  <!-- Splash Screens (optional but recommended) -->
  <link rel="apple-touch-startup-image" href="/splash-640x1136.png" media="(device-width: 320px) and (device-height: 568px) and (-webkit-device-pixel-ratio: 2)">
</head>
```

**iOS Specific Testing:**
- Test extensively on actual iOS devices (simulator has limitations)
- Verify caching behavior (iOS may be more aggressive with cache clearing)
- Check offline functionality after device restart
- Validate Add to Home Screen flow

### 7.3 Cache-Control Headers

**Adapter Configuration:**

For **adapter-auto** or **adapter-vercel**:
```typescript
// svelte.config.js
export default {
  kit: {
    adapter: adapter(),
    // Headers handled by deployment platform
  }
}
```

For **adapter-node**:
```typescript
// Custom server.js or hooks.server.ts
export async function handle({ event, resolve }) {
  const response = await resolve(event)
  
  if (event.url.pathname === '/service-worker.js') {
    response.headers.set('Cache-Control', 'public, max-age=0, must-revalidate')
  }
  
  return response
}
```

**Key Headers:**
- Service worker: `Cache-Control: max-age=0`
- Static assets: Handled by Vite (long cache with hashed filenames)
- API responses: Configured per endpoint

---

## 8. Implementation Milestones

### Milestone 1: Core PWA Foundation
**Objective:** Establish basic PWA functionality with offline capability

**Tasks:**
- Install and configure @vite-pwa/sveltekit
- Create web app manifest with icons (192px, 512px minimum)
- Set up basic service worker with precaching
- Configure Tailwind CSS if not already present
- Test app installation on at least one platform

**Success Criteria:**
- App passes Lighthouse PWA audit
- App can be installed to home screen
- Basic offline functionality works

### Milestone 2: CMS Integration & Data Fetching
**Objective:** Connect Sanity CMS with proper data fetching patterns

**Tasks:**
- Set up Sanity client with proper configuration
- Implement GROQ queries for content types
- Create TypeScript types for content models
- Integrate data fetching in SvelteKit load functions
- Test data loading in both online and offline scenarios

**Success Criteria:**
- Content loads from Sanity correctly
- TypeScript types match Sanity schema
- Data fetching follows SvelteKit best practices

### Milestone 3: Advanced Caching Strategies
**Objective:** Implement intelligent caching for all resource types

**Tasks:**
- Configure stale-while-revalidate for CMS content
- Set up image caching for Sanity CDN
- Add cache expiration policies
- Implement cache size limits
- Configure cache-first for static assets
- Test offline content access

**Success Criteria:**
- API responses cache and revalidate correctly
- Images load from cache when offline
- Cache doesn't grow unbounded
- Stale content updates in background

### Milestone 4: Update Detection & Notification
**Objective:** Implement user-friendly update mechanism

**Tasks:**
- Create update detection logic
- Build UpdateNotification component
- Implement skipWaiting message handling
- Add periodic update checks
- Style notification to match app design
- Test update flow thoroughly

**Success Criteria:**
- Users see notification when update available
- "Update Now" triggers clean reload
- No broken state after update
- Dismissible notification option works

### Milestone 5: Testing, Optimization & Documentation
**Objective:** Ensure production-ready quality and team knowledge transfer

**Tasks:**
- Cross-browser testing (Chrome, Firefox, Safari, Edge)
- iOS Safari specific testing
- Android Chrome testing
- Lighthouse performance audits
- Bundle size optimization
- Cache strategy refinement
- Write team documentation
- Create deployment checklist

**Success Criteria:**
- All items in testing checklist pass
- Lighthouse scores: Performance 90+, PWA 90+
- Documentation complete and reviewed
- Team trained on PWA concepts

---

## 9. Resources & Documentation

### Official Documentation
- **SvelteKit Service Workers**: https://kit.svelte.dev/docs/service-workers
- **Vite PWA SvelteKit**: https://vite-pwa-org.netlify.app/frameworks/sveltekit.html
- **Workbox Strategies**: https://developers.google.com/web/tools/workbox/modules/workbox-strategies
- **Sanity JS Client**: https://www.sanity.io/docs/js-client
- **Sanity + SvelteKit**: https://www.sanity.io/sveltekit-cms

### Key GitHub Repositories
- **@vite-pwa/sveltekit**: https://github.com/vite-pwa/sveltekit
- **Sanity SvelteKit Template**: https://github.com/sanity-io/sanity-template-svelte-kit
- **Workbox**: https://github.com/GoogleChrome/workbox

### Learning Resources
- **MDN PWA Caching Guide**: https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Caching
- **web.dev PWA Update**: https://web.dev/learn/pwa/update
- **web.dev Workbox Guide**: https://web.dev/learn/pwa/workbox
- **PWA Training**: https://web.dev/learn/pwa

### Tools
- **Lighthouse**: Built into Chrome DevTools for PWA audits
- **Workbox Window**: For service worker communication
- **PWA Builder**: https://www.pwabuilder.com/ for generating assets

---

## 10. Success Metrics

### Technical Metrics
- **Lighthouse PWA Score**: Target 90+
- **First Contentful Paint**: < 1.5s
- **Time to Interactive**: < 3s
- **Largest Contentful Paint**: < 2.5s
- **Cache Hit Rate**: > 80% (after first visit)
- **Offline Functionality**: 100% for cached content
- **Bundle Size**: Monitor and optimize (target < 200KB initial)

### User Experience Metrics
- **App Install Rate**: Track installation conversions
- **Offline Session Frequency**: Monitor offline usage
- **Update Acceptance Rate**: Track how many users update immediately
- **Load Time Improvement**: Compare cached vs network-only loads
- **Error Rate**: Monitor failed requests and cache misses

### Business Metrics
- **Engagement**: Compare PWA users vs web-only users
- **Retention**: Track return visits from installed PWA
- **Performance Impact**: Measure conversion/engagement improvements
- **Bounce Rate**: Monitor reduction in bounce rate

---

## 11. Troubleshooting Common Issues

### Service Worker Not Updating
**Problem:** Changes don't appear after deployment

**Solutions:**
- Check browser DevTools → Application → Service Workers
- Verify service worker file has changed (content hash or version)
- Clear site data and reload
- Check Cache-Control headers on service worker
- Ensure `registration.update()` is called

### Images Not Caching
**Problem:** Images don't load offline

**Solutions:**
- Verify Sanity CDN URL pattern in workbox config
- Check Network tab to see if requests are intercepted
- Confirm cache name appears in Application → Cache Storage
- Test with simple image first before complex CDN URLs

### iOS Installation Issues
**Problem:** Add to Home Screen doesn't work on iOS

**Solutions:**
- Verify `apple-touch-icon` link is present
- Check manifest `display` mode is set to `standalone`
- Ensure HTTPS is being used (required for PWA)
- Test in Safari (not Chrome) as iOS uses Safari engine
- Clear Safari cache and try again

### Cache Taking Too Much Space
**Problem:** App uses excessive storage

**Solutions:**
- Reduce `maxEntries` in expiration config
- Decrease `maxAgeSeconds` for faster expiration
- Enable `purgeOnQuotaError: true`
- Implement selective caching for images (only cache viewed items)

---

## Conclusion

This strategy provides a comprehensive foundation for building a local-first PWA with SvelteKit, Sanity CMS, and Tailwind CSS. The stale-while-revalidate pattern balances performance (instant cached responses) with freshness (background updates), while the user-controlled update system ensures a smooth experience without disruptive automatic reloads.

Key principles to remember:
- **Offline-first mindset**: Design for offline, enhance with online
- **User control**: Always give users choice over updates
- **Progressive enhancement**: Start with basics, add features incrementally
- **Test extensively**: Especially on iOS and various network conditions

