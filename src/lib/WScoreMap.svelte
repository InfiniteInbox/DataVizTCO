<script>
    import { onMount } from 'svelte';
    import { fade } from 'svelte/transition';
    import maplibregl from 'maplibre-gl';
    import 'maplibre-gl/dist/maplibre-gl.css';
    import { Protocol } from 'pmtiles';
    import katex from 'katex';
    import 'katex/dist/katex.min.css';

    let { onContinue } = $props();

    let mapContainer;
    let map;
    let mapLoaded = $state(false);
    let showAdvance = $state(false);

    let showLine1 = $state(false);
    let showLine2 = $state(false);
    /** @type {'legend'|'math'|'weights'} */
    let panel = $state('legend');

    /** @type {{ x: number, y: number, w: number } | null} */
    let hovered = $state(null);
    let hoveredKey = null;
    let hoveredSrc = null;

    // ---------------- AHP weights ----------------
    const LABELS = ['radar', 'precip', 'stream', 'hourly', 'daily'];
    const LABEL_TEXT = {
        radar: 'Weather radar',
        precip: 'Rain gauges',
        stream: 'River gauges',
        hourly: 'Hourly weather stations',
        daily: 'Daily weather stations'
    };
    // Principal eigenvectors of the Saaty matrices in the notebook.
    const PRESETS = {
        flood:   { radar: 0.326, precip: 0.254, stream: 0.254, hourly: 0.100, daily: 0.064, cr: 0.010 },
        general: { radar: 0.413, precip: 0.206, stream: 0.206, hourly: 0.110, daily: 0.065, cr: 0.002 },
        nowcast: { radar: 0.436, precip: 0.191, stream: 0.095, hourly: 0.193, daily: 0.085, cr: 0.010 }
    };

    // --- Methodology, rendered with KaTeX. Each step pairs one plain-English
    // sentence with the equation it describes, plus a gloss of every symbol.
    const tex = (s) => katex.renderToString(s, { displayMode: true, throwOnError: false });
    const EQ = {
        adequacy: tex(String.raw`S_k(c) \;=\; 1 - \exp\!\Big(-\tfrac{1}{\kappa_k}\underbrace{\textstyle\sum_{i\,\in\,c}\rho_{k,i}\, r_k\, m_i}_{\text{usable observations per day}}\Big)`),
        blend:    tex(String.raw`W_c \;=\; \sum_{k} w_k\, S_k(c)\,, \qquad \sum_k w_k = 1 \;\Longrightarrow\; W_c \in [0,1]`),
        kernel:   tex(String.raw`K(d) \;=\; \exp\!\big(-(d/L)^2\big), \qquad K(0)=1`),
        radar:    tex(String.raw`S_{\text{radar}}(c) \;=\; \rho_r(d_c)\cdot R_c \;\in\; [0,\,0.95]`),
        inequity: tex(String.raw`\text{blind spot}(c) \;=\; 1 - W_c`)
    };

    let preset = $state('flood');
    // Raw slider values; the weights the map uses are these normalised to sum to 1.
    let raw = $state({ ...PRESETS.flood });

    let total = $derived(LABELS.reduce((s, k) => s + raw[k], 0) || 1);
    /** normalised weights — always sum to exactly 1, which is what AHP requires */
    let w = $derived(Object.fromEntries(LABELS.map(k => [k, raw[k] / total])));
    let isCustom = $derived(
        !LABELS.every(k => Math.abs(w[k] - PRESETS[preset][k]) < 0.0005)
    );

    function applyPreset(name) {
        preset = name;
        raw = { ...PRESETS[name] };
    }

    // ---------------- colour ramp ----------------
    // magma, matching the notebook's matplotlib export
    const MAGMA = [
        [0.0, '#000003'], [0.1, '#140d35'], [0.2, '#3b0f6f'], [0.3, '#63197f'],
        [0.4, '#8c2980'], [0.5, '#b63679'], [0.6, '#dd4968'], [0.7, '#f6705b'],
        [0.8, '#fd9f6c'], [0.9, '#fdcf92'], [1.0, '#fbfcbf']
    ];

    // W = sum_k w_k * S_k, evaluated live in the style expression so moving a
    // slider recolours 1M cells on the GPU with no re-fetch. If a tileset
    // predates the sub-score export, fall back to the baked W_score.
    function wExpr(weights) {
        const blend = ['+', ...LABELS.map(k => ['*', weights[k], ['coalesce', ['get', `S_${k}`], 0]])];
        return ['case', ['has', 'S_radar'], blend, ['coalesce', ['get', 'W_score'], 0]];
    }
    function fillColor(weights) {
        return ['interpolate', ['linear'], wExpr(weights), ...MAGMA.flat()];
    }

    // Repaint both layers whenever the weights change.
    $effect(() => {
        if (!mapLoaded || !map) return;
        const expr = fillColor(w);
        for (const id of ['wscore-fine-fill', 'wscore-coarse-fill']) {
            if (map.getLayer(id)) map.setPaintProperty(id, 'fill-color', expr);
        }
    });

    // ---------------- map ----------------
    // Zoom handoff: a coarse (5 km, pre-aggregated) tileset carries the low
    // zooms where 1M polygons can't all fit in a tile — this is what removes the
    // stippled "gaps" — and the full 1 km tileset takes over once zoomed in.
    const COARSE_MAX = 8;

    function addFillLayer(id, source, sourceLayer, minzoom, maxzoom) {
        map.addLayer({
            id, type: 'fill', source, 'source-layer': sourceLayer,
            minzoom, maxzoom,
            paint: {
                'fill-color': fillColor(w),
                'fill-opacity': [
                    'case', ['boolean', ['feature-state', 'hover'], false], 1.0, 0.0
                ],
                'fill-opacity-transition': { duration: 1200 },
                // no antialiasing => adjacent cells butt together instead of
                // showing a hairline seam, so the surface reads as continuous
                'fill-antialias': false
            }
        });
    }

    function attachHover(layerId, source, sourceLayer) {
        map.on('mousemove', layerId, (e) => {
            if (!e.features.length) return;
            const f = e.features[0];
            map.getCanvas().style.cursor = 'crosshair';
            if (hoveredKey !== null) {
                map.setFeatureState({ source: hoveredSrc, sourceLayer, id: hoveredKey }, { hover: false });
            }
            hoveredKey = f.id;
            hoveredSrc = source;
            map.setFeatureState({ source, sourceLayer, id: hoveredKey }, { hover: true });

            // Recompute the hovered cell's score under the *current* weights.
            const p = f.properties;
            const val = p.S_radar != null
                ? LABELS.reduce((s, k) => s + w[k] * (Number(p[`S_${k}`]) || 0), 0)
                : Number(p.W_score) || 0;
            hovered = { x: e.point.x, y: e.point.y, w: val };
        });
        map.on('mouseleave', layerId, () => {
            map.getCanvas().style.cursor = '';
            if (hoveredKey !== null) {
                map.setFeatureState({ source: hoveredSrc, sourceLayer, id: hoveredKey }, { hover: false });
                hoveredKey = null;
            }
            hovered = null;
        });
    }

    onMount(() => {
        const protocol = new Protocol();
        maplibregl.addProtocol('pmtiles', protocol.tile);

        map = new maplibregl.Map({
            container: mapContainer,
            style: 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
            center: [-85, 50],
            zoom: 4,
            minZoom: 4,
            maxZoom: 12
        });

        map.on('load', async () => {
            const bounds = await fetch('/data/ontario_wscore_bounds.json').then(r => r.json());
            const ontarioBounds = [[bounds.west, bounds.south], [bounds.east, bounds.north]];

            // maxBounds clamps the viewport, not the content — so a pan box set to
            // Ontario exactly will CROP the province on any window whose aspect ratio
            // doesn't match it. Pad the box (neighbouring provinces may peek in at the
            // edges, which is fine) so the whole extent always fits on screen.
            const PAD_LNG = 8, PAD_LAT = 4;
            const panBounds = [
                [bounds.west - PAD_LNG, bounds.south - PAD_LAT],
                [bounds.east + PAD_LNG, bounds.north + PAD_LAT]
            ];
            map.setMaxBounds(panBounds);

            // Frame the full extent, then make that zoom the floor: the user can zoom
            // in freely but never back out past the whole-province view.
            map.fitBounds(ontarioBounds, { padding: 40, duration: 0 });
            map.setMinZoom(map.getZoom());

            // Is the coarse overview tileset present? (Optional but strongly
            // recommended — see WSCORE_TILES_AND_SETUP.md.)
            const hasCoarse = await fetch('/data/ontario_wscore_coarse.pmtiles', { method: 'HEAD' })
                .then(r => r.ok).catch(() => false);

            if (hasCoarse) {
                map.addSource('wscore-coarse', {
                    type: 'vector',
                    url: 'pmtiles:///data/ontario_wscore_coarse.pmtiles',
                    promoteId: 'cell_id'
                });
                // minzoom 0, not 4: the zoom that fits all of Ontario can fall below
                // 4 on a small window, and the layer would silently not render.
                addFillLayer('wscore-coarse-fill', 'wscore-coarse', 'wscore', 0, COARSE_MAX);
            }

            map.addSource('wscore', {
                type: 'vector',
                url: 'pmtiles:///data/ontario_wscore.pmtiles',
                promoteId: 'cell_id'
            });
            // Fine grid only takes over above the coarse handoff (or everywhere
            // if the coarse file isn't there).
            addFillLayer('wscore-fine-fill', 'wscore', 'wscore', hasCoarse ? COARSE_MAX : 0, 12);

            map.addLayer({
                id: 'wscore-outline',
                type: 'line',
                source: 'wscore',
                'source-layer': 'wscore',
                minzoom: hasCoarse ? COARSE_MAX : 0,
                paint: {
                    'line-color': '#ffffff',
                    'line-width': ['case', ['boolean', ['feature-state', 'hover'], false], 1.4, 0]
                }
            });

            // Fade the surface in.
            const reveal = ['case', ['boolean', ['feature-state', 'hover'], false], 1.0, 0.85];
            setTimeout(() => {
                for (const id of ['wscore-fine-fill', 'wscore-coarse-fill']) {
                    if (map.getLayer(id)) map.setPaintProperty(id, 'fill-opacity', reveal);
                }
            }, 400);

            attachHover('wscore-fine-fill', 'wscore', 'wscore');
            if (hasCoarse) attachHover('wscore-coarse-fill', 'wscore-coarse', 'wscore');

            mapLoaded = true;
            setTimeout(() => showLine1 = true, 1600);
            setTimeout(() => showLine2 = true, 4200);
            setTimeout(() => showAdvance = true, 6000);
        });

        return () => {
            map?.remove();
            maplibregl.removeProtocol('pmtiles');
        };
    });

    const pct = (v) => `${(v * 100).toFixed(1)}%`;
</script>

<div class="map-scene">
    <div class="map-el" bind:this={mapContainer}></div>

    {#if hovered}
        <div class="w-tooltip" style="left:{hovered.x}px; top:{hovered.y}px;">
            <strong>Score {hovered.w.toFixed(3)}</strong>
            <span>blind spot {(1 - hovered.w).toFixed(3)}</span>
        </div>
    {/if}

    <div class="narrative">
        {#if showLine1}
            <p class="line1" transition:fade={{ duration: 900 }}>It's not what it seems.</p>
        {/if}
        {#if showLine2}
            <p class="line2" transition:fade={{ duration: 900 }}>
                A place should sit above <span class="thr good">0.5</span> to count as adequately watched —
                <span class="thr best">0.7</span> and up is genuinely well monitored.
                Ontario's average is <span class="thr bad">0.386</span>.
            </p>
        {/if}
        {#if showLine2}
            <p class="line3" transition:fade={{ duration: 900 }}>This is evident in the disaster databases.</p>
        {/if}
    </div>

    <div class="panel">
        <div class="tabs">
            <button class:active={panel === 'legend'} onclick={() => panel = 'legend'}>Map</button>
            <button class:active={panel === 'weights'} onclick={() => panel = 'weights'}>Adjust weights</button>
            <button class:active={panel === 'math'} onclick={() => panel = 'math'}>How it works</button>
        </div>

        {#if panel === 'legend'}
            <div class="pane" transition:fade={{ duration: 150 }}>
                <h3>Hydrometeorological Intelligence Score</h3>
                <p>How well each square kilometre of Ontario is actually watched — radar, rain gauges, river gauges and weather stations combined into one 0–1 score.</p>
                <div class="scale">
                    <span>blind</span>
                    <div class="bar">
                        <span class="tick" style="left:50%"></span>
                        <span class="tick" style="left:70%"></span>
                    </div>
                    <span>well&nbsp;watched</span>
                </div>
                <div class="thresholds">
                    <span><b>0.5</b> adequate</span>
                    <span><b>0.7</b> well monitored</span>
                </div>
                <p class="hint">Hover any cell for its score. Zoom in for the full 1&nbsp;km grid.</p>
            </div>
        {/if}

        {#if panel === 'weights'}
            <div class="pane" transition:fade={{ duration: 150 }}>
                <h3>What should count most?</h3>
                <p>The score is a weighted average. Drag a slider to say how much each network matters — the map recolours instantly.</p>

                <div class="presets">
                    {#each Object.keys(PRESETS) as name}
                        <button class="preset" class:active={preset === name && !isCustom} onclick={() => applyPreset(name)}>{name}</button>
                    {/each}
                </div>

                {#each LABELS as k}
                    <div class="slider-row">
                        <div class="slider-head">
                            <span>{LABEL_TEXT[k]}</span>
                            <span class="wval">{pct(w[k])}</span>
                        </div>
                        <input type="range" min="0" max="0.6" step="0.002" bind:value={raw[k]} aria-label={LABEL_TEXT[k]} />
                    </div>
                {/each}

                <p class="cr">
                    {#if isCustom}
                        Custom weights (normalised to 100%).
                        <button class="link" onclick={() => applyPreset(preset)}>Reset to {preset}</button>
                    {:else}
                        Saaty pairwise weights, consistency ratio CR&nbsp;=&nbsp;{PRESETS[preset].cr.toFixed(3)} — well under the 0.10 limit, so the judgements are internally consistent.
                    {/if}
                </p>
            </div>
        {/if}

        {#if panel === 'math'}
            <div class="pane math" transition:fade={{ duration: 150 }}>
                <h3>How the score is built</h3>
                <p class="lede">Four steps take a pile of sensors and turn them into one number per square kilometre.</p>

                <div class="step">
                    <span class="n">1</span>
                    <div>
                        <h4>Grade each network out of 1</h4>
                        <p>For one square, ask of each network: <em>how much usable data actually informs this place each day?</em> Weight every sensor by how reliable it is and how often it reports — a gauge that reports hourly is worth 24× one that reports daily.</p>
                        <p>Crucially, a sensor doesn't only serve the square it stands in. A rain gauge is representative of the neighbourhood around it, so its contribution <strong>fades with distance</strong> rather than stopping at the property line:</p>
                        <div class="eq">{@html EQ.kernel}</div>
                        <p><b>L</b> is the <em>decorrelation length</em> — how far a network's readings stay representative. Rain decorrelates fast (~20 km; a downpour is a local thing), temperature far more slowly (~40 km). Each sensor's reliability is multiplied by <b>K</b> before being added up:</p>
                        <p class="aside">Rivers are the exception. A streamflow gauge tells you about its <em>watershed</em> — the places that drain through it — not about a circle of land, so it's spread along the basin instead. A gauge just over a drainage divide tells you almost nothing.</p>
                        <div class="eq">{@html EQ.adequacy}</div>
                        <ul class="gloss">
                            <li><b>ρ</b> — reliability: the share of readings that actually arrive clean.</li>
                            <li><b>r</b> — reports per day (24 for hourly, 1 for daily).</li>
                            <li><b>m</b> — redundancy discount: two gauges side by side see the same storm, so the second one counts for less.</li>
                            <li><b>κ</b> — how many observations count as "enough".</li>
                            <li><b>d</b> — distance from the sensor to the square being scored.</li>
                        </ul>
                        <p>The <b>exponential is the important part</b>: it gives <strong>diminishing returns</strong>. Going from zero gauges to one transforms a square; the tenth gauge barely moves it. Nothing here → 0. Thoroughly watched → approaches 1, never past it.</p>
                    </div>
                </div>

                <div class="step">
                    <span class="n">2</span>
                    <div>
                        <h4>Let radar play by the same rules</h4>
                        <p>Radar isn't a point sensor, so it gets its own bounded grade: how reliable the nearest tower is at this distance, times whether the square is inside its reach. Capping it at 0.95 stops radar — by far the densest source — from swamping everything else, which is what broke v1 of this score.</p>
                        <div class="eq">{@html EQ.radar}</div>
                    </div>
                </div>

                <div class="step">
                    <span class="n">3</span>
                    <div>
                        <h4>Decide how much each network counts, then blend</h4>
                        <p>Instead of guessing percentages, we compare networks <strong>two at a time</strong> ("how much more useful is radar than a river gauge?") and let AHP turn those pairwise judgements into weights. Multiply each grade by its weight, add them up.</p>
                        <div class="eq">{@html EQ.blend}</div>
                        <p>Because the weights sum to 1 and every grade sits in [0,1], the result is guaranteed to land in <strong>[0,1]</strong> — so any two squares in Ontario are directly comparable. AHP also audits itself: a <strong>consistency ratio</strong> below 0.10 means the pairwise calls don't contradict each other. Ours is <strong>0.010</strong>.</p>
                        <p class="tryit">Those <b>w</b>'s are exactly what the <button class="link" onclick={() => panel = 'weights'}>Adjust weights</button> sliders change.</p>
                    </div>
                </div>

                <div class="step">
                    <span class="n">4</span>
                    <div>
                        <h4>Flip it to find the blind spots</h4>
                        <div class="eq">{@html EQ.inequity}</div>
                        <p>Across <strong>1,078,363</strong> square kilometres the mean score is <strong>0.386</strong> — a quarter of Ontario sits below <strong>0.254</strong>. Best cell: 0.936. Worst: a flat <strong>0</strong>.</p>
                    </div>
                </div>
            </div>
        {/if}
    </div>

    {#if showAdvance && onContinue}
        <button class="advance-arrow" onclick={onContinue} transition:fade aria-label="Who reports these disasters?">→</button>
    {/if}
</div>

<style>
    .map-scene { position: relative; width: 100%; height: 100vh; }
    .map-el { width: 100%; height: 100%; }

    .w-tooltip {
        position: absolute;
        transform: translate(-50%, calc(-100% - 12px));
        background: white; color: #1a1a2e;
        padding: 0.45rem 0.65rem; border-radius: 6px;
        font-family: sans-serif; font-size: 0.8rem;
        display: flex; flex-direction: column; gap: 0.1rem;
        pointer-events: none; box-shadow: 0 6px 18px rgba(0,0,0,0.45);
        z-index: 40;
    }
    .w-tooltip strong { font-size: 0.9rem; }
    .w-tooltip span { color: #475569; }

    .narrative {
        position: absolute; top: 50%; right: 3rem;
        transform: translateY(-50%);
        max-width: 320px; color: white; font-family: sans-serif;
        z-index: 5; display: flex; flex-direction: column; gap: 1rem;
    }
    .narrative p { margin: 0; line-height: 1.4; text-shadow: 0 2px 12px rgba(0,0,0,0.85); }
    .narrative .line1 { font-size: 1.8rem; font-weight: 700; }
    .narrative .line2 { font-size: 1.15rem; font-weight: 500; color: #cbd5e1; }
    .narrative .line3 { font-size: 1.05rem; font-weight: 500; color: #94a3b8; }
    .thr { font-weight: 800; }
    .thr.good { color: #fdcf92; }
    .thr.best { color: #fbfcbf; }
    .thr.bad  { color: #dd4968; }

    .panel {
        position: absolute; top: 1.5rem; left: 1.5rem;
        width: min(360px, 34vw);
        max-height: calc(100vh - 3rem);
        display: flex; flex-direction: column;
        background: rgba(10,10,18,0.9);
        border: 1px solid rgba(138,180,255,0.22);
        border-radius: 12px;
        color: white; font-family: sans-serif;
        box-shadow: 0 10px 40px rgba(0,0,0,0.6);
        overflow: hidden;
    }

    .tabs { display: flex; border-bottom: 1px solid rgba(255,255,255,0.08); flex: none; }
    .tabs button {
        flex: 1; padding: 0.6rem 0.3rem;
        background: none; border: none; cursor: pointer;
        color: #94a3b8; font-size: 0.78rem; font-weight: 600;
        border-bottom: 2px solid transparent;
    }
    .tabs button.active { color: #8ab4ff; border-bottom-color: #8ab4ff; background: rgba(138,180,255,0.08); }

    .pane { padding: 1rem 1.2rem; overflow-y: auto; }
    .pane h3 { margin: 0 0 0.4rem; font-size: 1rem; }
    .pane p { margin: 0 0 0.6rem; font-size: 0.82rem; color: #cbd5e1; line-height: 1.5; }
    .pane strong { color: #f8fafc; }
    .pane em { color: #8ab4ff; font-style: normal; }
    .advance-arrow {
        position: absolute;
        bottom: 2rem;
        right: 2rem;
        width: 3.5rem; height: 3.5rem;
        border: none; border-radius: 50%;
        background: rgba(138, 180, 255, 0.9);
        color: #0a0a12;
        font-size: 1.6rem; font-weight: bold; cursor: pointer;
        display: flex; align-items: center; justify-content: center;
        z-index: 50;
        animation: pulse 2s infinite;
    }
    @keyframes pulse {
        0%, 100% { transform: scale(1); box-shadow: 0 0 0 0 rgba(138,180,255,0.5); }
        50% { transform: scale(1.08); box-shadow: 0 0 0 12px rgba(138,180,255,0); }
    }

    .hint { font-size: 0.75rem !important; color: #94a3b8 !important; margin-top: 0.6rem !important; }

    .scale { display: flex; align-items: center; gap: 0.5rem; font-size: 0.72rem; color: #cbd5e1; }
    .scale .bar {
        position: relative;
        flex: 1; height: 10px; border-radius: 5px;
        background: linear-gradient(to right, #000004, #51127c, #b73779, #fc8961, #fcfdbf);
    }
    .scale .tick {
        position: absolute; top: -2px; bottom: -2px;
        width: 2px; background: #fff;
        box-shadow: 0 0 3px rgba(0,0,0,0.8);
    }
    .thresholds {
        display: flex; justify-content: space-between;
        margin-top: 0.3rem; font-size: 0.68rem; color: #94a3b8;
    }
    .thresholds b { color: #e2e8f0; }

    /* --- weights --- */
    .presets { display: flex; gap: 0.4rem; margin: 0.5rem 0 0.9rem; }
    .preset {
        flex: 1; padding: 0.35rem; border-radius: 6px; cursor: pointer;
        border: 1px solid rgba(138,180,255,0.35);
        background: rgba(138,180,255,0.08); color: #8ab4ff;
        font-size: 0.75rem; font-weight: 600; text-transform: capitalize;
    }
    .preset.active { background: #8ab4ff; color: #0a0a12; }

    .slider-row { margin-bottom: 0.7rem; }
    .slider-head {
        display: flex; justify-content: space-between;
        font-size: 0.78rem; color: #cbd5e1; margin-bottom: 0.2rem;
    }
    .wval { color: #8ab4ff; font-weight: 700; font-variant-numeric: tabular-nums; }
    .slider-row input[type="range"] {
        width: 100%; accent-color: #8ab4ff; cursor: pointer;
    }
    .cr { font-size: 0.73rem !important; color: #94a3b8 !important; margin-top: 0.8rem !important; line-height: 1.45; }
    .link {
        background: none; border: none; padding: 0;
        color: #8ab4ff; font-size: inherit; font-weight: 600;
        cursor: pointer; text-decoration: underline;
    }

    /* --- math, in plain language --- */
    .math .step { display: flex; gap: 0.7rem; margin-bottom: 1rem; }
    .math .n {
        flex: none; width: 1.4rem; height: 1.4rem; border-radius: 50%;
        background: rgba(138,180,255,0.15); color: #8ab4ff;
        display: flex; align-items: center; justify-content: center;
        font-size: 0.75rem; font-weight: 700;
    }
    .math h4 { margin: 0.1rem 0 0.35rem; font-size: 0.86rem; color: #e2e8f0; }
    .lede { font-size: 0.8rem !important; color: #94a3b8 !important; }
    .eq {
        background: rgba(255,255,255,0.05);
        border-radius: 8px;
        padding: 0.45rem 0.3rem;
        margin: 0.35rem 0 0.55rem;
        overflow-x: auto;
    }
    .eq :global(.katex) { color: #f1f5f9; font-size: 0.92rem; }
    .gloss {
        margin: 0 0 0.6rem; padding-left: 0.9rem;
        font-size: 0.76rem; color: #94a3b8; line-height: 1.45;
    }
    .gloss li { margin-bottom: 0.22rem; }
    .gloss b { color: #8ab4ff; font-family: 'KaTeX_Math', Georgia, serif; font-size: 0.85rem; }
    .math p b { color: #e2e8f0; }

    .tryit { margin-bottom: 0.2rem !important; }
    .aside {
        border-left: 2px solid rgba(138,180,255,0.35);
        padding-left: 0.6rem;
        font-size: 0.78rem !important;
        color: #94a3b8 !important;
    }
</style>