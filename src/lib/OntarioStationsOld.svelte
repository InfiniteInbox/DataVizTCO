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

    onMount(() => {
        // Narrative timing: side text first, then the map, then the button.
        const t1 = setTimeout(() => showText = true, 400);
        const t2 = setTimeout(() => showMap = true, 2200);
        const t3 = setTimeout(() => showButton = true, 6500);

        map = new maplibregl.Map({
            container: mapContainer,
            style: 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
            center: [-85, 50],   // Ontario-ish
            zoom: 4
        });

        map.on('load', async () => {
            const stations = await fetch('/data/stations.geojson').then(r => r.json());
            const radar = await fetch('/data/radar_coverage.geojson').then(r => r.json());

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
            <span>{hovered.network}{hovered.quality != null ? ` · ${(Number(hovered.quality) * 100).toFixed(0)}% complete` : ''}</span>
        </div>
    {/if}

    {#if showText}
        <div class="narrative" transition:fade={{ duration: 900 }}>
            <p>It looks like we cover Ontario pretty well.</p>
        </div>
    {/if}

    {#if showMap}
        <div class="legend" transition:fade={{ duration: 900 }}>
            <h3>Ontario monitoring coverage</h3>
            <p>Every weather and streamflow station, with radar reach.</p>
            {#each Object.entries(networkColors) as [net, color]}
                <div class="legend-row">
                    <span class="dot" style="background:{color}"></span>{net}
                </div>
            {/each}
            <div class="legend-row"><span class="dot ring"></span>radar coverage</div>
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

    .narrative {
        position: absolute;
        top: 50%;
        right: 3rem;
        transform: translateY(-50%);
        max-width: 320px;
        color: white;
        font-family: sans-serif;
        z-index: 5;
    }
    .narrative p {
        font-size: 1.6rem;
        line-height: 1.4;
        font-weight: 600;
        margin: 0;
        text-shadow: 0 2px 12px rgba(0,0,0,0.8);
    }

    .legend {
        position: absolute; top: 1.5rem; left: 1.5rem;
        background: rgba(10,10,18,0.85); color: white;
        padding: 1rem 1.25rem; border-radius: 10px; max-width: 260px;
        font-family: sans-serif;
    }
    .legend h3 { margin: 0 0 0.25rem; font-size: 1.1rem; }
    .legend p { margin: 0 0 0.75rem; font-size: 0.85rem; color: #cbd5e1; }
    .legend-row { display: flex; align-items: center; gap: 0.5rem; font-size: 0.85rem; margin: 0.25rem 0; text-transform: capitalize; }
    .dot { width: 12px; height: 12px; border-radius: 50%; display: inline-block; }
    .dot.ring { background: transparent; border: 2px solid #60a5fa; }

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
