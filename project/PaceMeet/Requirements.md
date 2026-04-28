# PaceMeet — Full Project Requirements

> A running community app for Bangkok (and Thai cities) where runners create pace-targeted events, share routes, and relive every run through a collaborative photo map.  
> Last updated: April 2026

---

## Table of contents

1. [[#1. Project overview|Project overview]]
2. [[#2. Geography strategy & tradeoffs|Geography strategy & tradeoffs]]
3. [[#3. User roles & access model|User roles & access model]]
4. [[#4. Functional requirements|Functional requirements]]
5. [[#5. The collaborative photo map (hero feature)|The collaborative photo map (hero feature)]]
6. [[#6. Non-functional requirements|Non-functional requirements]]
7. [[#7. Third-party integrations & data sources|Third-party integrations & data sources]]
8. [[#8. System architecture|System architecture]]
9. [[#9. Microservices breakdown|Microservices breakdown]]
10. [[#10. Database & storage design|Database & storage design]]
11. [[#11. Message queue design (Kafka)|Message queue design (Kafka)]]
12. [[#12. Scheduled jobs (Cloud Scheduler + Cloud Run Jobs)|Scheduled jobs (Cloud Scheduler + Cloud Run Jobs)]]
13. [[#13. Deployment & infrastructure|Deployment & infrastructure]]
14. [[#14. CI/CD pipeline|CI/CD pipeline]]
15. [[#15. Out of scope (v1)|Out of scope (v1)]]

---

## 1. Project overview

PaceMeet is a mobile-first community app (iOS + Android via React Native) for runners who want to run together, not just log solo miles. It solves the friction between three things that currently require separate apps — finding a group run (Facebook Groups), tracking the route (Strava), and discovering where to eat/drink after (Google Maps).

**The core loop:**

```
Create or join a pace-targeted event
    → Meet at the start point
    → Run the route (optional GPS tracking)
    → Hit highlight spots along the way, take photos together
    → Finish → browse post-run venues nearby
    → View the shared photo map: every member's geotagged photos
      plotted on the route — the visual memory of your run
```

**What makes it different:**

|Feature|Facebook Groups|Strava|PaceMeet|
|---|---|---|---|
|Pace-targeted events|No|No|Yes|
|Route on a map|No|Yes|Yes|
|RSVP + headcount|Approximate|No|Yes|
|Highlight spots|No|Segments only|Yes (user-curated)|
|Post-run venue discovery|No|No|Yes|
|Collaborative photo map|No|No|Yes — the hero feature|

**Launch target:** Bangkok, Thailand. English UI with Thai language support from day one.

---

## 2. Geography strategy & tradeoffs

### v1 — Bangkok + English-first, Thai-ready (recommended)

Build the app in English with Thai language strings included in the first release (`i18n` from day one, but Bangkok venue data only). Use Wongnai API for Thai venue discovery (coffee shops, restaurants near the finish line) and Google Maps for everything else.

**What this means technically:**

- Venue data seeded for Bangkok only (Wongnai + Google Places).
- Events are not city-scoped in the data model — they use coordinates, which is already city-agnostic.
- Push notifications and email in English + Thai.
- No geofencing or city-selection UI needed yet.

### v2 — Expand to Chiang Mai, Phuket, other Thai cities

This is a near-zero technical change because events are coordinate-based. The only additions needed:

- Seed venue data for new cities via the ETL pipeline (Wongnai covers all major Thai cities).
- Add a city filter to the event discovery feed.
- Add city-tagged community groups (optional social layer).

**Cost of waiting:** Zero. Design the data model with coordinates + optional `city` tag from day one, and v2 expansion is a config + data change.

### v3 — Global expansion

This requires real work:

- Drop Wongnai, fall back to Google Places only (or add Foursquare).
- Full i18n for all dynamic content (event titles, descriptions entered by users in any language).
- Time zone handling in every event datetime field (`stored as UTC`, `displayed in local tz`).
- Consider international running tourists in Bangkok as an earlier opportunity — English-only PaceMeet already serves them in v1.

**Recommendation:** Validate the product in Bangkok for 3–6 months before committing to global infrastructure complexity.

---

## 3. User roles & access model

|Role|Description|
|---|---|
|**Guest**|Can browse public events and routes on the map. Cannot join or create. Prompted to sign up on any action.|
|**Runner**|Registered user. Can join events, create routes, upload photos, follow others, and create events (organizer mode).|
|**Organizer**|Not a separate role — any Runner can create an event. The creator of an event is its organizer with extra controls.|
|**Admin**|Internal role for content moderation, venue data management, and flagged content review.|

Registration is open — no invite codes. Users sign up with email/password or Google/Apple OAuth.

---

## 4. Functional requirements

### 4.1 Authentication

- Sign up via email + password, Google OAuth, or Apple Sign-In (required for iOS App Store).
- JWT access tokens (15 min) + refresh tokens (30 days, httpOnly cookie).
- Password reset via email link (expires 1 hour).
- Profile: display name, profile photo, home city, pace group preference, bio (optional), total runs logged.

### 4.2 Event creation (organizer flow)

An organizer creates a run event with the following fields:

**Required:**

- Event title (e.g., "Saturday morning Lumpini loop")
- Date and time (stored UTC, displayed in Bangkok time / device local time)
- Meeting point — pin on map (lat/lng + optional address label)
- Pace group — one of: Easy (>7:00/km), Moderate (6:00–7:00/km), Brisk (5:00–6:00/km), Fast (<5:00/km)
- Max participants (1–200, or unlimited)
- Event visibility: Public (discoverable by anyone) or Private (join by link only)

**Optional:**

- Route — drawn on map or imported as GPX file (stored as GeoJSON)
- Finish point — separate from meeting point if different
- Post-run venue — pre-selected from venue search, or left open for group to decide on the day
- Description / notes (e.g., "We stop for water at km 5, bring your own")
- Highlight spots along the route (see Section 4.5)
- Cover photo

### 4.3 Event discovery & joining (joiner flow)

- The home feed shows upcoming public events sorted by distance from the user's current location (or last known location if location permission denied).
- Filters: date range, pace group, distance from me, max participants (full events hidden by default).
- Event detail page shows: route on map, pace group badge, current RSVP count, organizer profile, highlight spots, post-run venue (if set), member list (avatars).
- User taps "Join" → they are added to the RSVP list. The organizer receives a push notification.
- If the event is full, users can join a waitlist. Cancellations auto-promote from the waitlist.
- Users can cancel their RSVP up to 1 hour before the event start time.

### 4.4 Route creation & management

Users can create a route independently of any event (a reusable library of runs):

- **Draw on map:** Tap waypoints on the map; the app snaps to roads/paths using Google Maps Roads API. The route is stored as a GeoJSON `LineString`.
- **GPX import:** Upload a `.gpx` file (e.g., exported from Garmin or Strava). The app parses it and stores it as GeoJSON.
- **Pin only:** No route — just a meeting point pin. Valid for events where the organizer wants to decide the route on the day.

Route metadata stored:

- Estimated distance (km, calculated from GeoJSON)
- Estimated elevation gain (from Google Elevation API)
- Surface type (road / trail / mixed — user-selected)
- Public / private visibility
- Creator attribution
- Highlight spots (see Section 4.5)

Routes can be saved to a personal library and reused across multiple events. Other users can "save" a public route to their own library.

### 4.5 Highlight spots

A highlight spot is a named point of interest along or near a route — added by the organizer or any participant. Examples: a scenic bridge, a mural, a water fountain, a particularly hard hill, a temple gate.

- Stored as a `Point` (lat/lng) with a title, optional description, and optional category (scenic / facility / landmark / food & drink).
- Displayed as a distinct pin color on the route map.
- During or after a run, members can take/upload a photo tagged to a highlight spot (see Section 5).
- Any user can add a highlight spot to a public route (subject to organizer approval for events).
- Highlight spots persist on the route and accumulate photos across multiple events that use the same route.

### 4.6 Optional GPS tracking during a run

GPS tracking is opt-in per event per user:

- When the event is "live" (within 30 minutes of start time until end), participants can tap "Start tracking" in the app.
- The app records GPS breadcrumbs every 10 seconds and stores them locally on-device (no live upload during the run to save battery and data).
- After the user taps "End run," the full GPS track is uploaded as a GeoJSON `LineString` and attached to the event as the user's personal run log.
- The run log stores: distance (km), duration, average pace (min/km), elevation gain, and the GPS track.
- Live location sharing (where you are on the map right now, visible to other event members) is a v2 feature.

### 4.7 Post-run venue discovery

After an event ends (or any time from the event detail page):

- The app shows nearby venues within 1.5 km of the finish point, filtered by type: Coffee, Breakfast / brunch, Smoothie / juice, Full meal.
- Venues are sourced from Wongnai (Thai-language reviews, very relevant for Bangkok) supplemented by Google Places.
- Each venue card shows: name, category, distance from finish, Google/ Wongnai rating, opening hours, and a "Get directions" button.
- The organizer can pre-pin one venue as the "official post-run spot" when creating the event. This venue is highlighted at the top of the list.
- Users can tap "I'm going" on a venue to let other members know where they're headed after the run.

### 4.8 Collaborative photo map (see Section 5 for full detail)

- Members can take or upload photos during or after an event, each geotagged with a lat/lng.
- After the event ends, all members' photos are aggregated and plotted on the route map.
- This "event memory map" is visible to all event participants and is the primary post-run social experience.

### 4.9 Social & community features

- **Follow / followers:** Users can follow each other. Followed users' upcoming events appear in a "Friends running" section of the home feed.
- **Run history:** Each user has a public profile showing past events attended, routes created, and total distance logged.
- **Pace reputation:** After attending an event, the organizer can mark a user's actual pace as matching, faster, or slower than the event pace group. This builds a soft "pace credibility" score visible on profiles (helps organizers curate the right group).
- **Badges:** Milestone badges shown on profile — first run, 10 runs, 50 km logged, night runner, early bird (event before 7am), post-run regular (venue check-in 5+ times).
- **Event ratings:** After an event, participants can leave a 1–5 star rating and optional comment. Rating aggregates on the organizer's profile.
- **Report / flag:** Any user can flag inappropriate content (events, photos, comments). Flagged content is queued for admin review.

### 4.10 Notifications

Push notifications (Firebase Cloud Messaging) for:

- Someone joins your event (organizer)
- You are promoted from waitlist
- Event starts in 1 hour (reminder)
- New event posted by someone you follow
- New photos added to an event you attended
- Someone rated your event
- Badge unlocked

In-app notification bell for all of the above when the app is open.

---

## 5. The collaborative photo map (hero feature)

This is the feature that defines PaceMeet and does not exist elsewhere. Here is the full design and technical flow.

### 5.1 Taking photos during an event

- The in-app camera is accessible from the event "live" screen (available once the event start time is within 30 minutes).
- When a photo is taken or uploaded from camera roll, the app records the GPS coordinates at the moment of capture (from device location services).
- Photos are stored locally on-device until the user taps "End run" or "Upload photos," at which point they are uploaded to GCS (Google Cloud Storage).
- Members do not need to be GPS-tracking to take photos — photo geotag uses a one-time location read at the moment of photo capture.
- Photo upload is chunked (for large files on mobile data) and retries automatically if the connection drops.

### 5.2 Photo processing pipeline

After upload, each photo goes through an async processing pipeline (triggered via Kafka → Photo Processing Service):

1. **EXIF extraction** — if the photo already has EXIF GPS data (taken outside the app), use that lat/lng. Otherwise use the app-recorded lat/lng.
2. **Moderation** — run through Google Cloud Vision Safe Search API. Photos flagged as likely adult/violent content are held for admin review.
3. **Thumbnail generation** — create a 300×300 and 800×600 version stored in GCS. The original is retained.
4. **Nearest highlight spot tagging** — if the photo was taken within 100m of a highlight spot on the route, it is auto-tagged to that spot. Users can manually re-tag or untag.
5. **Event association** — photo is linked to the event, the user, the coordinates, and the timestamp.

### 5.3 The event memory map

Once the event ends (or the organizer marks it as ended), the memory map becomes available to all participants:

- The event route is drawn on the map as a colored line.
- Each geotagged photo appears as a circular avatar thumbnail pinned at its coordinates.
- Tapping a photo pin opens a full-screen view with the photo, the member's name, the timestamp, and the nearest highlight spot label (if auto-tagged).
- Multiple photos taken at the same location are clustered into a single pin showing a count; tapping expands the cluster into a grid.
- Highlight spot pins are shown in a different color. Tapping a highlight spot pin shows all photos taken near that spot across all events on that route (building a persistent "place memory" over time).
- The memory map can be scrolled/zoomed exactly like a normal map.

### 5.4 Sharing

- The memory map is viewable in-app by all event participants.
- An organizer can make the memory map "public" (viewable by anyone with the link, without logging in) — useful for sharing to Instagram/LINE.
- A static "event recap card" image is auto-generated after all photos are processed: route line on a map thumbnail, photo grid strip along the bottom, event name/date/distance/participant count overlaid. This is the shareable image for social media.

---

## 6. Non-functional requirements

|Category|Requirement|
|---|---|
|**App performance**|Home feed loads in under 1 second. Event detail page (with map) under 1.5 seconds.|
|**Photo upload**|Chunked upload; app remains usable during background upload.|
|**Memory map**|Renders with up to 200 photos on the map without jank. Cluster pins above 5 photos in the same 50m radius.|
|**Offline**|Event detail (route, meeting point, participant list) is cached locally after first view so it's accessible without signal on race day.|
|**Push notifications**|Delivered within 30 seconds of trigger event.|
|**GPS accuracy**|Photo geotag accurate to ±20m (standard device GPS).|
|**Image moderation**|Flagged photos hidden within 2 minutes of upload (async pipeline SLA).|
|**Scale (v1)**|Support 500 registered users, 50 concurrent events, 5,000 photos stored.|
|**Security**|Photos are private to event participants by default. HTTPS everywhere. No raw GPS data exposed via public API.|
|**Availability**|99% uptime. Planned maintenance only on weekday nights (minimal Thai running community activity 11pm–5am).|

---

## 7. Third-party integrations & data sources

### 7.1 Google Maps Platform

**What it provides:** Maps rendering, route drawing (snap-to-roads), place search, elevation data, and reverse geocoding.

**Pricing:** $200 free credit/month. For a community app at Bangkok scale (< 500 DAU), the free credit covers Maps SDK + Places API comfortably. Budget ~$50–100/month once you grow past ~2,000 DAU.

**Sign up:** https://console.cloud.google.com → APIs & Services → Enable: Maps SDK for Android, Maps SDK for iOS, Places API, Roads API, Elevation API.

**How to use in the project (React Native):**

```bash
npm install react-native-maps
# iOS: add Google Maps API key to AppDelegate.mm
# Android: add to AndroidManifest.xml
```

```javascript
// Render a route as a polyline on the map
import MapView, { Polyline, Marker } from 'react-native-maps';

const routeCoordinates = [
  { latitude: 13.7563, longitude: 100.5018 },  // Lumpini Park gate
  { latitude: 13.7580, longitude: 100.5040 },
  // ... decoded from GeoJSON LineString
];

<MapView initialRegion={{ latitude: 13.7563, longitude: 100.5018,
                           latitudeDelta: 0.02, longitudeDelta: 0.02 }}>
  <Polyline coordinates={routeCoordinates} strokeColor="#FF5733" strokeWidth={3} />
  <Marker coordinate={routeCoordinates[0]} title="Start" />
</MapView>
```

```python
# Backend: snap user-drawn waypoints to roads (Roads API)
import requests

def snap_to_roads(waypoints: list[dict]) -> list[dict]:
    """
    waypoints: list of {lat, lng} dicts
    returns: snapped list of {lat, lng} dicts
    """
    path = "|".join(f"{p['lat']},{p['lng']}" for p in waypoints)
    response = requests.get(
        "https://roads.googleapis.com/v1/snapToRoads",
        params={
            "path": path,
            "interpolate": True,
            "key": get_secret("google-maps-api-key"),
        },
        timeout=10
    )
    response.raise_for_status()
    snapped = response.json().get("snappedPoints", [])
    return [
        {"lat": p["location"]["latitude"], "lng": p["location"]["longitude"]}
        for p in snapped
    ]
```

```python
# Fetch elevation profile for a route (for display + stats)
def get_elevation(path_coords: list[dict]) -> list[float]:
    path = "|".join(f"{p['lat']},{p['lng']}" for p in path_coords[:512])
    response = requests.get(
        "https://maps.googleapis.com/maps/api/elevation/json",
        params={
            "path": f"enc:{encode_polyline(path_coords)}",
            "samples": min(len(path_coords), 100),
            "key": get_secret("google-maps-api-key"),
        },
        timeout=10
    )
    return [r["elevation"] for r in response.json().get("results", [])]
```

**Key APIs used:**

|API|Purpose|Called by|
|---|---|---|
|Maps SDK (mobile)|Render map, show route, pins|React Native app|
|Roads API|Snap drawn waypoints to roads|Route Service (backend)|
|Places API (Nearby Search)|Post-run venue discovery|Venue Service (backend)|
|Elevation API|Elevation gain for route stats|Route Service (backend)|
|Geocoding API|Convert lat/lng to address label|Route Service|

---

### 7.2 Wongnai API — Thai venue discovery

**What it provides:** Thailand's largest food and lifestyle review platform. Venues in Bangkok have far richer Thai-language data here than on Google Places — local coffee shops, rice porridge stalls, and runners' favourites are all listed with ratings, opening hours, and photos.

**Pricing:** Contact Wongnai for API access (https://www.wongnai.com/developer). There is a developer program for startups. Google Places is the free fallback if Wongnai access is not available at launch.

**How to use in the project (backend — Venue Service):**

```python
import requests

WONGNAI_API_KEY = get_secret("wongnai-api-key")
WONGNAI_BASE = "https://api.wongnai.com/v2"

def search_venues_near(lat: float, lng: float,
                        radius_m: int = 1500,
                        category: str = "cafe") -> list[dict]:
    """
    Fetch venues near a finish point for post-run discovery.
    category options: cafe, restaurant, bakery, juice_bar
    """
    response = requests.get(
        f"{WONGNAI_BASE}/venues/nearby",
        headers={"Authorization": f"Bearer {WONGNAI_API_KEY}"},
        params={
            "lat": lat,
            "lng": lng,
            "radius": radius_m,
            "category": category,
            "limit": 10,
            "open_now": True,         # Only show open venues
            "sort": "rating",
        },
        timeout=10
    )
    response.raise_for_status()
    venues = response.json().get("venues", [])

    return [
        {
            "id": v["id"],
            "name": v["name"],
            "name_th": v.get("name_th"),
            "category": v.get("category"),
            "rating": v.get("rating"),
            "review_count": v.get("review_count"),
            "distance_m": v.get("distance"),
            "address": v.get("address"),
            "lat": v["location"]["lat"],
            "lng": v["location"]["lng"],
            "opening_hours": v.get("opening_hours"),
            "photo_url": v.get("cover_photo", {}).get("url"),
            "wongnai_url": v.get("url"),
        }
        for v in venues
    ]

# Fallback: Google Places Nearby Search
def search_venues_google(lat: float, lng: float,
                          radius_m: int = 1500,
                          place_type: str = "cafe") -> list[dict]:
    response = requests.get(
        "https://maps.googleapis.com/maps/api/place/nearbysearch/json",
        params={
            "location": f"{lat},{lng}",
            "radius": radius_m,
            "type": place_type,
            "opennow": True,
            "key": get_secret("google-maps-api-key"),
        },
        timeout=10
    )
    results = response.json().get("results", [])
    return [
        {
            "id": r["place_id"],
            "name": r["name"],
            "rating": r.get("rating"),
            "distance_m": None,   # Not returned directly, compute from coords
            "lat": r["geometry"]["location"]["lat"],
            "lng": r["geometry"]["location"]["lng"],
            "address": r.get("vicinity"),
            "photo_url": None,    # Requires a second Places Photo call
        }
        for r in results
    ]

# Merge Wongnai + Google, deduplicate by proximity
def get_post_run_venues(lat: float, lng: float) -> list[dict]:
    try:
        venues = search_venues_near(lat, lng, category="cafe")
        venues += search_venues_near(lat, lng, category="restaurant")
    except Exception:
        venues = search_venues_google(lat, lng, place_type="cafe")
        venues += search_venues_google(lat, lng, place_type="restaurant")

    seen_names = set()
    unique = []
    for v in sorted(venues, key=lambda x: x.get("rating") or 0, reverse=True):
        if v["name"] not in seen_names:
            seen_names.add(v["name"])
            unique.append(v)
    return unique[:12]
```

**Cache venues per finish location in Redis** with a 6-hour TTL. Post-run venue lists don't need real-time freshness.

---

### 7.3 Google Cloud Vision — Photo moderation

**What it provides:** Safe Search Detection API flags photos containing adult content, violence, medical content, and racy content. Run on every uploaded photo before it is made visible to other users.

**Pricing:** 1,000 units/month free, then $1.50 per 1,000 units. For a beta with a few thousand photos, this is effectively free.

**How to use in the project:**

```python
from google.cloud import vision

vision_client = vision.ImageAnnotatorClient()

def moderate_photo(gcs_uri: str) -> dict:
    """
    gcs_uri: e.g. "gs://pacemeet-photos/events/abc123/photo_456.jpg"
    Returns: {safe: bool, flags: list[str]}
    """
    image = vision.Image(source=vision.ImageSource(gcs_image_uri=gcs_uri))
    response = vision_client.safe_search_detection(image=image)
    ss = response.safe_search_annotation

    THRESHOLD = vision.Likelihood.LIKELY  # Flag LIKELY and VERY_LIKELY

    flags = []
    if ss.adult >= THRESHOLD:
        flags.append("adult")
    if ss.violence >= THRESHOLD:
        flags.append("violence")
    if ss.racy >= THRESHOLD:
        flags.append("racy")

    return {
        "safe": len(flags) == 0,
        "flags": flags,
        "raw": {
            "adult": ss.adult.name,
            "violence": ss.violence.name,
            "racy": ss.racy.name,
        }
    }

# Called by the Photo Processing Service after upload
def process_uploaded_photo(photo_id: str, gcs_uri: str, db):
    result = moderate_photo(gcs_uri)
    if result["safe"]:
        db.execute(
            "UPDATE photos SET status='approved' WHERE id=%s", (photo_id,)
        )
    else:
        db.execute(
            "UPDATE photos SET status='pending_review', moderation_flags=%s WHERE id=%s",
            (result["flags"], photo_id)
        )
        # Notify admin via Kafka: moderation-queue topic
```

---

### 7.4 Firebase Cloud Messaging (FCM) — Push notifications

**What it provides:** Free, reliable push notification delivery to both iOS and Android via a single API.

**Pricing:** Free, no practical limits for a community app.

**How to use in the project:**

```python
# Backend: Notification Service
import firebase_admin
from firebase_admin import messaging, credentials

cred = credentials.Certificate(get_secret("firebase-service-account-json"))
firebase_admin.initialize_app(cred)

def send_push(device_token: str, title: str,
              body: str, data: dict = None) -> bool:
    message = messaging.Message(
        notification=messaging.Notification(title=title, body=body),
        data=data or {},
        token=device_token,
        android=messaging.AndroidConfig(priority="high"),
        apns=messaging.APNSConfig(
            payload=messaging.APNSPayload(
                aps=messaging.Aps(sound="default")
            )
        )
    )
    try:
        messaging.send(message)
        return True
    except Exception as e:
        print(f"FCM error: {e}")
        return False

def send_event_reminder(event_id: str, db):
    """Send 1-hour-before reminder to all RSVPs."""
    rows = db.execute(
        """SELECT u.fcm_token, u.display_name, e.title
           FROM rsvps r
           JOIN users u ON r.user_id = u.id
           JOIN events e ON r.event_id = e.id
           WHERE r.event_id = %s AND r.status = 'confirmed'
             AND u.fcm_token IS NOT NULL""",
        (event_id,)
    ).fetchall()

    for row in rows:
        send_push(
            device_token=row["fcm_token"],
            title=f"Your run starts in 1 hour",
            body=f"{row['title']} — see you at the start!",
            data={"event_id": event_id, "type": "event_reminder"}
        )
```

```javascript
// React Native: register device token on app open
import messaging from '@react-native-firebase/messaging';

async function registerFCMToken(userId) {
  const token = await messaging().getToken();
  await api.post('/users/fcm-token', { token });  // Save to backend

  // Handle incoming push when app is in foreground
  messaging().onMessage(async remoteMessage => {
    showInAppNotification(remoteMessage.notification);
  });
}
```

---

### 7.5 GPX parsing — Route import

No third-party API needed. GPX is an XML format. Parse it in the backend (Route Service) when a user uploads a `.gpx` file:

```python
import xml.etree.ElementTree as ET
import json

def parse_gpx(gpx_bytes: bytes) -> dict:
    """
    Parse a GPX file and return a GeoJSON LineString + metadata.
    """
    root = ET.fromstring(gpx_bytes)
    ns = {"gpx": "http://www.topografix.com/GPX/1/1"}

    points = []
    elevations = []

    # Try <trkseg> first (track), fall back to <rte> (route)
    trkpts = root.findall(".//gpx:trkpt", ns) or root.findall(".//gpx:rtept", ns)

    for pt in trkpts:
        lat = float(pt.attrib["lat"])
        lon = float(pt.attrib["lon"])
        ele_el = pt.find("gpx:ele", ns)
        ele = float(ele_el.text) if ele_el is not None else None
        points.append([lon, lat])  # GeoJSON is [lng, lat]
        if ele is not None:
            elevations.append(ele)

    if not points:
        raise ValueError("No track points found in GPX file")

    # Calculate distance (Haversine)
    total_km = sum(
        haversine(points[i], points[i+1])
        for i in range(len(points) - 1)
    )

    # Elevation gain (sum of positive differences)
    elev_gain = sum(
        max(0, elevations[i+1] - elevations[i])
        for i in range(len(elevations) - 1)
    ) if len(elevations) > 1 else None

    return {
        "geojson": {
            "type": "LineString",
            "coordinates": points
        },
        "distance_km": round(total_km, 2),
        "elevation_gain_m": round(elev_gain, 1) if elev_gain else None,
        "point_count": len(points),
    }

def haversine(p1: list, p2: list) -> float:
    """Returns distance in km between two [lng, lat] points."""
    from math import radians, sin, cos, sqrt, atan2
    R = 6371
    lon1, lat1 = map(radians, p1)
    lon2, lat2 = map(radians, p2)
    dlat = lat2 - lat1
    dlon = lon2 - lon1
    a = sin(dlat/2)**2 + cos(lat1)*cos(lat2)*sin(dlon/2)**2
    return R * 2 * atan2(sqrt(a), sqrt(1 - a))
```

---

### 7.6 Third-party summary

|Service|Purpose|Free tier|Paid needed at|
|---|---|---|---|
|Google Maps Platform|Maps, routing, places, elevation|$200/month credit|~2,000 DAU|
|Wongnai API|Thai venue discovery|Contact for dev access|From launch|
|Google Places API|Venue fallback (global)|Included in Maps credit|Same as above|
|Google Cloud Vision|Photo moderation|1,000 units/month free|~1,000 photo uploads/month|
|Firebase Cloud Messaging|Push notifications|Free, unlimited|Never|
|Google Cloud Storage|Photo + asset storage|5 GB free|~5,000 photos|

**Secrets setup:**

```bash
gcloud secrets create google-maps-api-key \
  --data-file=<(echo -n "YOUR_KEY")
gcloud secrets create wongnai-api-key \
  --data-file=<(echo -n "YOUR_KEY")
gcloud secrets create firebase-service-account-json \
  --data-file=firebase-service-account.json
```

---

## 8. System architecture

### 8.1 Architecture tiers

```
[ React Native App (iOS + Android) ]
              |
     [ API Gateway (Kong / Nginx) ]
              |
┌──────────┬──────────┬─────────────┬──────────┬────────────────┐
│   User   │  Event   │   Route     │  Venue   │     Photo      │
│ Service  │ Service  │  Service    │ Service  │    Service     │
└──────────┴──────────┴─────────────┴──────────┴────────────────┘
              |                                        |
[ Kafka: rsvp-events · photo-uploaded · notifications · moderation-queue ]
              |                                        |
┌─────────────────┬─────────────────┬──────────────────────────┐
│   PostgreSQL    │      Redis      │   Google Cloud Storage   │
│  + PostGIS      │    (cache)      │   (photos, thumbnails)   │
└─────────────────┴─────────────────┴──────────────────────────┘
              |
       [ BigQuery (OLAP) ]
              |
[ Cloud Scheduler + Cloud Run Jobs ]
```

### 8.2 Key data flows

**Creating and joining an event:**

```
App → API Gateway → Event Service
    → write to PostgreSQL (events, rsvps)
    → publish to Kafka: rsvp-events
        → Notification Service consumes → FCM push to organizer
```

**Uploading a photo during a run:**

```
App → Photo Service (multipart upload)
    → store original in GCS
    → publish to Kafka: photo-uploaded
        → Photo Processing Service consumes:
            → generate thumbnails → GCS
            → run Cloud Vision moderation
            → extract/confirm geotag
            → tag to nearest highlight spot
            → update PostgreSQL: photos table (status: approved)
            → publish to Kafka: notifications
                → Notification Service → FCM to all event participants
```

**Loading the event memory map:**

```
App requests /events/{id}/memory-map
    → Event Service reads from PostgreSQL:
        - route GeoJSON
        - all approved photos for this event (lat, lng, thumbnail_url, user)
        - highlight spots
    → Returns combined GeoJSON FeatureCollection
    → App renders on MapView with photo pins + route polyline
```

---

## 9. Microservices breakdown

### User Service

- Language: Node.js (TypeScript)
- Responsibilities: registration, login, OAuth, JWT, profile CRUD, FCM token registration, follow/unfollow, badge award logic
- DB: PostgreSQL (`users`, `follows`, `badges`, `refresh_tokens`)
- Exposes: REST `/auth/*`, `/users/*`

### Event Service

- Language: Go
- Responsibilities: event CRUD, RSVP management, waitlist logic, event lifecycle state machine (upcoming → live → ended), pace group management, event ratings
- DB: PostgreSQL (`events`, `rsvps`, `waitlist`, `event_ratings`)
- Publishes to Kafka: `rsvp-events`, `notifications`
- Exposes: REST `/events/*`

### Route Service

- Language: Python
- Responsibilities: route creation (draw + GPX import), GeoJSON storage, snap-to-roads via Google Roads API, elevation fetch, highlight spot CRUD, route library management
- DB: PostgreSQL with PostGIS extension (`routes`, `highlight_spots`)
- Exposes: REST `/routes/*`, `/highlight-spots/*`

### Venue Service

- Language: Python
- Responsibilities: fetch post-run venues from Wongnai + Google Places, cache in Redis, serve venue list for a given finish coordinate
- Cache: Redis (`venues:{lat_rounded}:{lng_rounded}` with 6h TTL)
- Exposes: REST `/venues/nearby`

### Photo Service

- Language: Go (for efficient multipart streaming)
- Responsibilities: receive photo uploads, stream to GCS, publish `photo-uploaded` Kafka event, serve photo metadata + memory map endpoint
- DB: PostgreSQL (`photos`)
- Storage: Google Cloud Storage
- Exposes: REST `/photos/*`, `/events/{id}/memory-map`

### Photo Processing Service

- Language: Python
- Responsibilities: Kafka consumer for `photo-uploaded` — generate thumbnails (Pillow), run Cloud Vision moderation, extract EXIF geotag, snap to nearest highlight spot, update photo status in PostgreSQL
- Consumes from Kafka: `photo-uploaded`
- Publishes to Kafka: `notifications`, `moderation-queue`

### Notification Service

- Language: Node.js
- Responsibilities: Kafka consumer for `notifications` — look up user FCM tokens from PostgreSQL, send via Firebase Admin SDK, write to in-app notification table
- Consumes from Kafka: `notifications`
- DB: PostgreSQL (`notifications`)

---

## 10. Database & storage design

### PostgreSQL + PostGIS (primary operational database)

PostGIS is a PostgreSQL extension that adds native support for geographic data types (`GEOMETRY`, `GEOGRAPHY`) and spatial queries (distance, contains, intersects). Required for route storage and proximity queries.

**Key tables (full schema designed separately):**

```sql
-- Core user data
users               (id, email, password_hash, display_name, profile_photo_url,
                     city, fcm_token, pace_preference, bio, created_at)
follows             (follower_id, following_id, created_at)
badges              (id, user_id, badge_type, awarded_at)

-- Events
events              (id, organizer_id, title, description, start_at, end_at,
                     meeting_point GEOGRAPHY(POINT),
                     finish_point GEOGRAPHY(POINT),
                     route_id, pace_group, max_participants,
                     status, visibility, cover_photo_url, created_at)
rsvps               (id, event_id, user_id, status, rsvp_at)
waitlist            (id, event_id, user_id, position, added_at)
event_ratings       (id, event_id, rater_id, score, comment, created_at)

-- Routes
routes              (id, creator_id, title, geojson GEOMETRY(LINESTRING,4326),
                     distance_km, elevation_gain_m, surface_type,
                     visibility, created_at)
highlight_spots     (id, route_id, creator_id, title, description, category,
                     location GEOGRAPHY(POINT), created_at)

-- Photos
photos              (id, event_id, user_id, gcs_original_uri,
                     gcs_thumbnail_300_uri, gcs_thumbnail_800_uri,
                     location GEOGRAPHY(POINT),
                     highlight_spot_id, taken_at, uploaded_at,
                     status, moderation_flags)

-- Venues (cached from external APIs)
venues              (id, external_id, source, name, name_th, category,
                     location GEOGRAPHY(POINT), rating, address,
                     opening_hours_json, photo_url, last_fetched_at)

-- Notifications
notifications       (id, user_id, type, title, body, data_json,
                     read, created_at)
```

**Spatial query examples:**

```sql
-- Find events within 5 km of user's location
SELECT * FROM events
WHERE ST_DWithin(
    meeting_point,
    ST_MakePoint(100.5018, 13.7563)::GEOGRAPHY,
    5000  -- meters
)
AND status = 'upcoming'
ORDER BY start_at ASC;

-- Find nearest highlight spot to a photo's coordinates
SELECT id, title,
       ST_Distance(location,
           ST_MakePoint(100.5150, 13.7420)::GEOGRAPHY) AS distance_m
FROM highlight_spots
WHERE route_id = 'abc123'
ORDER BY distance_m ASC
LIMIT 1;
```

### Redis (cache layer)

|Key|Value|TTL|
|---|---|---|
|`venues:{lat4}:{lng4}`|JSON array of venue objects|6 hours|
|`event:{id}:rsvp_count`|integer|60 seconds|
|`user:{id}:session`|session metadata|24 hours|
|`event:{id}:memory_map`|full GeoJSON FeatureCollection|5 minutes|

### Google Cloud Storage (object storage)

```
pacemeet-photos/
├── originals/{event_id}/{photo_id}.jpg
├── thumbnails/300/{event_id}/{photo_id}.jpg
├── thumbnails/800/{event_id}/{photo_id}.jpg
└── recap_cards/{event_id}/recap.jpg     ← auto-generated event card

pacemeet-assets/
├── profile_photos/{user_id}.jpg
└── event_covers/{event_id}.jpg
```

All photo URLs are served via GCS with a CDN (Cloud CDN) in front. Original photos are private (signed URLs only). Thumbnails are public.

### BigQuery (analytics)

|Table|Description|
|---|---|
|`events_log`|All event creation, RSVP, cancellation events with timestamps|
|`photo_uploads_log`|Photo upload events, moderation outcomes|
|`venue_clicks`|Which venues users tap on after events (for recommendation tuning)|
|`user_activity_daily`|Daily active users, runs attended, photos uploaded|

---

## 11. Message queue design (Kafka)

### Topics

|Topic|Producer|Consumers|Message schema|
|---|---|---|---|
|`rsvp-events`|Event Service|Notification Service, BigQuery sink|`{event_id, user_id, action (join/cancel/promote), timestamp}`|
|`photo-uploaded`|Photo Service|Photo Processing Service|`{photo_id, event_id, user_id, gcs_uri, lat, lng, taken_at}`|
|`notifications`|Event Service, Photo Processing|Notification Service|`{user_ids[], type, title, body, data}`|
|`moderation-queue`|Photo Processing Service|Admin notification|`{photo_id, flags[], gcs_uri}`|

**Configuration:** Use Redpanda Cloud free tier (same as StockFolio). 4 partitions per topic, 7-day retention.

---

## 12. Scheduled jobs (Cloud Scheduler + Cloud Run Jobs)

Each scheduled task is a standalone Docker container deployed as a **Cloud Run Job** and triggered by **Cloud Scheduler**. Jobs run to completion and exit — no persistent process, no Airflow overhead. Each job is independently deployable and independently retried by Cloud Scheduler on failure.

### Why this instead of Airflow

PaceMeet's pipeline has 5 sequential, independent jobs with no branching or fan-out. Cloud Run Jobs + Cloud Scheduler is the right fit: zero management overhead, pay only when running (effectively free at beta scale), and each job is just a normal Python script in a Docker container — the same pattern as the rest of the backend.

Migrate to Airflow (or Cloud Workflows) in v2 if inter-job dependencies grow complex or you need visual pipeline monitoring.

---

### Job 1 — `job-snapshot-event-stats`

**Schedule:** `0 17 * * *` (UTC) = 00:00 Bangkok time, daily

**What it does:**

- Queries PostgreSQL for all events that ended in the past 24 hours.
- For each event: counts total RSVPs, confirmed attendees, photos uploaded, and average rating.
- Writes one row per event to BigQuery table `events_log`.

```python
# jobs/snapshot_event_stats/main.py
import os
import psycopg2
from google.cloud import bigquery
from datetime import datetime, timedelta, timezone

def run():
    pg = psycopg2.connect(os.environ["POSTGRES_CONN_STR"])
    bq = bigquery.Client()

    since = datetime.now(timezone.utc) - timedelta(hours=24)
    rows = pg.cursor().execute("""
        SELECT
            e.id, e.title, e.organizer_id, e.start_at, e.pace_group,
            COUNT(DISTINCT r.id) FILTER (WHERE r.status='confirmed') AS rsvp_count,
            COUNT(DISTINCT p.id) AS photo_count,
            AVG(er.score) AS avg_rating
        FROM events e
        LEFT JOIN rsvps r ON r.event_id = e.id
        LEFT JOIN photos p ON p.event_id = e.id AND p.status='approved'
        LEFT JOIN event_ratings er ON er.event_id = e.id
        WHERE e.end_at BETWEEN %s AND NOW()
        GROUP BY e.id
    """, (since,)).fetchall()

    errors = bq.insert_rows_json(
        "pacemeet.analytics.events_log",
        [dict(zip([d.name for d in rows[0].description], row)) for row in rows]
    )
    if errors:
        raise RuntimeError(f"BigQuery insert errors: {errors}")
    print(f"Snapshotted {len(rows)} events to BigQuery.")

if __name__ == "__main__":
    run()
```

---

### Job 2 — `job-award-badges`

**Schedule:** `10 17 * * *` (UTC) = 00:10 Bangkok time, daily (runs 10 minutes after job 1 so stats are available)

**What it does:**

- Checks all users who attended an event in the past 24 hours against badge conditions.
- Inserts newly earned badges into PostgreSQL `badges` table.
- Publishes a Kafka notification event for each new badge so the Notification Service delivers an FCM push to the user.

**Badge conditions checked:**

|Badge|Condition|
|---|---|
|First run|Total events attended = 1|
|10 runs|Total events attended = 10|
|50 km logged|Cumulative distance ≥ 50 km|
|Early bird|Attended an event with start_at before 07:00 local time|
|Post-run regular|Post-run venue check-in 5+ times|
|Night runner|Attended an event with start_at after 20:00 local time|

```python
# jobs/award_badges/main.py
import os, json
import psycopg2
from confluent_kafka import Producer

BADGE_RULES = {
    "first_run":       lambda stats: stats["total_events"] == 1,
    "ten_runs":        lambda stats: stats["total_events"] == 10,
    "fifty_km":        lambda stats: stats["total_distance_km"] >= 50,
    "early_bird":      lambda stats: stats["early_morning_events"] >= 1,
    "post_run_regular":lambda stats: stats["venue_checkins"] >= 5,
    "night_runner":    lambda stats: stats["night_events"] >= 1,
}

def run():
    pg = psycopg2.connect(os.environ["POSTGRES_CONN_STR"])
    producer = Producer({"bootstrap.servers": os.environ["KAFKA_BOOTSTRAP"]})

    # Find users active in last 24h
    active_users = pg.cursor().execute("""
        SELECT DISTINCT user_id FROM rsvps
        WHERE status = 'confirmed'
          AND updated_at > NOW() - INTERVAL '24 hours'
    """).fetchall()

    for (user_id,) in active_users:
        stats = fetch_user_stats(pg, user_id)
        existing_badges = fetch_existing_badges(pg, user_id)

        for badge_type, condition in BADGE_RULES.items():
            if badge_type not in existing_badges and condition(stats):
                pg.cursor().execute(
                    "INSERT INTO badges (user_id, badge_type) VALUES (%s, %s)",
                    (user_id, badge_type)
                )
                pg.commit()
                # Publish push notification event
                producer.produce(
                    "notifications",
                    json.dumps({
                        "user_ids": [str(user_id)],
                        "type": "badge_unlocked",
                        "title": "Badge unlocked!",
                        "body": f"You earned the {badge_type.replace('_', ' ')} badge",
                        "data": {"badge_type": badge_type}
                    })
                )
    producer.flush()
    print(f"Badge check complete for {len(active_users)} users.")

if __name__ == "__main__":
    run()
```

---

### Job 3 — `job-refresh-venue-cache`

**Schedule:** `20 17 * * *` (UTC) = 00:20 Bangkok time, daily

**What it does:**

- Queries PostgreSQL for all events in the next 7 days that have a `finish_point`.
- For each finish point, calls Wongnai + Google Places to fetch nearby venues (coffee, food).
- Upserts venue records into the PostgreSQL `venues` table.
- Warms the Redis cache for each finish coordinate so the first user to open the event detail page gets instant venue results.

```python
# jobs/refresh_venue_cache/main.py
import os, json
import psycopg2
import redis
from venue_client import get_post_run_venues  # shared lib from Venue Service

def run():
    pg = psycopg2.connect(os.environ["POSTGRES_CONN_STR"])
    r = redis.from_url(os.environ["REDIS_URL"])

    upcoming = pg.cursor().execute("""
        SELECT DISTINCT
            ST_Y(finish_point::geometry) AS lat,
            ST_X(finish_point::geometry) AS lng
        FROM events
        WHERE start_at BETWEEN NOW() AND NOW() + INTERVAL '7 days'
          AND finish_point IS NOT NULL
    """).fetchall()

    for (lat, lng) in upcoming:
        venues = get_post_run_venues(lat, lng)
        cache_key = f"venues:{round(lat,3)}:{round(lng,3)}"
        r.setex(cache_key, 21600, json.dumps(venues))  # 6h TTL
        upsert_venues(pg, venues)

    pg.commit()
    print(f"Venue cache warmed for {len(upcoming)} finish points.")

if __name__ == "__main__":
    run()
```

---

### Job 4 — `job-generate-discovery-feed`

**Schedule:** `30 17 * * *` (UTC) = 00:30 Bangkok time, daily

**What it does:**

- Finds all public events in the next 14 days that are not yet full.
- Scores each event by: recency of creation, pace variety available that week, and organizer's average rating.
- Writes the top 20 to the PostgreSQL `discovery_feed` table, replacing the previous day's feed.
- The mobile app reads from `discovery_feed` — no on-the-fly computation needed at request time.

```python
# jobs/generate_discovery_feed/main.py
import os
import psycopg2

def score_event(event: dict) -> float:
    recency_score = max(0, 1 - (event["days_until"] / 14))
    rating_score  = (event["organizer_avg_rating"] or 3.0) / 5.0
    fill_score    = 1 - (event["rsvp_count"] / max(event["max_participants"], 1))
    return (recency_score * 0.4) + (rating_score * 0.4) + (fill_score * 0.2)

def run():
    pg = psycopg2.connect(os.environ["POSTGRES_CONN_STR"])
    cur = pg.cursor()

    events = cur.execute("""
        SELECT
            e.id,
            e.title,
            e.start_at,
            e.pace_group,
            e.max_participants,
            EXTRACT(DAY FROM e.start_at - NOW()) AS days_until,
            COUNT(r.id) AS rsvp_count,
            AVG(er.score) AS organizer_avg_rating
        FROM events e
        LEFT JOIN rsvps r ON r.event_id = e.id AND r.status = 'confirmed'
        LEFT JOIN event_ratings er ON er.organizer_id = e.organizer_id
        WHERE e.status = 'upcoming'
          AND e.visibility = 'public'
          AND e.start_at BETWEEN NOW() AND NOW() + INTERVAL '14 days'
        GROUP BY e.id
        HAVING COUNT(r.id) < e.max_participants OR e.max_participants IS NULL
    """).fetchall()

    scored = sorted(events, key=lambda e: score_event(e), reverse=True)[:20]

    cur.execute("TRUNCATE TABLE discovery_feed")
    for rank, event in enumerate(scored, start=1):
        cur.execute(
            "INSERT INTO discovery_feed (event_id, rank, generated_at) VALUES (%s, %s, NOW())",
            (event["id"], rank)
        )
    pg.commit()
    print(f"Discovery feed updated with {len(scored)} events.")

if __name__ == "__main__":
    run()
```

---

### Job 5 — `job-compute-user-stats`

**Schedule:** `40 17 * * *` (UTC) = 00:40 Bangkok time, daily

**What it does:**

- Recomputes aggregate stats for all users who attended a run in the past 30 days: total events attended, total distance logged, runs this month, and longest streak (consecutive weeks with at least one run).
- Upserts into the PostgreSQL `user_stats` table (used by profile pages).
- Appends a daily snapshot row to BigQuery `user_activity_daily` for analytics.

```python
# jobs/compute_user_stats/main.py
import os
import psycopg2
from google.cloud import bigquery
from datetime import date

def run():
    pg = psycopg2.connect(os.environ["POSTGRES_CONN_STR"])
    bq = bigquery.Client()

    stats = pg.cursor().execute("""
        SELECT
            u.id AS user_id,
            COUNT(DISTINCT r.event_id)                    AS total_events,
            COALESCE(SUM(rl.distance_km), 0)              AS total_distance_km,
            COUNT(DISTINCT r.event_id) FILTER (
                WHERE e.start_at > date_trunc('month', NOW())
            )                                             AS events_this_month
        FROM users u
        JOIN rsvps r ON r.user_id = u.id AND r.status = 'confirmed'
        JOIN events e ON e.id = r.event_id
        LEFT JOIN run_logs rl ON rl.event_id = e.id AND rl.user_id = u.id
        WHERE e.start_at > NOW() - INTERVAL '30 days'
        GROUP BY u.id
    """).fetchall()

    for row in stats:
        pg.cursor().execute("""
            INSERT INTO user_stats (user_id, total_events, total_distance_km, events_this_month, updated_at)
            VALUES (%s, %s, %s, %s, NOW())
            ON CONFLICT (user_id) DO UPDATE SET
                total_events = EXCLUDED.total_events,
                total_distance_km = EXCLUDED.total_distance_km,
                events_this_month = EXCLUDED.events_this_month,
                updated_at = NOW()
        """, (row["user_id"], row["total_events"], row["total_distance_km"], row["events_this_month"]))
    pg.commit()

    # Write daily snapshot to BigQuery
    bq.insert_rows_json(
        "pacemeet.analytics.user_activity_daily",
        [{"date": str(date.today()), "active_users": len(stats),
          "total_runs": sum(r["events_this_month"] for r in stats)}]
    )
    print(f"User stats computed for {len(stats)} users.")

if __name__ == "__main__":
    run()
```

---

### Job 6 — `job-event-reminders`

**Schedule:** `*/5 * * * *` (every 5 minutes, all day)

**What it does:**

- Finds events starting in the next 55–65 minute window (to catch the "1 hour before" moment without double-firing).
- POSTs to the Notification Service internal endpoint which dispatches FCM pushes to all confirmed RSVPs.

```python
# jobs/event_reminders/main.py
import os, requests
import psycopg2

def run():
    pg = psycopg2.connect(os.environ["POSTGRES_CONN_STR"])
    events = pg.cursor().execute("""
        SELECT id, title FROM events
        WHERE start_at BETWEEN NOW() + INTERVAL '55 minutes'
                           AND NOW() + INTERVAL '65 minutes'
          AND reminder_sent = FALSE
          AND status = 'upcoming'
    """).fetchall()

    for event in events:
        resp = requests.post(
            f"{os.environ['NOTIFICATION_SERVICE_URL']}/internal/send-reminder",
            json={"event_id": str(event["id"])},
            headers={"Authorization": f"Bearer {os.environ['INTERNAL_TOKEN']}"},
            timeout=10
        )
        resp.raise_for_status()
        pg.cursor().execute(
            "UPDATE events SET reminder_sent = TRUE WHERE id = %s", (event["id"],)
        )
    pg.commit()
    print(f"Reminders sent for {len(events)} events.")

if __name__ == "__main__":
    run()
```

---

### Cloud Scheduler configuration (Terraform)

```hcl
# terraform/modules/scheduled_jobs/main.tf

locals {
  jobs = {
    snapshot-event-stats  = { schedule = "0 17 * * *",  image = "job-snapshot-event-stats" }
    award-badges          = { schedule = "10 17 * * *", image = "job-award-badges" }
    refresh-venue-cache   = { schedule = "20 17 * * *", image = "job-refresh-venue-cache" }
    generate-discovery    = { schedule = "30 17 * * *", image = "job-generate-discovery-feed" }
    compute-user-stats    = { schedule = "40 17 * * *", image = "job-compute-user-stats" }
    event-reminders       = { schedule = "*/5 * * * *", image = "job-event-reminders" }
  }
}

resource "google_cloud_run_v2_job" "scheduled_job" {
  for_each = local.jobs
  name     = each.key
  location = var.region

  template {
    template {
      containers {
        image = "${var.region}-docker.pkg.dev/${var.project_id}/pacemeet/${each.value.image}:latest"
        env {
          name = "POSTGRES_CONN_STR"
          value_source {
            secret_key_ref {
              secret  = "postgres-conn-str"
              version = "latest"
            }
          }
        }
        env {
          name  = "KAFKA_BOOTSTRAP"
          value = var.kafka_bootstrap
        }
        env {
          name  = "REDIS_URL"
          value = var.redis_url
        }
      }
    }
  }
}

resource "google_cloud_scheduler_job" "trigger" {
  for_each  = local.jobs
  name      = "trigger-${each.key}"
  schedule  = each.value.schedule
  time_zone = "Asia/Bangkok"

  http_target {
    http_method = "POST"
    uri = "https://${var.region}-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/${var.project_id}/jobs/${each.key}:run"
    oauth_token {
      service_account_email = var.scheduler_sa_email
    }
  }

  retry_config {
    retry_count = 3
    min_backoff_duration = "30s"
    max_backoff_duration = "300s"
  }
}
```

### Job execution flow summary

```
Cloud Scheduler (cron)
    → HTTP POST to Cloud Run Jobs API
        → Cloud Run spins up a container
            → Job runs to completion
                → Container exits (success) or errors out
                    → Cloud Scheduler retries up to 3 times on failure
                        → Alert via Cloud Monitoring if all retries fail
```

Failures are surfaced in GCP Cloud Logging. Set a log-based alert in Cloud Monitoring on `severity=ERROR` from any job container to get notified if a nightly job fails.

---

## 13. Deployment & infrastructure

### GCP stack recommendation

|Component|GCP service|Notes|
|---|---|---|
|React Native app|Expo EAS Build + App Store / Play Store|EAS handles iOS/Android CI builds|
|Microservices (5)|Cloud Run|Stateless, scales to 0|
|API Gateway|Cloud Run + custom Nginx config, or Kong||
|PostgreSQL + PostGIS|Cloud SQL (Postgres 15 + PostGIS extension)|db-f1-micro for beta|
|Redis|Memorystore (Redis 7)|1 GB basic|
|Kafka|Redpanda Cloud free tier||
|Photo storage|Cloud Storage + Cloud CDN|CDN for fast image delivery|
|Photo processing jobs|Cloud Run Jobs|Triggered by Kafka consumer|
|ETL|Cloud Scheduler + Cloud Run Jobs|Replaces Composer to save cost|
|BigQuery|BigQuery|Serverless|
|Secrets|Secret Manager|All API keys|
|Push notifications|Firebase (free)||
|Container registry|Artifact Registry||

### Cost estimate (Bangkok beta, ~500 users)

|Service|Monthly estimate|
|---|---|
|Cloud Run (7 services)|~$10–20|
|Cloud SQL (db-f1-micro + PostGIS)|~$15|
|Memorystore (1 GB)|~$35|
|Cloud Storage + CDN (~10 GB photos)|~$5|
|Redpanda Cloud|$0 (free tier)|
|BigQuery|~$0–5|
|Google Maps Platform|~$0 (within $200 credit)|
|Firebase|$0|
|**Total**|**~$65–80/month**|

### Terraform structure

```
terraform/
├── main.tf
├── variables.tf
├── modules/
│   ├── cloud_run/          # Reused per service
│   ├── cloud_sql/          # With PostGIS flag
│   ├── memorystore/
│   ├── cloud_storage/      # Photo buckets + CDN
│   ├── bigquery/
│   └── secret_manager/
└── environments/
    ├── dev/
    └── prod/
```

**PostGIS on Cloud SQL:**

```hcl
resource "google_sql_database_instance" "main" {
  name             = "pacemeet-postgres"
  database_version = "POSTGRES_15"
  region           = var.region

  settings {
    tier = "db-f1-micro"
    database_flags {
      name  = "cloudsql.enable_pg_cron"
      value = "on"
    }
  }
}

resource "google_sql_database" "app_db" {
  name     = "pacemeet"
  instance = google_sql_database_instance.main.name
}

# Enable PostGIS via startup script (run once after provisioning)
# psql -c "CREATE EXTENSION IF NOT EXISTS postgis;"
# psql -c "CREATE EXTENSION IF NOT EXISTS postgis_topology;"
```

---

## 14. CI/CD pipeline

### Mobile app builds — Expo EAS

```yaml
# .github/workflows/build-mobile.yml
name: Build and submit mobile app

on:
  push:
    branches: [main]
    paths: ["mobile/**"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - working-directory: mobile
        run: |
          npm install
          eas build --platform all --non-interactive  # Builds iOS + Android
          # eas submit --platform all  # Uncomment to auto-submit to stores
```

### Backend services — Cloud Run (same pattern as StockFolio)

```yaml
# .github/workflows/deploy-event-service.yml
name: Deploy event service

on:
  push:
    branches: [main]
    paths: ["services/event/**"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      - run: |
          gcloud auth configure-docker ${{ vars.REGION }}-docker.pkg.dev
          docker build -t ${{ vars.REGION }}-docker.pkg.dev/${{ vars.PROJECT_ID }}/pacemeet/event-service:${{ github.sha }} services/event/
          docker push ${{ vars.REGION }}-docker.pkg.dev/${{ vars.PROJECT_ID }}/pacemeet/event-service:${{ github.sha }}
          gcloud run deploy event-service \
            --image ${{ vars.REGION }}-docker.pkg.dev/${{ vars.PROJECT_ID }}/pacemeet/event-service:${{ github.sha }} \
            --region ${{ vars.REGION }} --platform managed --no-traffic
      - run: |
          # Smoke test the new revision
          curl --fail "$(gcloud run revisions describe --region ${{ vars.REGION }} \
            --format='value(status.url)' $(gcloud run revisions list \
            --service event-service --region ${{ vars.REGION }} \
            --format='value(metadata.name)' --limit=1))/health"
      - run: |
          gcloud run services update-traffic event-service \
            --to-latest --region ${{ vars.REGION }}
```

---

## 15. Out of scope (v1)

- Live location sharing during a run (real-time "where is everyone on the map")
- Strava / Garmin / Apple Health import
- In-app messaging or group chat
- Monetization (premium features, promoted events, venue partnerships)
- Running club / organization accounts
- Event recurring scheduling (e.g., "every Saturday 6am")
- Leaderboards or timed segment competitions
- Weather integration (show forecast for event date/location)
- Multi-language content (user-generated text in Thai auto-translated)
- Web app version (mobile only in v1)
- Offline map tiles (app requires connection to render maps)
