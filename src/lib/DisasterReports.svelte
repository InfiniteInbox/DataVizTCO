<script>
    import { onMount } from 'svelte';
    import { fade } from 'svelte/transition';
    import maplibregl from 'maplibre-gl';
    import 'maplibre-gl/dist/maplibre-gl.css';

    let mapContainer;
    let map;
    let mapLoaded = $state(false);
    let showLine1 = $state(false);
    let showLine2 = $state(false);

    /** counts per source_database, sorted desc */
    let orgCounts = $state([]);
    let totalReports = $state(0);
    /** orgs currently switched off in the legend */
    let muted = $state(new Set());

    /** @type {{ x:number, y:number, org:string, date:string, kind:string, count:number, dates:string[] } | null} */
    let hovered = $state(null);
    /** id of the polygon under the cursor, so only it lights up */
    let hoveredId = $state(null);

    // Palette: known agencies get a fixed colour so the legend never shuffles;
    // anything new in the CSV falls through to the spare ramp.
    const KNOWN = {
        NRCAN: '#60a5fa',   // blue
        OEM:   '#f59e0b',   // amber
        CDD:   '#10b981',   // green
        PSC:   '#ef4444',   // red
        ECCC:  '#a78bfa'    // violet
    };
    const SPARE = ['#f472b6', '#22d3ee', '#facc15', '#fb923c', '#4ade80'];
    let colorFor = $state({});

    const ONTARIO_BOUNDS = [[-96.5, 41.2], [-73.5, 57.2]];
    // Padded pan box — see the note in WScoreMap: clamping the viewport to Ontario
    // exactly crops the province whenever the window's aspect ratio differs.
    const PAN_BOUNDS = [[-104.5, 37.2], [-65.5, 61.2]];

    /** rough extent of a geometry, used only to order polygons big -> small */
    function bboxArea(geom) {
        if (!geom || geom.type === 'Point') return 0;
        let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
        const walk = (c) => {
            if (typeof c[0] === 'number') {
                if (c[0] < minX) minX = c[0];
                if (c[0] > maxX) maxX = c[0];
                if (c[1] < minY) minY = c[1];
                if (c[1] > maxY) maxY = c[1];
            } else for (const x of c) walk(x);
        };
        walk(geom.coordinates);
        return (maxX - minX) * (maxY - minY);
    }

    /** MapLibre 'match' expression: org -> colour, grey fallback */
    function colorExpr(map_) {
        const pairs = Object.entries(map_).flat();
        return pairs.length ? ['match', ['get', 'source_database'], ...pairs, '#94a3b8'] : '#94a3b8';
    }

    /** hide muted orgs on every layer */
    function applyFilter() {
        const hiddenOrgs = [...muted];
        const base = hiddenOrgs.length
            ? ['!', ['in', ['get', 'source_database'], ['literal', hiddenOrgs]]]
            : true;
        if (map.getLayer('reports-points')) map.setFilter('reports-points', ['all', ['==', ['geometry-type'], 'Point'], base]);
        if (map.getLayer('reports-fill'))   map.setFilter('reports-fill',   ['all', ['==', ['geometry-type'], 'Polygon'], base]);
        if (map.getLayer('reports-outline'))map.setFilter('reports-outline',['all', ['==', ['geometry-type'], 'Polygon'], base]);
    }

    function toggleOrg(org) {
        const next = new Set(muted);
        next.has(org) ? next.delete(org) : next.add(org);
        muted = next;
        if (mapLoaded) applyFilter();
    }

    const fmtDate = (p) => {
        const d = Number(p.day), m = Number(p.month), y = Number(p.year);
        const months = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];
        if (!y) return '—';
        if (!m) return `${y}`;
        return `${d ? d + ' ' : ''}${months[m - 1] ?? ''} ${y}`.trim();
    };

    onMount(() => {
        map = new maplibregl.Map({
            container: mapContainer,
            style: 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
            bounds: ONTARIO_BOUNDS,
            fitBoundsOptions: { padding: 30 },
            maxZoom: 12,
            maxBounds: PAN_BOUNDS
        });

        map.on('load', async () => {
            map.setMinZoom(map.getZoom());   // full-province view is the zoom-out floor
            const data = await fetch('/data/disaster_reports.geojson').then(r => r.json());

            // Tally organisations and assign colours.
            const tally = new Map();
            for (const f of data.features) {
                const org = f.properties.source_database ?? 'Unknown';
                tally.set(org, (tally.get(org) ?? 0) + 1);
            }
            const sorted = [...tally.entries()].sort((a, b) => b[1] - a[1]);
            let spare = 0;
            const cmap = {};
            for (const [org] of sorted) cmap[org] = KNOWN[org] ?? SPARE[spare++ % SPARE.length];
            colorFor = cmap;
            orgCounts = sorted.map(([org, n]) => ({ org, n, color: cmap[org] }));
            totalReports = data.features.length;

            // Big areas first, small areas last: MapLibre paints in feature order,
            // so this puts a province-wide polygon *underneath* a city-sized one
            // instead of burying it. Points live in their own layer above both.
            data.features.sort((a, b) => bboxArea(b.geometry) - bboxArea(a.geometry));

            map.addSource('reports', { type: 'geojson', data, generateId: true });

            // Polygons first (some reports are areas, not points), then points on top.
            map.addLayer({
                id: 'reports-fill',
                type: 'fill',
                source: 'reports',
                filter: ['==', ['geometry-type'], 'Polygon'],
                paint: {
                    'fill-color': colorExpr(cmap),
                    // Deliberately faint. Dozens of these overlap around the GTA;
                    // at 0.25 each, ten stacked areas saturate to a solid blob and
                    // you can't read any of them. The hovered one pops instead.
                    'fill-opacity': [
                        'case', ['boolean', ['feature-state', 'hover'], false], 0.5, 0.08
                    ],
                    'fill-opacity-transition': { duration: 120 }
                }
            });
            map.addLayer({
                id: 'reports-outline',
                type: 'line',
                source: 'reports',
                filter: ['==', ['geometry-type'], 'Polygon'],
                paint: {
                    'line-color': colorExpr(cmap),
                    // The outline carries the shape; the fill just tints it.
                    'line-width': [
                        'case', ['boolean', ['feature-state', 'hover'], false], 2.2, 0.9
                    ],
                    'line-opacity': [
                        'case', ['boolean', ['feature-state', 'hover'], false], 1, 0.55
                    ]
                }
            });
            map.addLayer({
                id: 'reports-points',
                type: 'circle',
                source: 'reports',
                filter: ['==', ['geometry-type'], 'Point'],
                paint: {
                    // grow slightly with zoom so dense clusters stay readable
                    'circle-radius': ['interpolate', ['linear'], ['zoom'], 4, 3.2, 9, 7],
                    'circle-color': colorExpr(cmap),
                    'circle-opacity': 0.85,
                    'circle-stroke-width': 0.6,
                    'circle-stroke-color': '#0a0a12'
                }
            });

            // How many reports sit under the cursor? A single dot can hide a dozen
            // coincident records, and areas overlap heavily around the GTA — so
            // query a small box around the pointer rather than a single pixel.
            const HIT_PAD = 5;
            function reportsAt(point) {
                const box = [
                    [point.x - HIT_PAD, point.y - HIT_PAD],
                    [point.x + HIT_PAD, point.y + HIT_PAD]
                ];
                const hits = map.queryRenderedFeatures(box, {
                    layers: ['reports-points', 'reports-fill'].filter(l => map.getLayer(l))
                });
                // A geojson source is tiled internally, so one polygon can come
                // back once per tile it straddles. De-duplicate on the record's
                // own identity, not on the rendered feature.
                const seen = new Map();
                for (const f of hits) {
                    const p = f.properties;
                    const key = `${p.source_ID}|${p.year}-${p.month}-${p.day}|${p.source_database}`;
                    if (!seen.has(key)) seen.set(key, p);
                }
                return [...seen.values()];
            }

            const clearHover = () => {
                if (hoveredId !== null) {
                    map.setFeatureState({ source: 'reports', id: hoveredId }, { hover: false });
                    hoveredId = null;
                }
            };

            for (const layer of ['reports-points', 'reports-fill']) {
                map.on('mousemove', layer, (e) => {
                    if (!e.features.length) return;
                    map.getCanvas().style.cursor = 'pointer';

                    // e.features is topmost-first, and because the data is sorted
                    // big -> small the topmost polygon is the *smallest* one under
                    // the cursor — i.e. the most specific report.
                    const f = e.features[0];
                    const p = f.properties;

                    if (hoveredId !== f.id) {
                        clearHover();
                        hoveredId = f.id;
                        map.setFeatureState({ source: 'reports', id: hoveredId }, { hover: true });
                    }

                    const all = reportsAt(e.point);
                    // Newest first, and only show a handful before we summarise.
                    const dates = [...new Set(all.map(fmtDate))]
                        .sort((a, b) => (b.match(/\d{4}/)?.[0] ?? '').localeCompare(a.match(/\d{4}/)?.[0] ?? ''))
                        .slice(0, 4);

                    hovered = {
                        x: e.point.x,
                        y: e.point.y,
                        org: p.source_database ?? 'Unknown',
                        date: fmtDate(p),
                        kind: layer === 'reports-fill' ? 'area' : 'point',
                        count: all.length,
                        dates
                    };
                });
                map.on('mouseleave', layer, () => {
                    map.getCanvas().style.cursor = '';
                    clearHover();
                    hovered = null;
                });
            }

            mapLoaded = true;
            // Let the reports land first, then make the argument about them.
            setTimeout(() => showLine1 = true, 1400);
            setTimeout(() => showLine2 = true, 3800);
        });

        return () => map?.remove();
    });

    let maxCount = $derived(orgCounts.length ? orgCounts[0].n : 1);
</script>

<div class="map-scene">
    <div class="map-el" bind:this={mapContainer}></div>

    {#if hovered}
        <div class="report-tooltip" style="left:{hovered.x}px; top:{hovered.y}px;">
            {#if hovered.count > 1}
                <strong class="tt-count">{hovered.count} reports here</strong>
                <div class="tt-dates">
                    {#each hovered.dates as d}<span>{d}</span>{/each}
                    {#if hovered.count > hovered.dates.length}
                        <span class="tt-more">+{hovered.count - hovered.dates.length} more</span>
                    {/if}
                </div>
            {:else}
                <strong class="tt-date">{hovered.date}</strong>
            {/if}
            <div class="tt-meta">
                <span class="tt-org" style="background:{colorFor[hovered.org] ?? '#94a3b8'}"></span>
                <span>{hovered.org}</span>
                <span class="tt-kind">{hovered.kind}</span>
            </div>
        </div>
    {/if}

    <div class="narrative">
        {#if showLine1}
            <p class="line1" transition:fade={{ duration: 900 }}>
                The reports clump to cities and to the places we already watch closely.
            </p>
        {/if}
        {#if showLine2}
            <p class="line2" transition:fade={{ duration: 900 }}>
                That is a <strong>cycle of disinvestment</strong>: where there are no sensors,
                floods go unreported. Unreported floods are never studied. And regions with no
                studied floods are never judged to need monitoring — so the sensors never arrive.
            </p>
        {/if}
    </div>

    <div class="panel">
        <h3>Who reports a disaster?</h3>
        <p class="sub">
            Every flood record in the databases, coloured by the agency that filed it.
            No single organisation sees the whole province — and the map you just looked at
            says a lot about <em>where</em> they see anything at all.
        </p>

        {#if orgCounts.length}
            <div class="total" transition:fade={{ duration: 400 }}>
                <strong>{totalReports.toLocaleString()}</strong> reports · {orgCounts.length} source{orgCounts.length === 1 ? '' : 's'}
            </div>

            <div class="bars">
                {#each orgCounts as { org, n, color }}
                    <button
                        class="bar-row"
                        class:muted={muted.has(org)}
                        onclick={() => toggleOrg(org)}
                        aria-pressed={!muted.has(org)}
                    >
                        <span class="bar-label">
                            <span class="swatch" style="background:{color}"></span>{org}
                        </span>
                        <span class="bar-track">
                            <span class="bar-fill" style="width:{(n / maxCount) * 100}%; background:{color}"></span>
                        </span>
                        <span class="bar-n">{n.toLocaleString()}</span>
                    </button>
                {/each}
            </div>
            <p class="hint">Click an agency to show or hide its reports. Dots are single points; outlined shapes are reported areas — hover one to bring it forward.</p>
        {:else}
            <p class="hint">Loading reports…</p>
        {/if}
    </div>
</div>

<style>
    .map-scene { position: relative; width: 100%; height: 100vh; }
    .map-el { width: 100%; height: 100%; }

    .report-tooltip {
        position: absolute;
        transform: translate(-50%, calc(-100% - 14px));
        background: white; color: #1a1a2e;
        padding: 0.45rem 0.65rem; border-radius: 8px;
        font-family: sans-serif;
        white-space: nowrap;
        pointer-events: none;
        box-shadow: 0 8px 24px rgba(0,0,0,0.5);
        z-index: 40;
    }
    .tt-date { display: block; font-size: 0.9rem; line-height: 1.2; }
    .tt-count { display: block; font-size: 0.9rem; line-height: 1.2; }
    .tt-dates {
        display: flex; flex-direction: column;
        margin-top: 0.2rem; font-size: 0.75rem; color: #334155; line-height: 1.35;
    }
    .tt-more { color: #94a3b8; }
    .tt-meta {
        display: flex; align-items: center; gap: 0.35rem;
        margin-top: 0.15rem; font-size: 0.7rem; color: #64748b;
    }
    .tt-org { width: 8px; height: 8px; border-radius: 50%; flex: none; }
    .tt-kind {
        text-transform: uppercase; letter-spacing: 0.04em;
        color: #94a3b8; font-size: 0.62rem;
    }

    .narrative {
        position: absolute;
        top: 50%;
        right: 3rem;
        transform: translateY(-50%);
        max-width: 330px;
        color: white;
        font-family: sans-serif;
        z-index: 5;
        display: flex;
        flex-direction: column;
        gap: 1.1rem;
        pointer-events: none;
    }
    .narrative p {
        margin: 0;
        line-height: 1.45;
        text-shadow: 0 2px 12px rgba(0,0,0,0.85);
    }
    .narrative .line1 { font-size: 1.45rem; font-weight: 700; }
    .narrative .line2 { font-size: 1rem; font-weight: 500; color: #cbd5e1; }
    .narrative .line2 strong { color: #fbbf24; font-weight: 700; }

    .panel {
        position: absolute; top: 1.5rem; left: 1.5rem;
        width: min(340px, 33vw);
        max-height: calc(100vh - 3rem);
        overflow-y: auto;
        background: rgba(10,10,18,0.9);
        border: 1px solid rgba(138,180,255,0.22);
        border-radius: 12px;
        padding: 1rem 1.2rem;
        color: white; font-family: sans-serif;
        box-shadow: 0 10px 40px rgba(0,0,0,0.6);
    }
    .panel h3 { margin: 0 0 0.4rem; font-size: 1.05rem; }
    .sub { margin: 0 0 0.8rem; font-size: 0.82rem; line-height: 1.5; color: #cbd5e1; }
    .sub em { color: #8ab4ff; font-style: normal; font-weight: 600; }

    .total {
        font-size: 0.8rem; color: #94a3b8;
        padding-bottom: 0.6rem; margin-bottom: 0.6rem;
        border-bottom: 1px solid rgba(255,255,255,0.08);
    }
    .total strong { color: #f8fafc; font-size: 1rem; }

    .bars { display: flex; flex-direction: column; gap: 0.35rem; }
    .bar-row {
        display: grid;
        grid-template-columns: 6.2rem 1fr 2.6rem;
        align-items: center;
        gap: 0.5rem;
        width: 100%;
        padding: 0.25rem 0.15rem;
        background: none; border: none; border-radius: 5px;
        cursor: pointer; text-align: left;
        transition: opacity 0.15s, background 0.15s;
    }
    .bar-row:hover { background: rgba(255,255,255,0.05); }
    .bar-row.muted { opacity: 0.35; }

    .bar-label {
        display: flex; align-items: center; gap: 0.35rem;
        font-size: 0.78rem; color: #e2e8f0; font-weight: 600;
        overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
    }
    .swatch { width: 10px; height: 10px; border-radius: 3px; flex: none; }
    .bar-track {
        height: 8px; border-radius: 4px;
        background: rgba(255,255,255,0.07); overflow: hidden;
    }
    .bar-fill { display: block; height: 100%; border-radius: 4px; }
    .bar-n {
        font-size: 0.74rem; color: #94a3b8; text-align: right;
        font-variant-numeric: tabular-nums;
    }

    .hint { margin: 0.8rem 0 0; font-size: 0.72rem; color: #94a3b8; line-height: 1.45; }
</style>