# Cookie Consent Implementation Guide

## Overview

**GDPR and privacy laws require explicit user consent before loading tracking/analytics scripts.** This guide shows the universal pattern for implementing ANY tracking feature with proper consent gating.

All v8 apps include a cookie consent banner (`src/components/CookieBanner.tsx`) and consent API (`src/lib/analytics-consent.ts`) that handle consent management automatically.

## The Universal Pattern

All tracking implementations follow the same pattern:

```typescript
import { getAnalyticsConsent, onConsentChange } from '@/lib/analytics-consent';

function loadTrackingScript(): void {
  if (document.getElementById('tracking-script')) return; // Check if already loaded

  // Initialize any required globals
  // Load the tracking script dynamically
  const script = document.createElement('script');
  script.id = 'tracking-script';
  script.src = 'https://tracking-provider.com/script.js';
  script.async = true;
  document.head.appendChild(script);
}

export default function TrackingComponent() {
  useEffect(() => {
    // Check consent on mount
    if (getAnalyticsConsent()) {
      loadTrackingScript();
    }

    // Listen for consent changes
    const cleanup = onConsentChange((consented) => {
      if (consented) {
        loadTrackingScript();
      }
    });

    return cleanup;
  }, []);

  return null;
}
```

**Key principles:**
1. **Never load immediately** — always check `getAnalyticsConsent()` first
2. **Subscribe to changes** — use `onConsentChange()` for mid-session consent changes
3. **Lazy load scripts** — use `document.createElement()` to load scripts conditionally
4. **Clean up** — return cleanup function from `onConsentChange()`
5. **Idempotent** — check if script already exists before adding

## Cookie Consent API

```typescript
import { getAnalyticsConsent, onConsentChange } from '@/lib/analytics-consent';

// Check current consent (returns boolean)
const hasConsent = getAnalyticsConsent();

// Listen for changes (returns cleanup function)
const cleanup = onConsentChange((consented: boolean) => {
  if (consented) {
    // User accepted - load tracking
  } else {
    // User declined/revoked - disable tracking
  }
});
```

**How it works:**
- Consent stored in `localStorage` as `c2_analytics_consent`
- Cookie banner dispatches `cookie-consent-changed` event on accept/decline
- `onConsentChange()` subscribes to this event
- Consent expires after 365 days

## When Consent Is Required

Apply consent gating for ANY script that:

- Tracks user behavior (analytics, heatmaps, session replay)
- Sets tracking cookies (advertising pixels, conversion tracking)
- Sends visitor data to third-party domains
- Contains keywords: `analytics`, `tracking`, `pixel`, `ads`, `tag`

**Common tracking domains:**
- `googletagmanager.com`, `google-analytics.com`
- `facebook.net`, `connect.facebook.net`
- `analytics.tiktok.com`, `linkedin.com/px`
- `hotjar.com`, `fullstory.com`, `logrocket.com`

**User intent phrases:**
- "add analytics", "track visitors", "website stats"
- "GTM", "Google Tag Manager", "Facebook Pixel"
- "session recording", "heatmap", "conversion tracking"

**Rule of thumb**: If you're unsure whether something tracks users, apply consent gating.

## Reference Implementation: Google Analytics

The GoogleAnalytics component demonstrates all best practices:

```typescript
import { useEffect, useRef } from 'react';
import { useLocation } from 'react-router';
import { getAnalyticsConsent, onConsentChange } from '@/lib/analytics-consent';

declare global {
  interface Window {
    dataLayer: unknown[];
    gtag: (...args: unknown[]) => void;
    [key: `ga-disable-${string}`]: boolean;
  }
}

function loadGtag(measurementId: string): void {
  delete window[`ga-disable-${measurementId}`]; // Clear opt-out flag
  if (document.getElementById('gtag-script')) return;

  window.dataLayer = window.dataLayer ?? [];
  window.gtag = (...args: unknown[]): void => {
    window.dataLayer.push(args);
  };
  window.gtag('js', new Date());
  window.gtag('config', measurementId);

  const script = document.createElement('script');
  script.id = 'gtag-script';
  script.src = `https://www.googletagmanager.com/gtag/js?id=${measurementId}`;
  script.async = true;
  document.head.appendChild(script);
}

function disableGtag(measurementId: string): void {
  window[`ga-disable-${measurementId}`] = true; // Official GA4 opt-out
}

export default function GoogleAnalytics() {
  const location = useLocation();
  const measurementIdRef = useRef<string | null>(null);
  const consentedRef = useRef<boolean>(false);

  useEffect(() => {
    let cancelled = false;

    async function init(): Promise<void> {
      try {
        const res = await fetch('/api/analytics/config');
        if (!res.ok || cancelled) return;
        const data = (await res.json()) as { measurementId?: string };
        if (!data.measurementId || cancelled) return;

        measurementIdRef.current = data.measurementId;
        if (getAnalyticsConsent()) {
          consentedRef.current = true;
          loadGtag(data.measurementId);
        }
      } catch {
        // Analytics must never break the app
      }
    }

    void init();

    const cleanup = onConsentChange((consented) => {
      const id = measurementIdRef.current;
      if (!id) return;

      if (consented) {
        consentedRef.current = true;
        loadGtag(id);
      } else {
        consentedRef.current = false;
        disableGtag(id);
      }
    });

    return () => {
      cancelled = true;
      cleanup();
    };
  }, []);

  // Track SPA page changes
  useEffect(() => {
    if (!consentedRef.current || typeof window.gtag !== 'function') return;
    window.gtag('event', 'page_view', { page_path: location.pathname });
  }, [location.pathname]);

  return null;
}
```

**What makes this a good reference:**
1. ✅ Lazy loading after consent check
2. ✅ Consent change handling with `onConsentChange()`
3. ✅ Proper cleanup on unmount
4. ✅ Revocation support via GA4 opt-out flag
5. ✅ Error resilience (try/catch, silent failures)
6. ✅ SPA page tracking with React Router
7. ✅ Idempotent script loading

**Apply this pattern to ANY tracking tool** — just replace the script URL, initialization code, and optional disable mechanism.

## Anti-Patterns

### ❌ DON'T: Load immediately

```html
<!-- WRONG -->
<script src="https://tracking.com/script.js"></script>
```

### ❌ DON'T: Use inline scripts

```tsx
{% raw %}
// WRONG
<script dangerouslySetInnerHTML={{__html: '...'}} />
{% endraw %}
```

### ❌ DON'T: Skip consent checks

```typescript
// WRONG
function loadAnalytics() {
  const script = document.createElement('script');
  script.src = 'https://tracking.com/script.js';
  document.head.appendChild(script);
}
```

### ✅ DO: Follow the pattern

```typescript
// CORRECT
if (getAnalyticsConsent()) {
  loadAnalytics();
}
onConsentChange((consented) => {
  if (consented) loadAnalytics();
});
```

## Testing

**Manual:**
1. Clear `localStorage`
2. Load app → cookie banner appears
3. Click "Decline" → tracking scripts NOT loaded (check Network tab)
4. Refresh, click "Accept" → tracking scripts load

**Automated:**
- E2E test coverage via `analytics-consent.test.ts` in the template library
- Gosym-probe scenario: `evals/gosym-probe/src/scenarios/cookie-consent-gtm.ts`

## Resources

- Cookie banner: `src/components/CookieBanner.tsx`
- Consent API: `src/lib/analytics-consent.ts`
- Google Analytics skill: Complete GA4 setup via skill system
