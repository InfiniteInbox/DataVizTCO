<script>
    import { onMount } from 'svelte';
    import { fade } from 'svelte/transition';
    import maplibregl from 'maplibre-gl';
    import 'maplibre-gl/dist/maplibre-gl.css';

    let { onContinue } = $props();

    let mapContainer;
    let map;

    let showText = $state(false);
    let showMap = $state(false);
    let showButton = $state(false);
    /** @type {{ x: number, y: number, name: string, network: string, quality: number } | null} */
    let hovered = $state(null);

    const networkColors = {
        streamflow: '#3b82f6',
        precip: '#10b981',
        hourly: '#f59e0b',
        daily: '#ef4444'
    };

    // What each layer in the legend actually means. Clicking the (i) opens one
    // of these; opening a second closes the first.
    const DESCRIPTIONS = {
        streamflow: 'A station measuring the discharge of a river or channel.',
        precip: 'A station measuring liquid precipitation.',
        hourly: 'A station reporting hourly temperature observations.',
        daily: 'A station reporting daily temperature observations.',
        'radar coverage': 'The standard 240 km radius of reflectivity a radar can pick up.',
        'radar tower': 'The location of the radar tower itself.',
        'clean data reported': 'The percentage of a station\'s data that is actually usable.'
    };

    /** @type {string | null} */
    let openInfo = $state(null);
    const toggleInfo = (key) => openInfo = openInfo === key ? null : key;

    // Rough Ontario bounding box (lon/lat) — the area we always want in frame.
    const ONTARIO_BOUNDS = [
        [-96.5, 41.2],   // south-west
        [-73.5, 57.2]    // north-east
    ];

    // maxBounds clamps the *viewport*, not the content: if the pan box is exactly
    // Ontario, a window whose aspect ratio doesn't match the province's will crop
    // it rather than letterbox it. So pad the pan box — neighbouring provinces and
    // states are allowed to show at the edges — and the whole of Ontario always fits.
    const PAD_LNG = 8;
    const PAD_LAT = 4;
    const PAN_BOUNDS = [
        [ONTARIO_BOUNDS[0][0] - PAD_LNG, ONTARIO_BOUNDS[0][1] - PAD_LAT],
        [ONTARIO_BOUNDS[1][0] + PAD_LNG, ONTARIO_BOUNDS[1][1] + PAD_LAT]
    ];

    // Average a polygon's exterior ring to get a usable "tower center" for the
    // near-circular radar coverage domes exported from the notebook.
    function ringCentroid(feature) {
        const ring = feature.geometry.coordinates[0];
        const n = ring.length - 1; // last vertex duplicates the first
        let sx = 0, sy = 0;
        for (let i = 0; i < n; i++) { sx += ring[i][0]; sy += ring[i][1]; }
        return [sx / n, sy / n];
    }

    const RADAR_ICON = `
        <svg viewBox="0 0 24 24" width="24" height="24" fill="none"
             stroke="#fbbf24" stroke-width="1.6" stroke-linecap="round">
            <path d="M8 19 L12 13 L16 19 Z" fill="rgba(251,191,36,0.25)" stroke="none"/>
            <path d="M12 13 V4.5"/>
            <path d="M6.8 8.6 A7 7 0 0 1 17.2 8.6"/>
            <path d="M4.2 6.2 A11 11 0 0 1 19.8 6.2"/>
            <circle cx="12" cy="13" r="2" fill="#fbbf24" stroke="none"/>
        </svg>`;

    onMount(() => {
        // Narrative timing: side text first, then the map, then the button.
        const t1 = setTimeout(() => showText = true, 400);
        const t2 = setTimeout(() => showMap = true, 2200);
        const t3 = setTimeout(() => showButton = true, 6500);

        map = new maplibregl.Map({
            container: mapContainer,
            style: 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
            // Start framed on the whole province rather than guessing a centre/zoom,
            // which crops Ontario on tall or narrow windows.
            bounds: ONTARIO_BOUNDS,
            fitBoundsOptions: { padding: 30 },
            maxZoom: 9,              // can zoom in to inspect clusters
            maxBounds: PAN_BOUNDS    // pan clamped, but loosely enough to fit it all
        });

        map.on('load', async () => {
            const stations = await fetch('/data/stations.geojson').then(r => r.json());
            const radar = await fetch('/data/radar_coverage.geojson').then(r => r.json());

            // Whatever zoom it took to fit the province becomes the floor, so the
            // user can zoom in but never back out past the full-Ontario view.
            map.fitBounds(ONTARIO_BOUNDS, { padding: 30, duration: 0 });
            map.setMinZoom(map.getZoom());

            // Radar coverage circles (drawn first, underneath)
            map.addSource('radar', { type: 'geojson', data: radar });
            map.addLayer({
                id: 'radar-fill',
                type: 'fill',
                source: 'radar',
                paint: { 'fill-color': '#60a5fa', 'fill-opacity': 0.12 }
            });
            map.addLayer({
                id: 'radar-line',
                type: 'line',
                source: 'radar',
                paint: { 'line-color': '#60a5fa', 'line-width': 1, 'line-opacity': 0.5 }
            });

            // Radar tower centers, marked with an icon.
            for (const f of radar.features) {
                const el = document.createElement('div');
                el.className = 'radar-marker';
                el.innerHTML = RADAR_ICON;
                el.title = `Radar tower: ${f.properties.radar_name}`;
                new maplibregl.Marker({ element: el, anchor: 'bottom' })
                    .setLngLat(ringCentroid(f))
                    .addTo(map);
            }

            // Station points, colored by network
            map.addSource('stations', { type: 'geojson', data: stations });
            map.addLayer({
                id: 'stations',
                type: 'circle',
                source: 'stations',
                paint: {
                    'circle-radius': 3.5,
                    'circle-color': [
                        'match', ['get', 'network'],
                        'streamflow', networkColors.streamflow,
                        'precip', networkColors.precip,
                        'hourly', networkColors.hourly,
                        'daily', networkColors.daily,
                        '#888'
                    ],
                    'circle-opacity': 0.85,
                    'circle-stroke-width': 0.5,
                    'circle-stroke-color': '#000'
                }
            });

            // Custom hover tooltip (pointer-events:none so it can't steal the hover)
            map.on('mousemove', 'stations', (e) => {
                map.getCanvas().style.cursor = 'pointer';
                const p = e.features[0].properties;
                hovered = {
                    x: e.point.x,
                    y: e.point.y,
                    name: p.name,
                    network: p.network,
                    quality: p.quality
                };
            });
            map.on('mouseleave', 'stations', () => {
                map.getCanvas().style.cursor = '';
                hovered = null;
            });
        });

        return () => {
            clearTimeout(t1);
            clearTimeout(t2);
            clearTimeout(t3);
            map?.remove();
        };
    });
</script>

<div class="map-scene">
    <div class="map-el" class:visible={showMap} bind:this={mapContainer}></div>

    {#if hovered}
        <div class="station-tooltip" style="left:{hovered.x}px; top:{hovered.y}px;">
            <strong>{hovered.name}</strong>
            <span>{hovered.network}{hovered.quality != null ? ` · ${(Number(hovered.quality) * 100).toFixed(0)}% clean data reported` : ''}</span>
        </div>
    {/if}

    {#if showText}
        <div class="narrative" transition:fade={{ duration: 900 }}>
            <p>It looks like we cover Ontario pretty well.</p>
            <p class="sub">
                Precipitation and streamflow stations are spread <strong>evenly across the province</strong> —
                verified by a Moran's <em>I</em> spatial-bias analysis.
            </p>
            <p class="sub">
                And Ontario's radar network reaches
                <strong class="stat">99.8%</strong> of the province's population.
            </p>
        </div>
    {/if}

    {#if showMap}
        <div class="legend" transition:fade={{ duration: 900 }}>
            <h3>Ontario monitoring coverage</h3>
            <p>Every weather and streamflow station, with radar reach. Tap any <span class="i-inline">i</span> to see what a layer measures.</p>

            {#each Object.entries(networkColors) as [net, color]}
                <div class="legend-item">
                    <div class="legend-row">
                        <span class="dot" style="background:{color}"></span>
                        <span class="legend-name">{net}</span>
                        <button class="info-btn" class:open={openInfo === net}
                            onclick={() => toggleInfo(net)}
                            aria-expanded={openInfo === net}
                            aria-label={`What does ${net} mean?`}>i</button>
                    </div>
                    {#if openInfo === net}
                        <p class="info-desc" transition:fade={{ duration: 120 }}>{DESCRIPTIONS[net]}</p>
                    {/if}
                </div>
            {/each}

            {#each [['radar coverage', 'ring'], ['radar tower', 'icon'], ['clean data reported', 'none']] as [key, mark]}
                <div class="legend-item">
                    <div class="legend-row">
                        {#if mark === 'ring'}
                            <span class="dot ring"></span>
                        {:else if mark === 'icon'}
                            <span class="radar-legend-icon">{@html RADAR_ICON}</span>
                        {:else}
                            <span class="dot pct">%</span>
                        {/if}
                        <span class="legend-name">{key}</span>
                        <button class="info-btn" class:open={openInfo === key}
                            onclick={() => toggleInfo(key)}
                            aria-expanded={openInfo === key}
                            aria-label={`What does ${key} mean?`}>i</button>
                    </div>
                    {#if openInfo === key}
                        <p class="info-desc" transition:fade={{ duration: 120 }}>{DESCRIPTIONS[key]}</p>
                    {/if}
                </div>
            {/each}
        </div>
    {/if}

    {#if showButton}
        <button class="advance-arrow" onclick={onContinue} transition:fade aria-label="Continue">→</button>
    {/if}
</div>

<style>
    .map-scene { position: relative; width: 100%; height: 100vh; }
    .map-el {
        width: 100%; height: 100%;
        opacity: 0;
        transition: opacity 1.2s ease;
    }
    .map-el.visible { opacity: 1; }

    .station-tooltip {
        position: absolute;
        transform: translate(-50%, calc(-100% - 12px));
        background: white;
        color: #1a1a2e;
        padding: 0.5rem 0.7rem;
        border-radius: 6px;
        max-width: 220px;
        font-family: sans-serif;
        font-size: 0.82rem;
        display: flex;
        flex-direction: column;
        gap: 0.15rem;
        pointer-events: none;
        box-shadow: 0 6px 18px rgba(0,0,0,0.45);
        z-index: 40;
    }
    .station-tooltip strong { font-size: 0.9rem; line-height: 1.2; }
    .station-tooltip span { color: #475569; text-transform: capitalize; }

    /* radar tower marker */
    :global(.radar-marker) {
        cursor: pointer;
        filter: drop-shadow(0 0 5px rgba(251, 191, 36, 0.8));
    }

    .narrative {
        position: absolute;
        top: 50%;
        right: 3rem;
        transform: translateY(-50%);
        max-width: 340px;
        color: white;
        font-family: sans-serif;
        z-index: 5;
    }
    .narrative p {
        font-size: 1.6rem;
        line-height: 1.4;
        font-weight: 600;
        margin: 0 0 1rem;
        text-shadow: 0 2px 12px rgba(0,0,0,0.8);
    }
    .narrative p.sub {
        font-size: 1rem;
        font-weight: 500;
        color: #e2e8f0;
        line-height: 1.5;
    }
    .narrative p.sub strong { color: #fff; }
    .narrative .stat { color: #fbbf24; font-weight: 700; }

    .legend {
        position: absolute; top: 1.5rem; left: 1.5rem;
        background: rgba(10,10,18,0.85); color: white;
        padding: 1rem 1.25rem; border-radius: 10px; max-width: 260px;
        font-family: sans-serif;
    }
    .legend h3 { margin: 0 0 0.25rem; font-size: 1.1rem; }
    .legend p { margin: 0 0 0.75rem; font-size: 0.85rem; color: #cbd5e1; }
    .legend-item { border-bottom: 1px solid rgba(255,255,255,0.05); }
    .legend-item:last-child { border-bottom: none; }
    .legend-row { display: flex; align-items: center; gap: 0.5rem; font-size: 0.85rem; margin: 0.3rem 0; text-transform: capitalize; }
    .legend-name { flex: 1; }

    .info-btn {
        flex: none;
        width: 1.1rem; height: 1.1rem;
        display: flex; align-items: center; justify-content: center;
        border: 1px solid rgba(138,180,255,0.45);
        border-radius: 50%;
        background: transparent;
        color: #8ab4ff;
        font-size: 0.7rem; font-weight: 700; font-style: italic;
        line-height: 1; cursor: pointer; padding: 0;
    }
    .info-btn:hover { background: rgba(138,180,255,0.18); }
    .info-btn.open { background: #8ab4ff; color: #0a0a12; }

    .info-desc {
        margin: 0 0 0.5rem 1.6rem !important;
        font-size: 0.76rem !important;
        line-height: 1.4;
        color: #cbd5e1 !important;
        text-transform: none;
    }

    .i-inline {
        display: inline-flex; align-items: center; justify-content: center;
        width: 0.95rem; height: 0.95rem;
        border: 1px solid rgba(138,180,255,0.45); border-radius: 50%;
        color: #8ab4ff; font-size: 0.62rem; font-weight: 700; font-style: italic;
        vertical-align: middle;
    }

    .dot.pct {
        background: transparent;
        border: 1px solid rgba(255,255,255,0.3);
        display: flex; align-items: center; justify-content: center;
        font-size: 0.55rem; color: #94a3b8; font-weight: 700;
    }
    .dot { width: 12px; height: 12px; border-radius: 50%; display: inline-block; }
    .dot.ring { background: transparent; border: 2px solid #60a5fa; }
    .radar-legend-icon { display: inline-flex; width: 16px; height: 16px; }
    .radar-legend-icon :global(svg) { width: 16px; height: 16px; }

    .advance-arrow {
        position: absolute;
        bottom: 2rem;
        right: 2rem;
        width: 3.5rem;
        height: 3.5rem;
        border: none;
        border-radius: 50%;
        background: rgba(138, 180, 255, 0.9);
        color: #0a0a12;
        font-size: 1.6rem;
        font-weight: bold;
        cursor: pointer;
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 50;
        animation: pulse 2s infinite;
    }

    @keyframes pulse {
        0%, 100% { transform: scale(1); box-shadow: 0 0 0 0 rgba(138,180,255,0.5); }
        50% { transform: scale(1.08); box-shadow: 0 0 0 12px rgba(138,180,255,0); }
    }
</style>