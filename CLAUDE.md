# WinterWatch by Yamin

> Real-time winter road condition monitoring and traffic camera tracking.

---

## Stack

- **Framework:** Next.js 16 (App Router) + React 19
- **Bundler:** Turbopack
- **Styling:** Tailwind CSS v4 (`@tailwindcss/postcss`)
- **Map:** Leaflet (loaded via `next/dynamic` with `ssr: false`), CartoDB tiles (no API key)
- **Charts:** Recharts (weather widget temperature chart)
- **Icons:** Lucide React
- **Components:** shadcn/ui (Radix UI)
- **Deployment:** Vercel

---

## Features

- **Real-Time Traffic Cameras** — Live feeds from Austin's TxDOT camera network
- **AI-Powered Hazard Detection** — Condition detection (black ice, snow, accidents) based on weather
- **Interactive Map** — Leaflet + CartoDB dark tiles with camera markers and hazard zones
- **Weather Dashboard** — Open-Meteo current conditions, ice risk, 8-hour forecast
- **Smart Alerts** — Accordion-stacked critical notifications with auto-collapse
- **Route Planning** — Mock routes with external nav links (Google Maps, Waze, Apple Maps)
- **Location Picker** — Preset US cities + Nominatim search + geolocation
- **Responsive Design** — Desktop sidebar+map / Mobile tab-switching

---

## File Structure

```
app/
  page.tsx              # Main orchestrator — state management, layout, modals
  layout.tsx            # Root layout with metadata
  globals.css           # Tailwind config + CSS custom properties
  api/
    camera-image/
      route.ts          # Proxy for Austin camera feeds (only allows cctv.austinmobility.io)

components/winterwatch/
  sidebar.tsx           # Left sidebar — weather, alerts (accordion stack), camera list, filters
  map-view.tsx          # Dynamic import wrapper for MapComponent (SSR-safe)
  map-component.tsx     # Leaflet map — markers, hazards, alerts, tile switching, filters
  camera-preview.tsx    # Bottom sheet camera detail — feed, conditions, actions (bypass/notify/report)
  camera-card.tsx       # Sidebar camera list card
  alert-card.tsx        # Alert card component — clickable, with View Road / Route Around buttons
  weather-widget.tsx    # Weather display — temp, conditions, ice risk, 24h chart, road analysis
  route-planner.tsx     # Route planning modal — Nominatim search, mock routes, external nav links
  location-picker.tsx   # Location change modal — presets, search, Locate Me button
  report-hazard-modal   # (Inline in page.tsx) — hazard type grid, auto-detect location, submit

hooks/
  use-mobile.ts         # useIsMobile() hook for responsive layout switching

lib/
  winterwatch-types.ts  # TypeScript interfaces: Camera, Alert, HazardZone, WeatherData, Route
  austin-cameras.ts     # Camera feed definitions, simulateConditions(), generateCamerasForLocation()
  austin-realtime.ts    # fetchRealTimeIncidents(), fetchDriveTexasConditions()
  utils.ts              # Helper functions

public/
  winterwatch-logo.png  # Cyan wordmark logo (used in sidebar header + collapsed sidebar)
```

---

## Design System (CSS Custom Properties)

Defined in `globals.css`:

| Token | Purpose |
|-------|---------|
| `--ww-ice-blue` (`#00d4ff`) | Primary accent — selected states, ice indicators, CTAs |
| `--ww-critical` (`#ef4444`) | Critical alerts, accidents |
| `--ww-hazard` (`#f97316`) | Warning/hazard indicators |
| `--ww-caution` (`#fbbf24`) | Caution-level conditions |
| `--ww-safe` (`#10b981`) | Clear/safe indicators |
| `--ww-text-primary` | Main text color |
| `--ww-text-secondary` | Secondary text |
| `--ww-text-tertiary` | Muted/hint text |
| `--ww-surface` (`#151b24`) | Panel/card backgrounds |
| `--ww-surface-elevated` (`#1e2530`) | Elevated elements |
| `--background` (`#0a0e14`) | Page background |
| `--border` (`#2a3441`) | Borders |
| `--glass-border` | Glass-morphism border |

### Semantic Badge Colors (Solid)

| Condition | Background | Border |
|-----------|------------|--------|
| Black Ice | `#1a3a4a` | `#2a4a5a` |
| Accident | `#3a1a1a` | `#4a2a2a` |
| Low Visibility | `#3a351a` | `#4a4520` |
| Clear | `#1a2a24` | `#2a3a34` |

### Utility Classes

- `glass` — backdrop-blur panel with border and shadow
- `glass-light` — subtle glass effect for inner cards

### Animations

- `ice-glow` — Pulsing blue glow for ice alerts
- `wave-ripple` — Ripple effect for map markers
- `pulse-dot` — Breathing animation for live indicators
- `marker-pulse` — Map marker pulse animation

### Z-Index Hierarchy

| Layer | Z-Index |
|-------|---------|
| Leaflet map controls | `z-[1000]` |
| FAB buttons, mobile menu | `z-[1001]` |
| Camera preview panel | `z-[1002]` |
| Modals (route planner, location picker, report) | `z-[1003]` |
| Fullscreen feed viewer | `z-[2000]` |
| Notify confirmation popup | `z-[2001]` |

### Responsive Breakpoints

- **Mobile**: < 768px (sidebar OR map, tab-switching)
- **Desktop**: >= 768px (sidebar + map side-by-side, resizable sidebar)

---

## Data Flow

### Weather
- **Source:** Open-Meteo API (free, no key required)
- **Fetch:** `fetchWeatherData(lat, lng)` in `page.tsx`
- **Refresh:** Every 5 minutes (separate `useEffect` with stable deps)
- **WMO codes** mapped to condition strings + emoji icons (day/night variants)

### Cameras
- **Austin area:** Real camera feeds from `AUSTIN_CAMERA_FEEDS` in `austin-cameras.ts`
- **Other locations:** Generated via `generateCamerasForLocation()` with simulated data
- **Conditions:** `simulateConditions()` generates detection results based on weather
- **Images:** Proxied through `/api/camera-image` route (Austin cameras only)
- **Refresh:** Conditions re-simulated every 30 seconds via `camerasRef` pattern (avoids interval cascade)

### Alerts
- **Sources:** Three merged streams:
  1. `fetchRealTimeIncidents()` — Austin 311 / traffic API (prefixed `atx_`)
  2. `fetchDriveTexasConditions()` — TxDOT road conditions (prefixed `dtx_`)
  3. `generateAlerts()` — AI-detected from camera conditions (prefixed `alert_`)
- **User reports:** Added to both `alerts` and `hazards` arrays (prefixed `user_alert_` / `user_report_`)
- **Sorting:** Most recent first (by `createdAt` descending) in sidebar
- **Refresh:** Camera-based alerts regenerated every 30s; real-time alerts every 5 minutes

### Hazards
- **Generated from:** Camera detectedConditions via `generateHazards()`
- **User reports:** Added to hazards array with `source: "user_report"`
- **Display:** Pulsing markers + radius circles + bridge polylines on map

### Auto-Refresh Architecture
Two separate `useEffect` intervals avoid dependency cascade:
1. **30s interval** (camera conditions) — uses `camerasRef`/`weatherRef` refs, deps `[]`
2. **5min interval** (weather + real-time incidents) — deps `[userCoords, isNearAustin]`

---

## Key Interactions

### Alert Card Click
- Clicking anywhere on an alert card with a `cameraId` flies the map to that camera and opens the camera preview
- `handleViewCameraFromAlert()` injects the alert's hazard condition into the camera data if the camera's current conditions show "clear" — ensures the preview shows warning state

### Alert Accordion (Sidebar)
- On first load, all alerts are expanded; after 3 seconds they auto-collapse into a visual stack
- Stack shows the top card with peek layers behind, count hint below
- Click to expand all; "Collapse alerts" button to re-stack

### Camera Preview Actions
- **"Alert me when safe"** — sets a notification flag (UI-only confirmation)
- **"Bypass this road"** — opens Route Planner with `avoidLocation` pre-set (uses `camera.location` for road name)
- **"Report Hazard"** — opens Report Hazard modal with suggested type from detected conditions

### Map Markers
- **Camera markers:** 32px default, 40px when selected with glow shadow + permanent tooltip
- **Fly-to offset:** When a camera is selected, the map centers the marker in the visible area ABOVE the 52vh detail card using Leaflet `project`/`unproject` with a `window.innerHeight * 0.26` pixel offset
- **Selected tooltip:** Ice-blue background with glow (`.camera-tooltip-selected` CSS class)
- **Hazard markers:** Pulsing circles with permanent labels, radius circles, bridge polylines
- **Alert markers:** Shown for alerts without nearby hazard markers; clickable to trigger alert flow
- **User location:** Blue pulsing dot

### Map Tile Switching
- Dark (CartoDB dark_all) and Light (CartoDB light_all / Positron) modes
- Sun/Moon toggle button in top-right corner

### Location Picker
- Preset US cities + Nominatim autocomplete search (`&countrycodes=us`)
- "Use my current location" button with geolocation API
- Selecting a location recenters the map, refetches weather and camera data

### Route Planner
- Read-only "From" field shows current location
- "Avoiding" badge when opened from camera bypass
- Nominatim destination search with debounce
- Mock route generation (3 options: safest/fastest/shortest)
- External navigation links (Google Maps, Waze, Apple Maps)

---

## State in page.tsx

All major state lives in `page.tsx` and is passed down via props:
- `cameras`, `weather`, `alerts`, `hazards` — data arrays
- `selectedCamera`, `showCameraPreview` — camera detail state
- `showRoutePlanner`, `showReportModal`, `showLocationPicker` — modal visibility
- `userCoords`, `userLocation`, `userLocality` — user's location context
- `avoidLocation` — bypass context for route planner
- `mapFilter`, `tempUnit`, `isMapFullscreen` — UI preferences
- `mapActionsRef` — imperative handle to map's `locateUser()` function

---

## External APIs

| API | Usage | Auth |
|-----|-------|------|
| Open-Meteo | Weather data (current + hourly) | None (free) |
| Nominatim (OpenStreetMap) | Geocoding, reverse geocoding, location search | None (User-Agent header) |
| Austin Mobility CCTV | Camera feed images | None (proxied) |
| Austin 311 / SeeClickFix | Real-time traffic incidents | None |
| DriveTexas (TxDOT) | Road conditions, closures | None |

No environment variables required. All APIs are free/public.

---

## Data Models

### Camera
```typescript
{
  id, name, location, lat, lng,
  type: 'highway' | 'bridge' | 'intersection',
  status: 'operational' | 'offline' | 'maintenance',
  feedUrl, lastUpdate, detectedConditions[]
}
```

### HazardZone
```typescript
{
  id, type, lat, lng,
  severity: 'low' | 'moderate' | 'high' | 'critical',
  detectedAt, source, verificationCount
}
```

### WeatherData
```typescript
{
  temp, feelsLike, condition, icon,
  windSpeed, windDirection, visibility, humidity,
  roadTemp, iceRisk: 'low' | 'moderate' | 'high' | 'critical'
}
```

### Alert
```typescript
{
  id, type: 'critical' | 'warning' | 'info',
  title, description, location, cameraId, createdAt, dismissed
}
```

---

## Development

```bash
npm run dev      # Start dev server (port 3000)
npm run build    # Production build
npm run lint     # Run ESLint
npm run start    # Run production build
```

---

## Known Limitations

1. **Real camera feeds only work for Austin, TX** — other cities use generated/simulated cameras with placeholder images
2. **Route planning uses mock data** — routes are generated client-side, not from a real routing API
3. **Notifications are UI-only** — "Alert me when safe" doesn't persist or use real push notifications
4. **No authentication** — no user accounts, preferences aren't persisted
5. **Hazard reports are session-only** — user-reported hazards disappear on page refresh
6. **Camera conditions are simulated** — `simulateConditions()` uses weather data + randomness, not real CV
7. **DriveTexas/311 APIs may rate-limit** — no caching layer beyond the 5-minute refresh interval
8. **Accordion auto-collapse** — triggers once per mount; if alerts change count, it doesn't re-trigger
9. **Map alert markers** — only shown for alerts that don't have a nearby hazard marker (to avoid clutter)
