# Trip Planner (Hike & Bike)

Generate logical, real-world hiking and cycling routes around any city, visualize them on a map, split by days, check weather, and save/view your trip history.

- **Client**: React + React Router + Leaflet + Tailwind  
- **Server**: Node/Express + MongoDB (Mongoose) + OSRM + OpenStreetMap + optional OpenAI  
- **Ports**: Client `3000`, Server `3001`

> Hike distance: **5–15 km/day** · Bike distance: **30–60 km/day**  
> Hike routes are **loops** (start=finish). Bike routes can be **open**.

---

## ✨ Features

- **AI/Procedural Route Generation**
  - `/api/llm/generate` → Uses OpenAI to propose waypoints, then OSRM to snap to real paths; falls back to a procedural ring sized to your day limits.
  - `/api/routes/generate` → Fast procedural generator (no LLM), useful offline.
- **Map & Markers**
  - Leaflet polyline, Start/Finish markers, **Day break markers** (evenly split by km).
- **Trip History**
  - Save routes, list them in **My Trips**, click to **reopen on the map**.
- **Weather**
  - 3-day midday snapshot from OpenWeather for the route center.
- **Auth (simple)**
  - Login/Register; client stores token; history persists server-side.

---

## 🗂️ Monorepo Structure

```
/client
  src/
    api.js               # axios instance (baseURL, auth header)
    index.js, App.js
    pages/
      Home.jsx           # welcome + CTA
      Login.jsx, Register.jsx
      Planner.jsx        # map & generation/saving UI
      Trips.jsx          # history list
    styles/index.css     # Tailwind + button components
  tailwind.config.js
  package.json

/server
  src/
    server.js            # Express app bootstrap
    routes/
      routeAuth.js       # auth endpoints
      weather.js         # 3-day forecast proxy
      routeGen.js        # procedural generator
      routeLLM.js        # LLM+OSRM generator
      trips.js           # save/list/get trips
    utils/
      geo.js             # haversine, destPoint
      osrm.js            # OSRM helper (lon,lat ↔ lat,lon)
  package.json
```

---

## ⚙️ Prerequisites

- Node.js 18+ (has global `fetch`)
- npm or yarn
- MongoDB (Atlas or local)

---

## 🔑 Environment Variables

Create **`server/.env`**:
```env
PORT=3001
MONGO_URI=mongodb+srv://<user>:<pass>@<cluster>/<db>?retryWrites=true&w=majority
JWT_SECRET=change_me
JWT_EXPIRATION=5h

# Optional (for /api/llm/generate)
OPENAI_API_KEY=sk-...

# Optional (for /api/weather)
OWM_API_KEY=your_openweather_api_key

# Optional: override model
LLM_MODEL=gpt-4o-mini
```

Create **`client/.env.development`** (dev quality of life):
```env
REACT_APP_API=http://localhost:3001/api
```

> In production you can keep `REACT_APP_API=/api` if you reverse-proxy the backend.

---

## ▶️ Running Locally

**Install deps**
```bash
# server
cd server
npm i

# client
cd ../client
npm i
```

**Start both**
```bash
# terminal 1
cd server
npm run dev     # API on http://localhost:3001

# terminal 2
cd client
npm start       # React app on http://localhost:3000
```

Health check:
```
GET http://localhost:3001/api/test
→ { "ok": true, "message": "API is working" }
```

---

## 🧭 Front-End Pages

- **Home** – friendly welcome, “Open Planner” button (+ shows your name if available).
- **Planner** – city input, days, mode (Trek loop / Bike open), Generate/Reset/Weather/Save.
  - Draws polyline; shows Start, Finish, and **Day N** markers.
- **My Trips** – your saved trips list. Click a card → opens Planner with that saved route.

**Navigation details**

- Trips → Planner navigation uses `?open=<id>` or route state.  
- Planner loads a saved trip on mount if query param `open` is present.

---

## 🧠 API Overview

### Auth
```
POST /api/auth/register   { fullName, email, password }
POST /api/auth/login      { email, password }
```
Returns `{ token, user }`. Client stores token (localStorage) and sets `Authorization: Bearer <token>`.

### Route Generation

**LLM + OSRM**
```
POST /api/llm/generate
Body: { location: "Paris", days: 3, type: "bike" | "hike" }
Resp: {
  ok, center: { lat, lon, name }, 
  points: [[lat,lon], ...],
  meta: { days, type, totalKm, breaks: [indices] }
}
```
- Uses OpenAI to propose waypoint names (if `OPENAI_API_KEY`), geocodes them inside a city-bounded box, snaps with OSRM to **real roads/trails**. Ensures total km within your per-day bands; otherwise falls back to a procedural ring and re-snaps.

**Procedural only**
```
POST /api/routes/generate
Body: { location | lat+lon, days: 2, type: "Trek (loop)" | "Bike (open)" }
Resp: { ok, points, meta }
```
- Fast geometric ring/open shape sized to your day limits (not snapped to roads).

### Trips (History)
```
POST /api/trips
Body: { name, description, points, meta, center }
→ { ok, id }

GET  /api/trips
→ { ok, trips: [ { id, name, createdAt, meta, (center), ... } ] }  // lightweight

GET  /api/trips/:id
→ { ok, trip: { id, name, description, points, meta, center, createdAt } }
```

### Weather
```
GET /api/weather?lat=..&lon=..
→ { ok, city, days: [ { dt, dt_txt, temp, feels_like, description, icon } ] }
```
- 3× midday snapshots (today + next 2), °C (metric).

---

## 🗺️ Data Shapes

**Polyline**
```ts
points: Array<[lat: number, lon: number]>
```

**Trip meta**
```ts
meta: {
  days: number
  type: "bike" | "hike"
  totalKm: number
  breaks: number[]      // indices into points
}
```

**Center**
```ts
center: [lat, lon] | { lat, lon, name? }
```

---

## 🎨 Styling (Tailwind)

- Tailwind enabled via `@tailwind base; components; utilities;` in `client/src/styles/index.css`.
- Custom component classes for buttons: `.btn`, `.btn-primary`, `.btn-ghost`, `.btn-danger`, `.btn-sm`, `.btn-lg`.
- Dark mode variants provided (`dark:*`).

---

## 🧩 Key Implementation Notes

- **OSRM** wants `lon,lat` in the URL; we store `[lat, lon]`. Helpers flip it.
- **Distance logic** uses haversine on a spherical Earth (good accuracy for trip planning).
- **Hike vs Bike**
  - Hike routes **close the loop** by appending the first waypoint.
  - Bike routes may be **point-to-point**.
- **LLM fallback**: If OpenAI or geocoding fails, you still get a sized route via the procedural generator.

---

## 🛠️ Troubleshooting

- **401/Unauthorized when saving trips**  
  Ensure the client sets the token: `setAuthToken(token)` so axios sends `Authorization: Bearer <token>`.

- **Request failed / CORS**  
  In dev, set `client/.env.development` to:
  ```env
  REACT_APP_API=http://localhost:3001/api
  ```
  so the browser hits the API directly (no fragile proxy).

- **OSRM “bad response”**  
  Public OSRM can be flaky. The LLM route handler already falls back to a procedural line; try again or reduce frequency.

- **Nothing renders on Planner**  
  Verify response has `points.length > 1`. Check Network tab for `/api/llm/generate` or `/api/routes/generate`.

---

## 🚀 Deployment (outline)

- **Server**: Node 18+, set env vars, expose `PORT` or reverse-proxy via Nginx.
- **Client**:
  ```bash
  cd client
  npm run build
  ```
  Serve `build/`; proxy `/api/*` to the Node server; set `REACT_APP_API=/api` at build time.

---

## 📸 Screenshots (optional)

Place images under `docs/screenshots/` and reference:

```md
![Home](docs/screenshots/home.png)
![Planner](docs/screenshots/planner.png)
![My Trips](docs/screenshots/trips.png)
```

---

## ⚠️ Security

- Don’t commit secrets (`OPENAI_API_KEY`, `MONGO_URI`, `JWT_SECRET`). Use `.env` and `.gitignore`.
- Validate input server-side (IDs, lengths, etc.).

---

## 💡 Roadmap

- User-scoped trips (by userId)
- Elevation profile & difficulty
- POI-aware day splits (towns/stops)
- Export GPX/GeoJSON
- Shareable read-only trip view
