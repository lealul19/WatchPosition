<script>
    import { onMount } from 'svelte';
    import { isPointWithinRadius, getPreciseDistance, getGreatCircleBearing } from 'geolib';
  
    const TARGET = { latitude: 42.05913363079434, longitude: 19.51725368912721 };
    const THRESHOLD_METERS = 15;
  
    let hasGeo = typeof navigator !== 'undefined' && 'geolocation' in navigator;
    let watching = false;
    let watchId = null;
  
    let position = null;
    let distance = null;
    let bearing = null;
    let within = false;
    let errorMsg = '';
  
    // NEW: Kompass/Orientierung
    let hasOrientation = typeof window !== 'undefined' && 'DeviceOrientationEvent' in window; // NEW
    let heading = null;          // NEW: Kompasskurs (0=N, 90=E, …)
    let relBearing = null;       // NEW: Drehwinkel von Blickrichtung bis zum Ziel (0..360)
    let orientationEnabled = false; // NEW
  
    function toFixedOrDash(value, digits = 1) {
      return (value === null || value === undefined) ? '—' : value.toFixed(digits);
    }
  
    // NEW: Hilfen
    const clamp360 = (x) => ((x % 360) + 360) % 360; // NEW
    const sectorText = (deg) => {                    // NEW
      const dirs = ['N','NNO','NO','ONO','O','OSO','SO','SSO','S','SSW','SW','WSW','W','WNW','NW','NNW'];
      const idx = Math.round(clamp360(deg) / 22.5) % 16;
      return dirs[idx];
    };
  
    function startWatching() {
      if (!hasGeo || watching) return;
  
      try {
        watchId = navigator.geolocation.watchPosition(
          (pos) => {
            const { latitude, longitude, accuracy } = pos.coords;
            position = { lat: latitude, lon: longitude, accuracy };
  
            distance = getPreciseDistance(
              { latitude, longitude },
              TARGET
            );
  
            try {
              bearing = getGreatCircleBearing(
                { latitude, longitude },
                TARGET
              );
            } catch {
              const φ1 = (latitude * Math.PI) / 180;
              const φ2 = (TARGET.latitude * Math.PI) / 180;
              const Δλ = ((TARGET.longitude - longitude) * Math.PI) / 180;
              const y = Math.sin(Δλ) * Math.cos(φ2);
              const x = Math.cos(φ1) * Math.sin(φ2) -
                        Math.sin(φ1) * Math.cos(φ2) * Math.cos(Δλ);
              const θ = Math.atan2(y, x);
              bearing = ((θ * 180) / Math.PI + 360) % 360;
            }
  
            within = isPointWithinRadius(
              { latitude, longitude },
              TARGET,
              THRESHOLD_METERS
            );
  
            // NEW: relative Richtung aktualisieren
            updateRelBearing(); // NEW
          },
          (err) => {
            errorMsg = `Geolocation-Fehler (${err.code}): ${err.message}`;
          },
          {
            enableHighAccuracy: true,
            maximumAge: 0,
            timeout: 15000
          }
        );
        watching = true;
      } catch (e) {
        errorMsg = 'Geolocation konnte nicht gestartet werden.';
      }
    }
  
    function stopWatching() {
      if (watchId !== null && hasGeo) {
        navigator.geolocation.clearWatch(watchId);
        watchId = null;
        watching = false;
      }
    }
  
    // NEW: DeviceOrientation -> Heading
    function onOrientation(event) { // NEW
      // iOS liefert oft webkitCompassHeading (0=N, CW)
      const wk = event.webkitCompassHeading;
      if (typeof wk === 'number' && !Number.isNaN(wk)) {
        heading = clamp360(wk);
      } else {
        // fallback: alpha (0..360). Viele Browser: 0=N
        if (typeof event.alpha === 'number') {
          heading = clamp360(event.alpha);
        }
      }
      updateRelBearing();
    }
  
    function updateRelBearing() { // NEW
      if (bearing == null || heading == null) { relBearing = null; return; }
      relBearing = clamp360(bearing - heading); // wie weit (rechtsrum) bis zum Ziel
    }
  
    async function enableCompass() { // NEW
      if (!hasOrientation) { errorMsg = 'Dieses Gerät/Browser unterstützt keine Geräteausrichtung.'; return; }
      try {
        // iOS 13+ braucht explizite Erlaubnis
        if (typeof DeviceOrientationEvent !== 'undefined' &&
            typeof DeviceOrientationEvent.requestPermission === 'function') {
          const perm = await DeviceOrientationEvent.requestPermission();
          if (perm !== 'granted') { errorMsg = 'Kompasszugriff verweigert.'; return; }
        }
        window.addEventListener('deviceorientation', onOrientation, true);
        orientationEnabled = true;
      } catch (e) {
        errorMsg = 'Kompass konnte nicht aktiviert werden.';
      }
    }
  
    function disableCompass() { // NEW
      window.removeEventListener('deviceorientation', onOrientation, true);
      orientationEnabled = false;
    }
  
    onMount(() => {
      if (hasGeo) startWatching();
      // Kompass bewusst erst per Button aktivieren (Permissions) // NEW
      return () => { stopWatching(); disableCompass(); }; // NEW
    });
  </script>
  
  <style>
    :global(html, body) {
      height: 100%;
      margin: 0;
      background: #fff;
      font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, sans-serif;
      color: #111;
    }
  
    .wrap {
      min-height: 100dvh;
      display: grid;
      grid-template-rows: 1fr auto;
    }
  
    .center {
      display: grid;
      place-items: center;
      padding: 2rem;
    }
  
    .circle {
      position: relative; /* NEW: für Pfeil-Overlay */
      width: min(60vmin, 420px);
      height: min(60vmin, 420px);
      border-radius: 9999px;
      transition: background-color 200ms ease, box-shadow 200ms ease, transform 120ms ease;
      box-shadow: 0 20px 60px rgba(0,0,0,0.08), inset 0 0 0 8px rgba(0,0,0,0.04);
      overflow: hidden; /* NEW */
    }
    .circle.red { background: #e53935; }
    .circle.green { background: #43a047; }
  
    /* NEW: Pfeil, der zum Ziel zeigt (relativ zur Blickrichtung) */
    .arrow {
      position: absolute;
      left: 50%; top: 50%;
      width: 0; height: 0;
      transform: translate(-50%, -50%) rotate(var(--rot, 0deg));
      transition: transform 120ms ease;
      border-left: 16px solid transparent;
      border-right: 16px solid transparent;
      border-bottom: 40px solid rgba(255,255,255,0.95);
      filter: drop-shadow(0 4px 10px rgba(0,0,0,.25));
    }
  
    /* NEW: kleine Mittelpunktsmarke */
    .dot {
      position: absolute;
      left: 50%; top: 50%;
      width: 10px; height: 10px;
      transform: translate(-50%, -50%);
      border-radius: 9999px;
      background: rgba(0,0,0,.25);
    }
  
    .panel {
      padding: 1.25rem 2rem 2rem;
      border-top: 1px solid #eee;
      background: #fff;
    }
  
    .rows { display: grid; gap: .35rem; }
    .row { display: flex; justify-content: space-between; gap: 1rem; }
    .label { color: #555; }
    .value { font-variant-numeric: tabular-nums; font-weight: 600; }
  
    .controls {
      margin-top: 1rem;
      display: flex;
      gap: .5rem;
      flex-wrap: wrap; /* NEW */
    }
  
    button {
      padding: .6rem 1rem;
      border-radius: 10px;
      border: 1px solid #ddd;
      background: #fafafa;
      cursor: pointer;
    }
    button:hover { background: #f3f3f3; }
    button:active { transform: translateY(1px); }
  
    .hint { font-size: .9rem; color: #666; margin-top: .5rem; }
    .err { color: #b00020; margin-top: .5rem; }
  </style>
  
  <div class="wrap">
    <div class="center">
      <div
        class="circle {within ? 'green' : 'red'}"
        aria-label={within ? 'Grüner Punkt' : 'Roter Punkt'}
      >
        <!-- NEW: Pfeil zeigt Richtung zum Ziel relativ zur aktuellen Blickrichtung -->
        <div class="arrow" style="--rot: {relBearing ?? 0}deg;"></div>
        <div class="dot"></div>
      </div>
    </div>
  
    <div class="panel">
      <div class="rows">
        <div class="row">
          <div class="label">Entfernung zum Ziel</div>
          <div class="value">
            {distance === null ? '—' : `${distance.toFixed(1)} m`}
          </div>
        </div>
        <div class="row">
          <div class="label">Kurs zum Ziel (geo)</div>
          <div class="value">
            {bearing === null ? '—' : `${bearing.toFixed(0)}° (${sectorText(bearing)})`}
          </div>
        </div>
  
        <!-- NEW: Kompass & relative Richtung -->
        <div class="row">
          <div class="label">Kompasskurs (Heading)</div>
          <div class="value">
            {heading === null ? '—' : `${heading.toFixed(0)}° (${sectorText(heading)})`}
          </div>
        </div>
        <div class="row">
          <div class="label">Richtung zum Ziel (relativ)</div>
          <div class="value">
            {relBearing === null ? '—' : `${relBearing.toFixed(0)}° ${relBearing <= 180 ? 'rechts' : 'links'}`}
          </div>
        </div>
  
        <div class="row">
          <div class="label">Roter bzw. grüner Punkt</div>
          <div class="value">{within ? 'Grün (≤ 15 m)' : 'Rot (> 15 m)'}</div>
        </div>
        <div class="row">
          <div class="label">Aktuelle Position</div>
          <div class="value">
            {position
              ? `${position.lat.toFixed(6)}, ${position.lon.toFixed(6)} (±${toFixedOrDash(position.accuracy,0)} m)`
              : '—'}
          </div>
        </div>
        <div class="row">
          <div class="label">Zielkoordinaten</div>
          <div class="value">
            {TARGET.latitude}, {TARGET.longitude}
          </div>
        </div>
      </div>
  
      <div class="controls">
        <button on:click={startWatching} disabled={!hasGeo || watching}>Start (watchPosition)</button>
        <button on:click={stopWatching} disabled={!watching}>Stop</button>
  
        <!-- NEW: separater Button wegen iOS-Permission -->
        <button on:click={enableCompass} disabled={orientationEnabled}>Kompass aktivieren</button>
      </div>
  
      <div class="hint">
        Erlaube Standort & Bewegung/Orientierung (iOS: „Bewegung & Ausrichtung“). Richte das Handy waagerecht aus.
        Der Pfeil zeigt an, in welche Richtung du dich drehen musst, um zum Ziel zu blicken.
      </div>
      {#if !hasGeo}
        <div class="err">Geolocation wird von diesem Gerät/Browser nicht unterstützt.</div>
      {/if}
      {#if errorMsg}
        <div class="err">{errorMsg}</div>
      {/if}
    </div>
  </div>
  