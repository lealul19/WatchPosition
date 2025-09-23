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
  
    function toFixedOrDash(value, digits = 1) {
      return (value === null || value === undefined) ? '—' : value.toFixed(digits);
    }
  
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
              bearing = ( (θ * 180) / Math.PI + 360 ) % 360;
            }
  
            within = isPointWithinRadius(
              { latitude, longitude },
              TARGET,
              THRESHOLD_METERS
            );
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
  
    onMount(() => {
      if (hasGeo) startWatching();
      return () => stopWatching();
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
      width: min(60vmin, 420px);
      height: min(60vmin, 420px);
      border-radius: 9999px;
      transition: background-color 200ms ease, box-shadow 200ms ease, transform 120ms ease;
      box-shadow: 0 20px 60px rgba(0,0,0,0.08), inset 0 0 0 8px rgba(0,0,0,0.04);
    }
    .circle.red { background: #e53935; }   
    .circle.green { background: #43a047; } 
  
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
      <div class="circle {within ? 'green' : 'red'}" aria-label={within ? 'Grüner Punkt' : 'Roter Punkt'} />
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
          <div class="label">Kurs zum Ziel</div>
          <div class="value">
            {bearing === null ? '—' : `${bearing.toFixed(0)}°`}
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
      </div>
  
      <div class="hint">
        Erlaubnis für Standortzugriff im Browser/App erteilen. Für beste Ergebnisse: GPS aktiv & im Freien.
      </div>
      {#if !hasGeo}
        <div class="err">Geolocation wird von diesem Gerät/Browser nicht unterstützt.</div>
      {/if}
      {#if errorMsg}
        <div class="err">{errorMsg}</div>
      {/if}
    </div>
  </div>
  