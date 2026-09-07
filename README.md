# Sight Reduction Sphere

A self-contained browser visualization for celestial navigation. The app combines a Three.js 3D globe with a compact sight-reduction panel and a plotting sheet for the intercept method.

Open `index.html` directly in a modern browser. The page loads Three.js r128 and two fonts from CDNs, so an internet connection is needed unless those dependencies are vendored locally.

## What the application shows

- An Earth globe textured from procedurally drawn continent polygons.
- A virtual celestial sphere, equatorial plane, ecliptic plane, poles, Greenwich meridian, and starfield.
- The selected body's geographic position (GP), the observer's assumed position (AS), and the celestial zenith.
- The PZX spherical navigation triangle:
  - `P`: selected elevated pole, North or South.
  - `Z`: observer/zenith.
  - `X`: selected body's geographic position on Earth, or its corresponding celestial-sphere projection.
- Computed altitude `Hc`, true azimuth `Zn`, Greenwich hour angle `GHA`, declination, and local hour angle `LHA`.
- A plotting sheet centered on the dead-reckoning position, including the AS-to-intercept line and line of position (LOP).

## File architecture

`index.html` is intentionally a single-file application. It contains four layers:

1. **Markup and styling**
   - The header contains the triangle/ecliptic toggles and time animation control.
   - The left sidebar contains navigation inputs, body selection, positions, and sight data.
   - The center viewport hosts the Three.js renderer.
   - The right panel displays the plotting sheet and computed values.
   - The footer exposes interaction hints and the camera reset button.

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
   - The animation timer advances UTC by four minutes every 100 ms while playing.

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
RA = \operatorname{atan2}(\cos\epsilon\sin\lambda, \cos\lambda)
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
RA = \operatorname{atan2}(\sin\lambda\cos\epsilon - \tan\beta\sin\epsilon, \cos\lambda)
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

The longitude offset and negative X sign align the Earth texture and Greenwich meridian with the scene's chosen orientation. The Earth uses `radius = 1.5`; the celestial sphere uses `radius = 3.2`.

### Great-circle interpolation

`getGreatCirclePoints(v1, v2)` normalizes the endpoints, finds the central angle `omega`, and uses spherical linear interpolation (slerp):

$$
P(t) = \frac{\sin((1-t)\omega)}{\sin\omega}P_1
     + \frac{\sin(t\omega)}{\sin\omega}P_2
$$

The result is rescaled to the requested radius. It is used for the Earth and celestial-sphere triangle edges.

### Ecliptic plane

The ecliptic ring is sampled at `lambda` from `0` to `360` degrees with ecliptic latitude zero. Each sample is converted to equatorial coordinates:

$$
RA_\lambda = \operatorname{atan2}(\sin\lambda\cos\epsilon, \cos\lambda)
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

The plotting sheet is a local flat approximation. It is appropriate for the small `+-2` degree window used here, but it is not a global map projection.

## Rendering model

The scene has two dynamic groups:

- `dynamicGroup` is attached to `earthGroup` and contains markers, labels, Earth arcs, the Greenwich meridian, and the celestial triangle.
- `eclipticGroup` is attached to the scene and contains the ecliptic ring and translucent fan.

On every rebuild, children in these groups are removed and recreated. This keeps the implementation straightforward and ensures that changing time, body, observer position, pole, or sight data updates every dependent visual consistently.

The Earth texture is generated once by `generateEarthTexture()`. Its map projection is a simple equirectangular projection:

$$
 x = (lon + 180)\frac{W}{360}, \qquad
 y = (90 - lat)\frac{H}{180}
$$

The renderer uses a perspective camera, ambient light, a directional light, antialiased WebGL output, and a pixel-ratio cap of 2.

## Interaction model

- **Left drag:** orbit the camera around the scene.
- **Right drag:** pan the camera target.
- **Wheel or zoom slider:** change camera radius, clamped to `2.2..16`.
- **Reset view:** restores the initial radius, angles, and target.
- **Nav triangle toggle:** rebuilds without the Earth/celestial PZX triangle.
- **Ecliptic toggle:** rebuilds without the ecliptic ring, fan, and label.
- **Use current UTC time:** fills the date/time controls and rebuilds.
- **Use my location:** requests browser geolocation, updates AS and DR, then rebuilds.
- **Animate time:** advances the selected UTC timestamp by four minutes per tick.

## Accuracy and scope

This is an educational visualization, not a certified navigation calculator. Important limitations include:

- Approximate solar, lunar, and planetary ephemerides.
- No topocentric parallax correction for the Moon or planets.
- No atmospheric refraction, dip, index error, semi-diameter, or observer height corrections.
- No nutation, aberration, light-time, proper motion, or detailed Earth orientation corrections.
- The star catalog coordinates are intentionally compact and approximate.
- The Earth texture is illustrative rather than a geographic dataset.
- WebGL line width is effectively limited on many platforms, so primary triangle edges use tube geometry for visual weight.

For real navigation, compare results with an approved nautical almanac and apply the complete sight-correction workflow.

## Extension points

The most useful places to extend the application are:

- Add a body in `getBodyRaDec()` and `populateBodySelect()`.
- Replace the low-precision ephemerides while preserving the `{ ra, dec }` return contract.
- Add correction terms before `sightReduce()` or expose corrected altitude as a separate input.
- Replace `drawPlotSheet()` with a geodesic or chart projection if the plotting area grows beyond a few degrees.
- Split the inline script into modules once the application needs automated testing or multiple views.
