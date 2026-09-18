# Sight Reduction Sphere

A self-contained browser visualization for celestial navigation. The app combines a Three.js 3D globe with a compact sight-reduction panel and a plotting sheet for the intercept method.

Open `index.html` directly in a modern browser. The page loads Three.js r128 and two fonts from CDNs, so an internet connection is needed for those and for the satellite Earth imagery (also CDN-hosted); the Sun/Moon/planet textures are embedded directly in the file and always render, even offline.

## A primer on celestial navigation

The sections below build up, from first principles, the ideas the app puts on screen: a position on Earth, a position on the sky, and the spherical triangle that links them.

### 1. Latitude and longitude

Latitude is easy because it can be measured locally, with no clock at all. The altitude of the celestial pole above the horizon equals the observer's latitude, so a fixed star near the pole (Polaris, in the northern hemisphere) or the Sun's altitude at local noon, corrected for declination, gives latitude directly from a single angle measurement. Sailors and astronomers have done this since antiquity with instruments like the astrolabe, cross-staff, and later the quadrant and sextant.

Longitude has no equivalent shortcut. Because Earth rotates, "where you are east-west" is really "what time it is where you are, compared with what time it is at a reference meridian." Without an accurate reference clock, there is nothing local to measure. This is the historical "longitude problem": for centuries, ships could find their latitude precisely but their east-west position only by dead reckoning, with errors that grew every day at sea. The 1707 Scilly naval disaster, which cost roughly 1,400–2,000 lives to a longitude error, made the problem a matter of national urgency. The 1714 Longitude Act established a British Board of Longitude and a prize for a practical solution. Two rival approaches emerged: the lunar-distance method (astronomical, needing no new hardware, but demanding lengthy calculation) and the marine chronometer, a clock accurate enough to keep reference time through a long voyage at sea. John Harrison's chronometers, culminating in H4, proved the chronometer approach on sea trials in the 1760s; both methods were in active use by mariners well into the 19th century, until chronometers became affordable and lunar distances fell out of use.

The precision required is unforgiving because Earth turns a full 360° in 24 hours:

$$
1^h \rightarrow 15^\circ, \qquad 1^m \rightarrow 15', \qquad 4^s \rightarrow 1'
$$

At the equator, 1 arcminute of longitude is about 1 nautical mile. So a clock error of just 4 seconds translates to roughly a 1 nautical mile position error, and a clock drifting by only a few seconds a day can put a ship many miles off course over a multi-week Atlantic crossing — enough to miss an island or misjudge a landfall in fog.

<p align="center"><img src="diagrams/lat-lon-grid.svg" alt="Globe graticule contrasting latitude parallels with longitude meridians" width="460"></p>

```mermaid
timeline
    title The quest for longitude
    1707 : Scilly naval disaster exposes the cost of dead reckoning
    1714 : British Longitude Act creates the Board of Longitude
    1730s : Lunar-distance method matured for practical use
    1761 : Harrison's H4 sea trial to Jamaica, accurate to seconds/day
    1767 : First Nautical Almanac published, with lunar-distance tables
    1773 : Harrison awarded the full longitude prize
    1884 : International Meridian Conference fixes Greenwich as 0°
```

### 2. Time — GMT and the Moon as a clock

Greenwich Mean Time (GMT), today formalized as Universal Time (UT), is simply the time of day on the Greenwich meridian. Every celestial-navigation calculation ultimately asks "what did the sky look like from Greenwich's meridian at the same instant the observer took a sight?" — which is why a reliable reference to GMT, however it is obtained, is the missing ingredient for longitude.

Before mechanical chronometers were trusted for long voyages, the Moon itself served as a natural clock. The Moon moves against the background stars at roughly 0.5° per hour — fast enough to notice, slow enough to measure precisely with a sextant. An observer measured the angular distance between the Moon and the Sun (or a reference star), then looked that angle up in precomputed tables — published from 1767 onward in Nevil Maskelyne's *Nautical Almanac* — which gave the corresponding GMT. Comparing that derived GMT with the observer's own local time (found from the Sun's altitude) yielded longitude, with no clock required beyond a stable local timekeeper for the duration of the sight.

```mermaid
flowchart LR
    A["Measure angle: Moon to Sun / reference star"] --> B["Look up angle in Nautical Almanac lunar-distance tables"]
    B --> C["Table yields GMT at that instant"]
    C --> D["Compare with local time from Sun's own altitude"]
    D --> E["Time difference -> longitude"]
```

### 3. Hour angle, GHA, and declination — coordinates for Sun, Moon, and planets

Just as latitude/longitude locates a point on Earth, declination/hour-angle locates a point on the sky. **Declination (Dec)** is the sky's equivalent of latitude: the angular distance of a body north or south of the celestial equator. **Greenwich Hour Angle (GHA)** is the sky's equivalent of longitude, with one key difference — it is always measured westward from the Greenwich meridian to the body's hour circle, and because Earth keeps turning, it constantly increases with time rather than staying fixed to a place on the ground.

<p align="center"><img src="diagrams/gha-dec.svg" alt="GHA measured westward from the Greenwich meridian, and Dec measured from the celestial equator" width="480"></p>

GHA and Dec for the Sun, Moon, and planets change from minute to minute as the bodies orbit and Earth rotates, so this application recomputes them continuously from the current UTC time (see "Time and sidereal rotation" and "Celestial-body positions" below).

### 4. Aries, SHA, and declination — coordinates for stars

Stars are, for navigational purposes, fixed on the sky, so it is more convenient to give each one a catalog coordinate that does not change with time, then add the time-varying part separately. The reference point is the **First Point of Aries (♈)**, the direction of the vernal equinox. **Sidereal Hour Angle (SHA)** is measured westward from Aries to a star's hour circle and, like Dec, stays essentially constant for a given star. GHA of Aries carries all of the time dependence:

$$
GHA_{\text{star}} = GHA_{\Upsilon} + SHA_{\text{star}} \pmod{360^\circ}
$$

<p align="center"><img src="diagrams/aries-sha.svg" alt="GHA of a star equals GHA of Aries plus the star's SHA" width="460"></p>

This is exactly the fixed-catalog approach this application uses internally, with J2000 right ascension/declination precessed to the current date (see "Stars" below).

### 5. Observed altitude, zenith distance, and the observer's two horizons

**Observed altitude (Hs)** is the angle measured with a sextant between a celestial body and the visible horizon. For the geometry of sight reduction, what actually matters is the angle from the body down to the **zenith** — the point directly overhead — called the **zenith distance**:

$$
\text{Zenith distance} = 90^\circ - Hs
$$

The subtlety is that there are two different horizons in play. The observer's real, "sensible" horizon is the plane tangent to Earth's surface at the observer's feet. The **celestial horizon** used in the geometry is a parallel plane passing through Earth's center. Because celestial bodies are so far away, the two planes point in essentially the same direction, so the distinction is negligible for the Sun, Moon, planets, and stars — the small residual (dip of the horizon, from the observer's height of eye) is corrected for separately and is not part of the spherical triangle itself.

<p align="center"><img src="diagrams/horizons.svg" alt="The observer's sensible horizon versus the celestial horizon through Earth's centre, with Hs and zenith distance" width="500"></p>

### 6. The noon shot

The classic **noon sight** finds latitude without needing a longitude or even an accurate clock. As the Sun crosses the observer's meridian at **Local Apparent Noon (LAN)**, its altitude reaches a daily maximum and its azimuth flips from increasing to decreasing (roughly east-of-south to west-of-south, or the equivalent in the southern hemisphere) — an event easy to detect by simply tracking the sextant altitude and waiting for it to stop rising.

<p align="center"><img src="diagrams/noon-altitude.svg" alt="Altitude curve peaking at Local Apparent Noon" width="520"></p>

At that instant, latitude follows directly from the observed altitude at meridian passage (Ho) and the Sun's declination. The flat cross-section below shows why: because the Sun's rays arriving at the observer and at Earth's centre are effectively parallel, the declination angle is the same at both places, and the observer's zenith direction (extended) passes straight through Earth's centre, so the two angles simply add:

<p align="center"><img src="diagrams/noon-shot-geometry.svg" alt="Angle-chasing diagram showing Lat equals declination plus 90 degrees minus Ho" width="560"></p>

$$
Lat = 90^\circ - Ho \pm Dec
$$

with the sign depending on whether the observer's zenith and the Sun's declination are on the same side of the equator (same name, subtract) or opposite sides (contrary name, add), and on which pole is elevated. This is why latitude-by-noon-sight was routine navigational practice long before the longitude problem was solved.

### 7. LHA

Everything above (GHA, Dec, SHA) is referenced to the Greenwich meridian. But the spherical triangle actually solved for a sight is built at the *observer's* meridian, so GHA must be shifted by the observer's own longitude to get the **Local Hour Angle (LHA)**:

$$
LHA = GHA + \lambda_{E} \qquad \text{or} \qquad LHA = GHA - \lambda_{W} \pmod{360^\circ}
$$

<p align="center"><img src="diagrams/gha-lha.svg" alt="LHA is GHA shifted by the observer's own longitude" width="460"></p>

LHA is the angle this application ultimately feeds into sight reduction alongside declination and assumed latitude (see "Sight-reduction formulae" below); it is the true angular separation, at the observer's own meridian, between the observer and the body.

## What the application shows

- An Earth globe rendered with a real satellite photo (NASA Blue Marble), an ocean specular mask, and a normal map for surface relief, layered over a procedurally drawn vector map that shows instantly and stays as an offline-safe fallback.
- A real-time day/night terminator on Earth, driven by the actual computed Sun position, with city lights fading in on the night side.
- Real photographic textures for the Sun, Moon, and visible planets, each with a small text label; unlisted bodies (stars) fall back to a flat catalog color.
- A virtual celestial sphere, equatorial plane, ecliptic plane, poles, Greenwich meridian, and starfield.
- The selected body's geographic position (GP), the observer's assumed position (AS), and the celestial zenith.
- The PZX spherical navigation triangle:
  - `P`: selected elevated pole, North or South.
  - `Z`: observer/zenith.
  - `X`: selected body's geographic position on Earth, or its corresponding celestial-sphere projection.
- Computed altitude `Hc`, true azimuth `Zn`, Greenwich hour angle `GHA`, declination, and local hour angle `LHA`.
- A circle of equal altitude (circle of position) drawn on the globe when an assumed position is present. The focus body's circle uses angular radius `90 - Hc`, or `90 - Hs` when an observed altitude is entered, and is centered on that body's GP. Visible bodies can draw independent circles from their own `Hs` fields, each in that body's palette color.
- A plotting sheet centered on the dead-reckoning position, including the AS-to-intercept line and line of position (LOP) for the focus body, plus one additional colored LOP per visible body that has an Hs entered.

## File architecture

`index.html` is intentionally a single-file application. It contains four layers:

1. **Markup and styling**
   - The header contains the triangle/ecliptic toggles and a kiosk-mode control.
   - The left sidebar contains navigation inputs, body selection, positions, and sight data.
   - The center viewport hosts the Three.js renderer.
   - The right panel displays the plotting sheet and computed values.
   - The footer exposes interaction hints and the camera reset button.
   - Kiosk mode hides the sidebar, right panel, and footer, centers the viewport full-bleed, and overlays a compact readout of UTC time, AS position, LHA, declination, Hc, and Zn.

2. **Astronomy math**
   - Angle normalization, Julian date, and J2000 epoch helpers.
   - Approximate Sun and Moon positions.
   - A fixed navigational star catalog with J2000 right ascension and declination.
   - Low-precision orbital elements for Venus, Mars, Jupiter, and Saturn.
   - Conversion from body right ascension/declination to GHA, LHA, GP latitude, and GP longitude.
   - Spherical sight reduction for altitude and azimuth.

3. **2D/3D geometry**
   - Canvas rendering for the plotting sheet.
   - Geographic latitude/longitude to Three.js Cartesian coordinates.
   - Great-circle interpolation for spherical triangle edges.
   - Dynamic markers, labels, meridians, planes, and arcs.

4. **Application lifecycle and events**
   - `rebuildScene()` is the central recomputation and redraw routine.
   - Input changes trigger a rebuild.
   - Camera events update only the camera; astronomy data is not recomputed for camera motion.
   - A one-second interval timer advances the displayed UTC time by one second and rebuilds the scene automatically, keeping the visualization and computed values live without user input.

## Data flow

The main computation path is:

```text
UTC date/time
  -> Julian date
  -> GMST / GHA of Aries
  -> selected body's RA and Dec
  -> body GHA and local hour angle
  -> sight reduction using assumed latitude
  -> Hc, Zn, Z, GP latitude, GP longitude
  -> 3D markers/arcs + right-side values + plotting sheet
```

`rebuildScene()` clears `dynamicGroup` and `eclipticGroup`, reads the current controls, runs this pipeline, and then rebuilds all dynamic geometry. Static scene objects such as the Earth mesh, celestial sphere, equatorial plane, lights, and starfield are created once during initialization.

## Angle and time conventions

- Internal trigonometric functions use radians.
- User-facing angles are degrees.
- `DEG = pi / 180` converts degrees to radians.
- `RAD = 180 / pi` converts radians to degrees.
- `norm360(d)` returns an angle in `[0, 360)`.
- `norm180(d)` returns an angle in `[-180, 180]`.
- Longitude inputs use positive east and negative west.
- Latitude inputs use positive north and negative south.
- GHA is represented as a positive westward angle.
- The displayed GP longitude is `-GHA`, normalized to the `[-180, 180]` range.

## Time and sidereal rotation

### Julian date

For a JavaScript `Date`:

$$
JD = \frac{\text{milliseconds since Unix epoch}}{86\,400\,000} + 2\,440\,587.5
$$

The J2000 epoch offset is expressed in Julian centuries:

$$
T = \frac{JD - 2\,451\,545.0}{36\,525}
$$

### Greenwich mean sidereal time

`gmstDeg(jd)` uses the standard polynomial approximation:

$$
GMST = 280.46061837
+ 360.98564736629(JD - 2451545.0)
+ 0.000387933T^2
- \frac{T^3}{38\,710\,000}
$$

The result is normalized to `[0, 360)`. In this application it is treated as the Greenwich hour angle of Aries, `gha0`.

For a body's right ascension `RA`:

$$
GHA = GMST - RA
$$

For observer longitude `\lambda` (positive east):

$$
LHA = GHA + \lambda
$$

Both results are normalized with `norm360()`.

## Celestial-body positions

### Sun

`sunPosition(jd)` uses a low-precision solar model:

1. Compute the mean longitude `L0` and mean anomaly `M`.
2. Compute the equation of center:

$$
C = A_1\sin M + A_2\sin(2M) + A_3\sin(3M)
$$

3. Set true ecliptic longitude to `L0 + C`.
4. Convert ecliptic longitude to equatorial right ascension and declination using the obliquity `epsilon`:

$$
RA = \mathrm{atan2}(\cos\epsilon\sin\lambda, \cos\lambda)
$$

$$
Dec = \arcsin(\sin\epsilon\sin\lambda)
$$

The obliquity is approximated by:

$$
\epsilon = 23.439291 - 0.0130042T
$$

### Moon

`moonPosition(jd)` uses a truncated periodic model for ecliptic longitude and latitude. The principal terms are summed in degrees, then transformed to equatorial coordinates:

$$
RA = \mathrm{atan2}(\sin\lambda\cos\epsilon - \tan\beta\sin\epsilon, \cos\lambda)
$$

$$
Dec = \arcsin(\sin\beta\cos\epsilon + \cos\beta\sin\epsilon\sin\lambda)
$$

Here `lambda` is ecliptic longitude and `beta` is ecliptic latitude.

### Stars

The star catalog contains fixed J2000 right ascensions and declinations. `precessJ2000ToDate()` applies the IAU-style zeta, z, and theta precession angles:

$$
\zeta = (2306.2181T + 0.30188T^2 + 0.017998T^3)\,\text{arcsec}
$$

$$
z = (2306.2181T + 1.09468T^2 + 0.018203T^3)\,\text{arcsec}
$$

$$
\theta = (2004.3109T - 0.42665T^2 - 0.041833T^3)\,\text{arcsec}
$$

The resulting coordinates are returned as date-of-observation RA and declination. Proper motion, nutation, aberration, and refraction are not modeled.

### Planets

Planet positions use approximate Keplerian elements valid approximately for 1800-2050:

1. Linearly evaluate orbital elements from `T`.
2. Compute mean anomaly `M = L - w`.
3. Solve Kepler's equation with Newton iteration:

$$
M = E - e\sin E
$$

with update:

$$
E_{n+1} = E_n + \frac{M - (E_n - e\sin E_n)}{1 - e\cos E_n}
$$

4. Convert the orbital-plane coordinates to heliocentric ecliptic Cartesian coordinates.
5. Subtract Earth's heliocentric position to obtain geocentric coordinates.
6. Rotate by the J2000 obliquity into equatorial coordinates.
7. Convert the vector to RA and declination and apply precession.

This is intended for visualization and educational sight reduction, not precision ephemeris work.

## Sight-reduction formulae

`sightReduce(lhaDeg, decDeg, latDeg)` implements the astronomical triangle altitude equation. Let:

- `L` be assumed latitude.
- `D` be declination.
- `H` be LHA.

Then computed altitude is:

$$
\sin H_c = \sin L\sin D + \cos L\cos D\cos H
$$

The implementation clamps the inverse-trigonometric input to `[-1, 1]` to avoid floating-point domain errors.

The interior azimuth angle at the observer is computed with:

$$
\cos Z = \frac{\sin D - \sin L\sin H_c}{\cos L\cos H_c}
$$

`Z` is constrained to `[0, 180]` degrees. The true azimuth is then selected by the sign of `sin(LHA)`:

$$
Z_n =
\begin{cases}
360 - Z, & \sin(LHA) > 0\\
Z, & \sin(LHA) \le 0
\end{cases}
$$

The implementation returns `{ Hc, Zn, Z }` in degrees.

### PZX side values

For the selected elevated pole, `PZX` values displayed in the information panel are:

- `PZ`, pole-to-observer arc, or colatitude:

$$
PZ = 90 - sL
$$

- `PX`, pole-to-GP arc, or polar distance:

$$
PX = 90 - sD
$$

- `ZX`, zenith distance:

$$
ZX = 90 - H_c
$$

where `s = +1` for North and `s = -1` for South. The meridian angle display uses normalized signed LHA to add an east/west label.

## Coordinate routines

### Geographic coordinates to Three.js

`latLonToVector3(lat, lon, radius)` maps latitude/longitude to a sphere whose north pole is Three.js `+Y`:

```text
phi   = (90 - latitude) * DEG
 theta = (longitude + 180) * DEG
 x = -r * sin(phi) * cos(theta)
 y =  r * cos(phi)
 z =  r * sin(phi) * sin(theta)
```

The longitude offset and negative X sign align the Earth texture and Greenwich meridian with the scene's chosen orientation. The Earth uses `radius = 1.2`; the celestial sphere uses `radius = 3.2`.

### Great-circle interpolation

`getGreatCirclePoints(v1, v2)` normalizes the endpoints, finds the central angle `omega`, and uses spherical linear interpolation (slerp):

$$
P(t) = \frac{\sin((1-t)\omega)}{\sin\omega}P_1
     + \frac{\sin(t\omega)}{\sin\omega}P_2
$$

The result is rescaled to the requested radius. It is used for the Earth and celestial-sphere triangle edges.

### Circle of equal altitude

`getSmallCirclePoints(centerDir, angularRadiusDeg, radius)` generates a circle of constant angular distance from a center direction, rather than a great circle. It builds an orthonormal basis `(c, u, v)` around the normalized center direction and samples:

$$
P(t) = r\big(\cos R \cdot c + \sin R \cdot (\cos t \cdot u + \sin t \cdot v)\big)
$$

for `t` from `0` to `2\pi`, where `R` is the angular radius in radians. This is the true circle of equal altitude: every point on it is exactly `90 - H` degrees from the body's GP, so an observation of altitude `H` places the observer somewhere on this circle. `rebuildScene()` draws the focus-body circle (as a tube, for the same line-width reasons as the triangle sides) when the assumed-position fields are populated, using `Hc` by default or the entered `Hs` when available. Visible bodies draw independent circles only when their own `Hs` is entered, centered on each body's GP.

### Ecliptic plane

The ecliptic ring is sampled at `lambda` from `0` to `360` degrees with ecliptic latitude zero. Each sample is converted to equatorial coordinates:

$$
RA_\lambda = \mathrm{atan2}(\sin\lambda\cos\epsilon, \cos\lambda)
$$

$$
Dec_\lambda = \arcsin(\sin\epsilon\sin\lambda)
$$

Those coordinates are converted to GHA/longitude with the same sidereal rotation used for celestial bodies, ensuring that the ecliptic ring shares the scene's Earth-fixed coordinate system.

## Plotting sheet

`drawPlotSheet(asLat, asLon, Zn, Hc)` draws a square covering `+-2` degrees around the DR position.

- Horizontal coordinate is longitude difference from DR.
- Vertical coordinate is latitude difference from DR.
- The vertical axis is inverted for canvas coordinates, so north appears upward.
- AS and DR are joined by a dashed line.

The intercept is calculated from observed altitude `Hs` and computed altitude `Hc`:

$$
a = (H_s - H_c) \times 60
$$

Because one minute of altitude corresponds to one nautical mile, `a` is in nautical miles. Positive values plot toward the selected body's azimuth; negative values plot away.

For a bearing `b` and distance `d` in nautical miles, the approximate position offset is:

$$
\Delta lat = \frac{d}{60}\cos b
$$

$$
\Delta lon = \frac{d}{60\cos(lat_{DR})}\sin b
$$

The LOP is drawn through the intercept point along the direction `Zn + 90` degrees, making it perpendicular to the azimuth.

`drawPlotSheet(asLat, asLon, Zn, Hc, bodySights)` additionally accepts a `bodySights` array built in `rebuildScene()` from every visible body with a valid Hs (each entry carries that body's own `Hc`, `Zn`, `hs`, and palette color). The same intercept/LOP math is applied per entry and rendered in the body's color with a small text label, independently of the focus body's own (green) intercept and LOP above.

The plotting sheet is a local flat approximation. It is appropriate for the small `+-2` degree window used here, but it is not a global map projection.

## Rendering model

The scene has two dynamic groups:

- `dynamicGroup` is attached to `earthGroup` and contains markers, labels, Earth arcs, the Greenwich meridian, and the celestial triangle.
- `eclipticGroup` is attached to the scene and contains the ecliptic ring and translucent fan.

On every rebuild, children in these groups are removed and recreated. This keeps the implementation straightforward and ensures that changing time, body, observer position, pole, or sight data updates every dependent visual consistently.

### Earth material

A procedural vector texture is generated once by `generateEarthTexture()` and applied immediately so the globe is never blank. Its map projection is a simple equirectangular projection:

$$
 x = (lon + 180)\frac{W}{360}, \qquad
 y = (90 - lat)\frac{H}{180}
$$

`latLonToVector3()` uses the same `(lon + 180)` / `(90 - lat)` convention, so this procedural texture and any replacement equirectangular photo align with markers and arcs without extra transforms.

Once loaded asynchronously from a CDN, three photographic maps are layered onto the same material (`earthMaterial`):

- A satellite color photo (`map`) replaces the procedural texture.
- A specular mask (`specularMap`) makes oceans glint while land stays matte.
- A normal map (`normalMap`) adds subtle surface relief under the existing Phong lighting.

`earthMaterial.onBeforeCompile` patches the stock Phong shader to add a day/night terminator: a world-space normal is compared against a `sunDirection` uniform, the lit side is left alone, the night side is dimmed, and a city-lights texture (`nightMap`) is additively blended in on the dark side only. Each `rebuildScene()` call recomputes the true sub-solar point from `sunPosition()` and updates both `sunDirection` and the scene's `sunLight` position, so the rendered terminator (and the Moon/planets' lit phase, since they share the same lighting) tracks the real Sun rather than a fixed light.

### Body markers

`addBodyMarker()` gives the Sun, Moon, and planets real photographic textures (`BODY_TEXTURE_URLS`) instead of flat-colored spheres; the Sun uses an unlit material (it is a light source), while the Moon/planets use the same Phong lighting as Earth. Bodies without a texture (stars) fall back to the flat `bodyPalette` color. Each visible/focused body also gets a small text label via `makeLabel()`.

The renderer uses a perspective camera, ambient light, a directional light (synced to the real Sun direction, see above), antialiased WebGL output, and a pixel-ratio cap of 2.

## Interaction model

- **Left drag:** orbit the camera around the scene.
- **Right drag:** pan the camera target.
- **Wheel or zoom slider:** change camera radius, clamped to `2.2..16`.
- **Reset view:** restores the initial radius, angles, and target.
- **Nav triangle toggle:** rebuilds without the Earth/celestial PZX triangle.
- **Ecliptic toggle:** rebuilds without the ecliptic ring, fan, and label.
- **Use current UTC time:** fills the date/time controls and rebuilds.
- **Use my location:** requests browser geolocation, updates AS and DR, then rebuilds.
- **Time offset range:** supports 6, 12, 48, and 72 hours, plus 1 week, 1 month, 3 months, 6 months, and 1 year.
- **Time offset slider:** applies a positive or negative hour offset to the entered UTC date/time for astronomy calculations and rebuilds without changing the base date/time fields.
- **Visible-body Hs fields:** entering an observed altitude next to a visible body draws that body's circle of equal altitude on the globe and its line of position on the plotting sheet, colored to match the body; independent of the main "Sight Observation" Hs field for the focus body.
- **Automatic time update:** the UTC time advances by one second every second, continuously rebuilding the scene.
- **Kiosk mode:** toggles a full-screen presentation layout with a centered globe and an overlaid data readout; click the exit control (top right) to return to the normal layout.

## Accuracy and scope

This is an educational visualization, not a certified navigation calculator. Important limitations include:

- Approximate solar, lunar, and planetary ephemerides.
- No topocentric parallax correction for the Moon or planets.
- No atmospheric refraction, dip, index error, semi-diameter, or observer height corrections.
- No nutation, aberration, light-time, proper motion, or detailed Earth orientation corrections.
- The star catalog coordinates are intentionally compact and approximate.
- The satellite Earth photo and Sun/Moon/planet photos are illustrative imagery, not navigational charts; the procedural vector map is a simplified fallback, not a geographic dataset.
- WebGL line width is effectively limited on many platforms, so primary triangle edges use tube geometry for visual weight.

For real navigation, compare results with an approved nautical almanac and apply the complete sight-correction workflow.

## Imagery and licensing

- Earth's satellite photo, specular mask, normal map, and night-lights texture are NASA Blue Marble-derived assets, loaded from the `three.js` example assets on a `jsdelivr` CDN mirror.
- Sun/Moon/planet photos originate from Solar System Scope (CC BY 4.0), downscaled and embedded directly in `index.html` as base64 `data:` URIs so they render in every browser without any network request or CORS dependency.
- All photographic assets load asynchronously behind the procedural fallback texture and flat palette colors, so the app remains usable if a request fails or the page is offline.

## Extension points

The most useful places to extend the application are:

- Add a body in `getBodyRaDec()` and `populateBodySelect()`.
- Replace the low-precision ephemerides while preserving the `{ ra, dec }` return contract.
- Add correction terms before `sightReduce()` or expose corrected altitude as a separate input.
- Replace `drawPlotSheet()` with a geodesic or chart projection if the plotting area grows beyond a few degrees.
- Split the inline script into modules once the application needs automated testing or multiple views.
