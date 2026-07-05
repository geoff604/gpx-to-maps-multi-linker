# gpx-to-maps-multi-linker
Convert high-density GPX tracking files into optimized, multi-stop Google Maps directional routing URLs.

# GPX to Google Maps Multi-Linker

A lightweight, high-performance, client-side web application designed to convert high-density GPX tracking files into optimized, multi-stop Google Maps directional routing URLs. Built entirely with vanilla JavaScript and styled using Tailwind CSS v4, this utility runs 100% locally in your browser with zero server dependencies or API keys required.

---

## 🌐 Try It Out Live

The application is fully deployed and ready to use! You can access the interactive web interface directly here:

👉 **[Launch GPX to Google Maps Multi-Linker](https://geoff604.github.io/gpx-to-maps-multi-linker/gpxtomaps.html)**

### Why use the live app?
* **Instant Conversion:** No installation, repository cloning, or local setup required.
* **100% Secure & Private:** Your GPS tracking data stays entirely within your local browser cache and never uploads to an external server.
* **Interactive Testing:** Easily toggle navigation modes and watch the telemetry panel update in real time.

Feel free to drop in your own `.gpx` tracks or use the built-in sample data sandbox to see the greedy optimization engine in action!

---

## 🚀 Features

- **Zero Server Overhead:** All file reading, DOM parsing, and coordinate filtering are executed inside your local browser cache. Your tracking data never leaves your machine.
- **Smart Downsampling Core:**
  - **Distance Proximity Filter:** Uses the Haversine formula to calculate true surface distances, discarding intermediate tracking jitter and static data noise (e.g., waiting at traffic lights) under 15 meters.
  - **Greedy Polyline Simplification Engine:** Automatically optimizes routes exceeding Google Maps' limits down to a stable 25-point ceiling. It identifies and preserves critical turn apexes and sharp directional shifts instead of blindly dropping points.
- **Flexible Navigation Modes:** Includes a responsive UI checkbox (defaults to checked) that automatically appends transit parameters to enforce Cycling/Bike directions.
- **Real-time Synchronization:** Toggling the routing mode instantly recalibrates and regenerates the active payload without forcing you to re-upload your `.gpx` file.
- **Telemetry UI Panel:** Provides immediate feedback showing:
  - Total raw coordinate points imported.
  - Compressed URL routing segments utilized.
  - Total string length of the generated hyperlink.
- **Sample Data Sandbox:** Includes a built-in demo generator so you can test link builds and compression analytics with a single click.

## 🛠️ How it Works

1. **Upload:** Drag & drop a `.gpx` file or use the standard file selection dialog.
2. **Parsing:** The application extracts standard coordinate tracking components (`<trkpt>`, `<rtept>`, or `<wpt>`).
3. **Filtering:** Points within 15 meters of the last approved marker are stripped.
4. **Compression:** If the coordinate count remains above 25, an interactive perpendicular deviation check identifies the most critical vector corners until exactly 25 sequential stops remain, securing full compatibility with mobile Google Maps application frameworks.
5. **Output:** A customized directional multi-stop URL is assembled alongside camera coordinate hints focused directly over your route's origin.

## 📦 Getting Started

Because this project consists of a single standalone HTML document, setting it up is instant:

1. Clone or download this repository.
2. Open `gpxtomaps.html` directly in any modern desktop or mobile web browser (Chrome, Firefox, Safari, Edge).
3. (Optional) Deploy the single file straight to GitHub Pages, Vercel, Netlify, or any static file host.

## ⚙️ Core Technical Configurations

If you want to modify the internal behaviors of the conversion script, look for these variables inside the script block:

- **Proximity Gate:** `minDistanceThresholdMeters = 15` — Controls how far a GPS device must travel before registering a new milestone.
- **Stop Limit:** `maxSafeCapacity = 25` — The maximum route stop index limit enforced to prevent browser link truncation or mobile application crashes.
- **Cycling Transit Flag:** `/data=!4m2!4m1!3e1` — The data transport parameter appended to indicate bicycling mode to the Google Maps engine.

## 🧠 Technical Algorithm Details

This section provides a deep-dive look into the two-pass geometric and spatial data-reduction pipeline used to optimize high-density GPS track files for browser compatibility.

---

### 1. Pass 1: Distance Proximity Filtering (Haversine Formula)

Raw GPX tracks often suffer from "data bloating" caused by high polling frequencies (e.g., 1Hz recording) or positional noise when stationary. Pass 1 steps sequentially through the coordinate array and screens out micro-movements.

#### Mathematical Foundation
To compute the true surface distance between two points on the Earth's surface, the application avoids Euclidean geometry (which distorts over large distances) and uses the **Haversine Formula**:

$$\Delta \text{lat} = \text{lat}_2 - \text{lat}_1$$
$$\Delta \text{lon} = \text{lon}_2 - \text{lon}_1$$
$$a = \sin^2\left(\frac{\Delta \text{lat}}{2}\right) + \cos(\text{lat}_1) \cdot \cos(\text{lat}_2) \cdot \sin^2\left(\frac{\Delta \text{lon}}{2}\right)$$
$$c = 2 \cdot \text{atan2}\left(\sqrt{a}, \sqrt{1-a}\right)$$
$$d = R \cdot c$$

Where:
* $R$ is the mean radius of the Earth ($6,371,000 \text{ meters}$).
* $d$ is the calculated geodesic surface distance.

#### Execution Logic
* **Step A:** Point $0$ (the origin) is explicitly approved and pushed to the `filteredPoints` stack.
* **Step B:** The engine iterates through subsequent coordinates. The distance $d$ is calculated between the current point $i$ and the *last successfully approved milestone* in `filteredPoints`.
* **Step C:** If $d \ge 15\text{ meters}$, point $i$ is approved and becomes the new baseline. If $d < 15\text{ meters}$, it is rejected as static drift or redundant data.
* **Step D (Boundary Lock):** The final coordinate ($N-1$) is checked. To ensure the destination is precise, the last element of the array is swapped out or appended to match the exact end marker of the raw file.

---

### 2. Pass 2: Greedy Polyline Simplification Engine

Google Maps enforces URL length limits and stop constraints within its routing engine (safely capped at 25 multi-stops for cross-platform stability). If the output of Pass 1 still exceeds 25 coordinates, Pass 2 reduces the set down to exactly 25 points while preserving critical directional turns.

This acts as a fixed-capacity, greedy inverse variation of the **Ramer-Douglas-Peucker (RDP)** algorithm. Instead of recursively subdividing using a distance threshold ($\epsilon$), it iteratively inserts vertices based on maximum cross-track error until the array capacity is satisfied.

#### Coordinate Flattening & Mercator Scaling
Because cross-track errors are evaluated over localized segments, computing spherical geometry recursively for every intermediate point is computationally expensive. The engine flattens coordinates into a temporary local 2D Euclidean plane ($X, Y$). 

To correct for longitudinal convergence near the poles (where lines of longitude get closer together), a Mercator scaling factor is computed using the segment's average latitude:

$$x = \text{lon} \cdot \cos\left(\text{lat}_{\text{avg}} \cdot \frac{\pi}{180}\right)$$
$$y = \text{lat}$$

#### Cross-Track Error Vector Projection
For any line segment spanning from fixed waypoint $A$ to fixed waypoint $B$, intermediate points $K$ are evaluated using linear vector projection:

1. Calculate the baseline displacement vectors: $\vec{u} = B - A$.
2. Compute the projection factor $t$ of the point $K$ relative to the line segment:
   $$t = \frac{(K - A) \cdot \vec{u}}{\|\vec{u}\|^2}$$
3. Clamp $t$ to the interval $[0, 1]$ to constrain calculations strictly to the bounds of the line segment (handling sharp switches or overshoots).
4. Identify the perpendicular projection point $P$ along the segment: $P = A + t\vec{u}$.
5. Calculate the cross-track error deviation squared: $\text{Distance}^2 = \|K - P\|^2$.

```text
K (Intermediate Track Point)
    /|
   / |  <-- Cross-Track Error (Perpendicular Distance)
  /  v
 A---P----------B  (Line Segment Shortcut)
```

#### Greedy Decimation Pipeline
* **Initialization:** An index tracking array is initialized, locking down index `0` (Start) and index `M-1` (End).
* **Iterative Loop:** While total locked indices $< 25$:
  1. The engine loops over all active segments $(A, B)$.
  2. For every intermediate coordinate $K$ skipped inside those segments, the cross-track error deviation squared is computed.
  3. The system scans globally to find the single coordinate across the entire track that yields the highest structural deviation squared (`globalMaxDistSq`).
  4. The index of this worst-offending vertex is added to the tracking array, turning a straight shortcut into a dynamic two-segment vertex that matches a sharp turn.
  5. The index tracking array is re-sorted sequentially.
* **Termination:** Once exactly 25 indices are accumulated, the loop exits. This guarantees that the final 25 points represent the absolute highest-fidelity outline of the map's geometry.

## Author

Geoff Peters ( https://github.com/geoff604/ ) is a software developer based in Vancouver, BC, Canada.
Find him on Linkedin at http://www.linkedin.com/in/gpeters

## 📝 License

This project is open-source and available under the MIT License. Feel free to modify, distribute, and integrate it into your mapping workflows!
