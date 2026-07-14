<script>
    import { onMount } from 'svelte';
    import { fade } from 'svelte/transition';
    import maplibregl from 'maplibre-gl';
    import 'maplibre-gl/dist/maplibre-gl.css';

    let mapContainer;
    let map;

    let showLine1 = $state(false);
    let showLine2 = $state(false);

    onMount(() => {
        map = new maplibregl.Map({
            container: mapContainer,
            style: 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
            center: [-85, 50],
            zoom: 4
        });

        map.on('load', async () => {
            const bounds = await fetch('/data/ontario_wscore_bounds.json').then(r => r.json());

            // Image overlay: coordinates are [TL, TR, BR, BL] as [lng, lat]
            map.addSource('wscore', {
                type: 'image',
                url: '/data/ontario_wscore.png',
                coordinates: [
                    [bounds.west, bounds.north],
                    [bounds.east, bounds.north],
                    [bounds.east, bounds.south],
                    [bounds.west, bounds.south]
                ]
            });
            map.addLayer({
                id: 'wscore-layer',
                type: 'raster',
                source: 'wscore',
                paint: { 'raster-opacity': 0, 'raster-opacity-transition': { duration: 1200 } }
            });

            // Frame the province
            map.fitBounds(
                [[bounds.west, bounds.south], [bounds.east, bounds.north]],
                { padding: 40, duration: 0 }
            );

            // Fade the heatmap in, then reveal the narrative in two beats.
            setTimeout(() => map.setPaintProperty('wscore-layer', 'raster-opacity', 0.85), 400);
            setTimeout(() => showLine1 = true, 1600);
            setTimeout(() => showLine2 = true, 4200);
        });

        return () => map?.remove();
    });
</script>

<div class="map-scene">
    <div class="map-el" bind:this={mapContainer}></div>

    <div class="narrative">
        {#if showLine1}
            <p class="line1" transition:fade={{ duration: 900 }}>It's not what it seems.</p>
        {/if}
        {#if showLine2}
            <p class="line2" transition:fade={{ duration: 900 }}>This is evident in the disaster databases.</p>
        {/if}
    </div>

    <div class="legend">
        <h3>Hydrometeorological Intelligence Score</h3>
        <p>Density of high-quality, real-time monitoring — blended across radar, precip, streamflow, and temperature networks (AHP-weighted, flood scenario).</p>
        <div class="scale">
            <span>low</span>
            <div class="bar"></div>
            <span>high</span>
        </div>
    </div>
</div>

<style>
    .map-scene { position: relative; width: 100%; height: 100vh; }
    .map-el { width: 100%; height: 100%; }

    .narrative {
        position: absolute;
        top: 50%;
        right: 3rem;
        transform: translateY(-50%);
        max-width: 340px;
        color: white;
        font-family: sans-serif;
        z-index: 5;
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }
    .narrative p {
        margin: 0;
        line-height: 1.4;
        text-shadow: 0 2px 12px rgba(0,0,0,0.85);
    }
    .narrative .line1 { font-size: 1.8rem; font-weight: 700; }
    .narrative .line2 { font-size: 1.2rem; font-weight: 500; color: #cbd5e1; }

    .legend {
        position: absolute; top: 1.5rem; left: 1.5rem;
        background: rgba(10,10,18,0.85); color: white;
        padding: 1rem 1.25rem; border-radius: 10px; max-width: 300px;
        font-family: sans-serif;
    }
    .legend h3 { margin: 0 0 0.25rem; font-size: 1.05rem; }
    .legend p { margin: 0 0 0.75rem; font-size: 0.82rem; color: #cbd5e1; line-height: 1.4; }
    .scale { display: flex; align-items: center; gap: 0.5rem; font-size: 0.75rem; }
    .scale .bar {
        flex: 1; height: 10px; border-radius: 5px;
        background: linear-gradient(to right, #000004, #51127c, #b73779, #fc8961, #fcfdbf);
    }
</style>
