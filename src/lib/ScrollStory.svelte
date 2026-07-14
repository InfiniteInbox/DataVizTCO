<script>
    import { onMount } from 'svelte';
    import { fade } from 'svelte/transition';
    import scrollama from 'scrollama';
    import * as d3 from 'd3';
    import OntarioStations from '$lib/OntarioStations.svelte';
    import WScoreMap from '$lib/WScoreMap.svelte';
    import DisasterReports from '$lib/DisasterReports.svelte';

    let canvas;
    let animationFrameId;

    // Droplet radius encodes the damage cost of each event. Area is scaled
    // roughly proportional to cost (r ∝ √cost) so the visual weight is honest,
    // then normalised so Hazel (the costliest) reads ~26px.
    const COST_M = { hazel: 1300, 'july-2013': 1000, 'lake-ontario': 550 }; // $ millions
    const RADIUS_SCALE = 26 / Math.sqrt(1300);
    const costRadius = (id) => Math.sqrt(COST_M[id]) * RADIUS_SCALE;

    let eventDroplets = $state([
        { id: 'hazel', name: 'Hurricane Hazel', year: 1954, damage: '$1.3 billion in damage', x: 200, y: -100, vy: 3, a: costRadius('hazel') },
        { id: 'july-2013', name: 'July 2013 Flood', year: 2013, damage: '$1 billion in damage', x: 500, y: -300, vy: 2, a: costRadius('july-2013') },
        { id: 'lake-ontario', name: 'Lake Ontario Freshet', year: 2017, damage: '$550 million in damage', x: 800, y: -200, vy: 3, a: costRadius('lake-ontario') }
    ]);

    let hoveredEventId = $state(null);
    let hoveredEventData = $derived(eventDroplets.find(drop => drop.id === hoveredEventId));
    let introView;
    let currentStep = $state(0);

    // Game state
    let gameVisible = $state(false);
    let gameStarted = $state(false);
    let gameComplete = $state(false);
    let gameVerdict = $state(false);
    let userGuess = $state(null);
    let currentRound = $state(0);
    let correctCount = $state(0);
    let scene = $state(0);

    // Game rounds are loaded from static/data/game_rounds.json (exported from
    // the data-prep notebook). Each round: { name, flooded, event, blurb,
    // dates[], precip[] (mm/day), flow[] (m3/s/day) }.
    /** @type {Array<{ name?: string, flooded: boolean, event: string|null, blurb?: string, dates?: string[], precip: number[], flow: number[] }>} */
    let rounds = $state([]);
    let totalRounds = $derived(rounds.length);

    // Current round + axis extents, used to label the game charts.
    // Every series can contain nulls: days where the record simply doesn't exist.
    // Those are NOT zeros, and must never be plotted as zeros — a missing gauge
    // reading is the whole point of the "We may never know" round.
    const finite = (arr) => (arr ?? []).filter((v) => typeof v === 'number' && isFinite(v));

    let round = $derived(rounds[currentRound]);
    let precipMax = $derived(round ? Math.max(...finite(round.precip), 1) : 1);
    let flowVals = $derived(round ? finite(round.flow) : []);
    let flowMax = $derived(flowVals.length ? Math.max(...flowVals) : 1);
    let flowMin = $derived(flowVals.length ? Math.min(...flowVals) : 0);
    let roundDays = $derived(round ? round.precip.length : 0);
    // How much of this round is actually observed?
    let missingDays = $derived(round
        ? round.precip.filter(v => v == null).length + round.flow.filter(v => v == null).length
        : 0);

    // Friendly axis tick formatting.
    const fmtTick = (v) => (v >= 100 ? Math.round(v).toString()
        : v >= 10 ? v.toFixed(0)
        : v.toFixed(1));

    // Precip bars, scaled so the tallest observed day fills the axis.
    // Missing days come back flagged, so the template can draw a gap instead of a bar.
    /** @param {Array<number|null>} precip */
    function precipHeights(precip) {
        const max = Math.max(...finite(precip), 1);
        return (precip ?? []).map(v =>
            v == null ? { missing: true, h: 0 } : { missing: false, h: (v / max) * 100 }
        );
    }

    // Build the hydrograph (0..100 x, 0..50 y viewBox). Nulls BREAK the line into
    // separate segments rather than interpolating across them — drawing a straight
    // line over a data gap would invent a measurement that was never taken.
    /** @param {Array<number|null>} flow */
    function flowSegments(flow) {
        const vals = finite(flow);
        if (!flow || vals.length === 0) return { paths: [], points: [] };
        const max = Math.max(...vals);
        const min = Math.min(...vals);
        const range = (max - min) || 1;
        const n = flow.length;
        const xy = (v, i) => ({
            x: n > 1 ? (i / (n - 1)) * 100 : 50,
            y: 48 - ((v - min) / range) * 44  // higher flow -> higher line
        });

        const paths = [];
        const points = [];
        let run = [];
        const flush = () => {
            if (run.length >= 2) {
                paths.push(run.map((p, k) => `${k === 0 ? 'M' : 'L'} ${p.x.toFixed(1)} ${p.y.toFixed(1)}`).join(' '));
            } else if (run.length === 1) {
                points.push(run[0]);   // a lone reading between two gaps
            }
            run = [];
        };
        flow.forEach((v, i) => {
            if (v == null || !isFinite(v)) flush();
            else run.push(xy(v, i));
        });
        flush();
        return { paths, points };
    }

    // Canada map state
    let canadaPath = $state('');
    let mapReady = $state(false);

    // Real, sourced events where missing or broken monitoring left people without
    // warning. Each entry links to the reporting. lon/lat are projected to px/py
    // once the Canada basemap loads.
    let monitoringFailures = $state([
        {
            id: 'flinflon', lon: -101.86, lat: 54.77, px: 0, py: 0,
            name: 'Flin Flon, Manitoba — 2025',
            desc: 'Wildfires nearly encircled the town, but it sits outside the range of Manitoba\u2019s only two radars. Forecasters had satellite imagery and little else; the city of 5,000 was evacuated for weeks.',
            source: 'CBC News',
            url: 'https://www.cbc.ca/news/canada/manitoba/weather-radar-emergency-response-north-9.7180645'
        },
        {
            id: 'rossburn', lon: -100.80, lat: 50.69, px: 0, py: 0,
            name: 'Rossburn, Manitoba — 2026',
            desc: 'A tornado touched down with no advance alert. Experts warn that federal cuts to the weather-radar research group will further blunt forecasters\u2019 ability to see tornadoes coming.',
            source: 'CBC News',
            url: 'https://www.cbc.ca/news/canada/manitoba/radar-research-cuts-tornado-prediction-9.7261730'
        },
        {
            id: 'peterborough', lon: -78.32, lat: 44.31, px: 0, py: 0,
            name: 'Peterborough, Ontario — 2022',
            desc: 'The derecho\u2019s emergency alert went out to Toronto but never to Peterborough, where the storm tore roofs off cottages. Eleven people died in Ontario; 900,000 lost power.',
            source: 'CBC News',
            url: 'https://www.cbc.ca/news/canada/canada-storm-alert-failed-peterborough-1.6467263'
        },
        {
            id: 'barrie', lon: -79.69, lat: 44.39, px: 0, py: 0,
            name: 'Barrie, Ontario — 2021',
            desc: 'An EF2 tornado cut a 12.5 km scar through the city, injuring 11 and causing $107M in damage \u2014 many residents got the warning only after it hit. Researchers found 70% of Canadian tornadoes from 2019\u201321 came with no warning at all.',
            source: 'Canada\u2019s National Observer',
            url: 'https://www.nationalobserver.com/2022/10/27/news/devastating-tornadoes-hit-canada-without-warning'
        },
        {
            id: 'princerupert', lon: -130.32, lat: 54.32, px: 0, py: 0,
            name: 'Prince Rupert, B.C. — 2020',
            desc: 'Three coastal weather stations sat broken \u2014 two of them for nearly a year \u2014 at one of North America\u2019s busiest ports. \u201cWe are basically flying blind,\u201d said a mariner working those waters.',
            source: 'CBC News',
            url: 'https://www.cbc.ca/news/canada/british-columbia/weather-stations-need-repairing-prince-rupert-1.5455931'
        },
        {
            id: 'nain', lon: -61.69, lat: 56.54, px: 0, py: 0,
            name: 'Nain, Labrador — 2023',
            desc: 'Hundreds of kilometres separate weather stations across northern Canada. The gaps affect hunting, medical flights and safe travel on the land \u2014 researchers say they can be a matter of life or death.',
            source: 'CBC Radio, What On Earth',
            url: 'https://www.cbc.ca/radio/whatonearth/weather-data-gaps-life-or-death-1.6751883'
        },
        {
            id: 'cambridgebay', lon: -105.14, lat: 69.11, px: 0, py: 0,
            name: 'Cambridge Bay, Nunavut — 2023',
            desc: 'The number of far-northern stations reporting multiple climate variables has collapsed since the 1990s. The Arctic is warming faster than anywhere on Earth \u2014 and is being measured less.',
            source: 'Science',
            url: 'https://www.science.org/content/article/dwindling-weather-data-leave-canadians-cold'
        }
    ]);
    let hoveredStarId = $state(null);
    let hoveredStar = $derived(monitoringFailures.find(s => s.id === hoveredStarId));

    // Benefits typewriter sequence
    let showArrow = $state(false);
    let sequenceStarted = $state(false);
    let benefitStage = $state(0);
    let introTyped = $state({ text: '' });

    let benefits = $state([
        { full: 'improve early warning systems for storms', typed: { text: '' }, caption: 'Rain cells on urban radar; the Northern Mesonet Project', media: '/radar-cells.gif' },
        { full: 'inform smarter city planning', typed: { text: '' }, caption: 'Cities adapting rural flood-mitigation techniques', media: '/city-planning.jpg' },
        { full: 'learn more about the micro-scale changes of climate change', typed: { text: '' }, caption: 'The Canadian Severe Storms Lab, where the Northern Hail and Tornado Project are in', media: '/hail-project.jpg' },
        { full: 'advance weather forecasting knowledge and techniques', typed: { text: '' }, caption: 'AI forecasting: GraphCast, NVIDIA Earth-2, GenCast, FourCastNet', media: '/ai-forecasting.jpg' }
    ]);

    function typewrite(full, target, speed = 25) {
        let i = 0;
        target.text = '';
        const id = setInterval(() => {
            target.text = full.slice(0, i + 1);
            i++;
            if (i >= full.length) clearInterval(id);
        }, speed);
    }

    function startSequence() {
        sequenceStarted = true;
        typewrite('While many of these events were rural, this is an urban issue too. Improving monitoring and data collection across Canada lets us:', introTyped, 20);
        benefitStage = 0;
    }

    function advanceSequence() {
        if (benefitStage < benefits.length) {
            const b = benefits[benefitStage];
            typewrite(b.full, b.typed, 25);
            benefitStage += 1;
        }
    }

    // Falling event droplets (throttled)
    $effect(() => {
        const intervalId = setInterval(() => {
            for (let i = 0; i < eventDroplets.length; i++) {
                if (eventDroplets[i].id !== hoveredEventId) {
                    eventDroplets[i].y += eventDroplets[i].vy;
                    if (eventDroplets[i].y > window.innerHeight + 100) {
                        eventDroplets[i].y = -100;
                    }
                }
            }
        }, 80);

        return () => clearInterval(intervalId);
    });

    // Show the advance arrow 5s after the map appears
    $effect(() => {
        if (gameComplete) {
            const t = setTimeout(() => { showArrow = true; }, 5000);
            return () => clearTimeout(t);
        }
    });

    // A round with flooded == null has no right answer — the record isn't there, so
    // no guess can be correct. It still counts toward the total, which caps the best
    // possible score at 3 of 4. That ceiling is the point.
    function submitGuess(guess) {
        userGuess = guess;
        const truth = rounds[currentRound].flooded;
        if (truth == null) return;              // unanswerable: nobody scores this one
        if ((guess === 'Y') === truth) {
            correctCount += 1;
        }
    }

    function nextRound() {
        userGuess = null;
        if (currentRound + 1 >= totalRounds) {
            gameVerdict = true;
        } else {
            currentRound += 1;
        }
    }

    // ---- Main scene controller (back / forward) ----
    // Only active once the scroll-driven intro is over (map onward), so the
    // scrollama flow in the rain / game / verdict phases is never overridden.
    let showController = $derived(gameComplete || gameVerdict || scene !== 0);
    let canGoForward = $derived(scene !== 14);

    function goForward() {
        if (scene === 13) { scene = 14; return; }
        if (scene === 12) { scene = 13; return; }
        if (scene === 11) { scene = 12; return; }
        if (scene !== 0) return;
        if (sequenceStarted) {
            if (benefitStage < benefits.length) advanceSequence();
            else scene = 11;
            return;
        }
        if (gameComplete) { startSequence(); return; }
    }

    // Fully unwind the game/benefits flow back to the falling-rain intro.
    // The chosen droplet is frozen huge once the game runs, so it must be
    // released (velocity restored, un-chosen) or the intro renders as one
    // giant static drop. We also scroll to the top so scrollama re-syncs its
    // step indices instead of leaving the flow in a half-scrolled state.
    function resetToRain() {
        if (chosenDrop) {
            chosenDrop.a = 4;
            chosenDrop.vy = Math.random() * 2 + 5;
            chosenDrop.x = Math.random() * (canvas ? canvas.width : window.innerWidth);
            chosenDrop.y = -40;
            chosenDrop = null;
        }
        scene = 0;
        currentStep = 0;
        gameVisible = false;
        gameStarted = false;
        gameComplete = false;
        gameVerdict = false;
        currentRound = 0;
        correctCount = 0;
        userGuess = null;
        showArrow = false;
        sequenceStarted = false;
        benefitStage = 0;
        if (typeof window !== 'undefined') window.scrollTo({ top: 0, behavior: 'smooth' });
    }

    function goBack() {
        if (scene === 14) { scene = 13; return; }
        if (scene === 13) { scene = 12; return; }
        if (scene === 12) { scene = 11; return; }
        if (scene === 11) {
            // back into the fully-revealed benefits sequence
            scene = 0;
            gameComplete = true;
            sequenceStarted = true;
            benefitStage = benefits.length;
            return;
        }
        if (scene !== 0) return;
        if (sequenceStarted) {
            if (benefitStage > 0) benefitStage -= 1;
            else sequenceStarted = false;
            return;
        }
        if (gameComplete) { gameComplete = false; gameVerdict = true; return; }
        if (gameVerdict) { resetToRain(); return; }
    }

    let chosenDrop = null;

    onMount(() => {
        async function loadCanada() {
            const geo = await fetch('/canada.geojson').then(r => r.json());

            const projection = d3.geoConicConformal()
                .rotate([98, 0])
                .fitSize([1000, 700], geo);

            const pathGen = d3.geoPath(projection);
            canadaPath = pathGen(geo);

            for (const star of monitoringFailures) {
                const [px, py] = projection([star.lon, star.lat]);
                star.px = px;
                star.py = py;
            }

            mapReady = true;
        }
        loadCanada();

        fetch('/data/game_rounds.json')
            .then(r => r.json())
            .then(d => rounds = d)
            .catch(err => console.error('Failed to load game rounds:', err));

        if (canvas) {
            setTimeout(() => {
                const scroller = scrollama();
                scroller.setup({
                    step: '.step',
                    offset: 0.5,
                    progress: true
                });
                scroller.onStepEnter(handleStepEnter);
                scroller.onStepProgress(handleStepProgress);
            }, 100);

            let a = 4;
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
            const ctx = canvas.getContext('2d');
            const num_drops = 900;
            const droplets = Array.from({ length: num_drops }, () => ({
                x: Math.random() * canvas.width,
                y: Math.random() * canvas.height,
                vy: Math.random() * 2 + 5,
                a: a,
            }));

            function handleStepEnter(response) {
                currentStep = response.index;
                if (response.index === 2) {
                    var mindist = 9000;
                    for (const drop of droplets) {
                        if (Math.hypot(drop.x - (canvas.width / 2), drop.y - (canvas.height / 2)) < mindist) {
                            mindist = Math.hypot(drop.x - (canvas.width / 2), drop.y - (canvas.height / 2));
                            chosenDrop = drop;
                        }
                    }
                    chosenDrop.vy = 0;
                    chosenDrop.x = canvas.width / 2;
                    chosenDrop.y = canvas.height / 2;
                }

                if (response.index === 1 && response.direction === 'up') {
                    if (chosenDrop) {
                        chosenDrop.a = 4;
                        chosenDrop.vy = Math.random() * 2 + 5;
                        chosenDrop = null;
                        gameStarted = false;
                        gameComplete = false;
                        gameVerdict = false;
                        currentRound = 0;
                        correctCount = 0;
                        userGuess = null;
                        showArrow = false;
                        sequenceStarted = false;
                        benefitStage = 0;
                    }
                }
            }

            // Threshold at which the expanding droplet has swallowed enough of the
            // screen to hand over to the game.
            const GAME_THRESHOLD = 0.6;

            function handleStepProgress(response) {
                if (!chosenDrop) return;
                if (response.index < 2) {
                    gameVisible = false;
                    gameStarted = false;
                    return;
                }
                const globalProgress = (response.index - 2 + response.progress) / 2;

                let maxSize = Math.max(canvas.width, canvas.height);
                chosenDrop.a = 4 + (globalProgress * maxSize);

                // Once the game has been played through, its own overlays (verdict,
                // benefits) own the screen — scroll progress must not yank them away.
                if (gameVerdict || gameComplete) return;

                const past = globalProgress > GAME_THRESHOLD;

                // Scrolling back out below the threshold has to *un*-latch this.
                // Leaving it stuck true is what left the game floating over the rain.
                if (past !== gameStarted) {
                    gameStarted = past;
                    if (!past) {
                        // Abandon the run in progress so returning to it starts clean
                        // rather than resuming mid-round with a stale guess showing.
                        currentRound = 0;
                        correctCount = 0;
                        userGuess = null;
                        showArrow = false;
                    }
                }
                gameVisible = gameStarted;
            }

            function frame() {
                ctx.clearRect(0, 0, canvas.width, canvas.height);

                for (const drop of droplets) {
                    if (drop === chosenDrop) continue;
                    if (chosenDrop && Math.hypot(drop.x - chosenDrop.x, drop.y - chosenDrop.y) < chosenDrop.a) continue;
                    ctx.fillStyle = "blue";
                    ctx.strokeStyle = "black";
                    ctx.beginPath();
                    ctx.moveTo(drop.x, drop.y - (drop.a * 2));
                    ctx.lineTo(drop.x + drop.a, drop.y);
                    ctx.arc(drop.x, drop.y, drop.a, 0, Math.PI);
                    ctx.closePath();
                    ctx.fill();
                    ctx.stroke();
                }

                if (chosenDrop) {
                    ctx.fillStyle = "blue";
                    ctx.strokeStyle = "black";
                    ctx.beginPath();
                    ctx.moveTo(chosenDrop.x, chosenDrop.y - (chosenDrop.a * 2));
                    ctx.lineTo(chosenDrop.x + chosenDrop.a, chosenDrop.y);
                    ctx.arc(chosenDrop.x, chosenDrop.y, chosenDrop.a, 0, Math.PI);
                    ctx.closePath();
                    ctx.fill();
                    ctx.stroke();
                }

                for (const drop of droplets) {
                    if (drop === chosenDrop) continue;
                    if (chosenDrop && Math.hypot(drop.x - chosenDrop.x, drop.y - chosenDrop.y) < chosenDrop.a) continue;
                    drop.y += drop.vy;
                    if (drop.y - (drop.a * 2) > canvas.height) {
                        drop.y = -(drop.a * 2);
                        drop.x = Math.random() * canvas.width;
                    }
                }

                animationFrameId = window.requestAnimationFrame(frame);
            }

            frame();
        }
    });
</script>

{#if scene === 0 && currentStep < 2}
    <div class="sky-bg" transition:fade={{ duration: 500 }}></div>
{/if}
<canvas bind:this={canvas}></canvas>

{#if scene === 0 && currentStep < 2}
    <div class="lightning" transition:fade={{ duration: 500 }}></div>
{/if}

{#if currentStep === 0 && scene === 0 && !gameComplete}
    <div class="intro-title" transition:fade={{ duration: 600 }}>
        <h1>The Monitoring Mirage</h1>
        <div class="scroll-cue">
            <span>Scroll to begin</span>
            <span class="chevron">⌄</span>
        </div>
    </div>
{/if}

<div bind:this={introView} id="intro-view" class="intro-view">
    <div class="sticky-graphic">
        <svg class="interactive-layer">
            {#each eventDroplets as drop}
                <path
                    d="M {drop.x} {drop.y - drop.a} 
                    L {drop.x + drop.a} {drop.y} 
                    A {drop.a} {drop.a} 0 1 1 {drop.x - drop.a} {drop.y} 
                    Z"
                    fill={hoveredEventId === drop.id ? "red" : "gold"}
                    stroke="black"
                    class="event-droplet"
                    role="button"
                    tabindex="0"
                    onmouseenter={() => hoveredEventId = drop.id}
                    onmouseleave={() => hoveredEventId = null}
                />
                <text
                    x="{drop.x}"
                    y={drop.y + 4}
                    text-anchor="middle"
                    font-size="10"
                    pointer-events="none"
                    fill="white"
                >
                    {drop.year}
                </text>
            {/each}
        </svg>

        {#if hoveredEventData}
            <div
                class="tooltip"
                style="left: {hoveredEventData.x}px; top: {hoveredEventData.y}px;"
            >
                <h3>{hoveredEventData.name} ({hoveredEventData.year})</h3>
                <p>{hoveredEventData.damage}</p>
                <p class="tooltip-hint">Bigger drop = costlier flood</p>
            </div>
        {/if}

        {#if gameVisible && !gameVerdict && !gameComplete && rounds[currentRound]}
            <div class="game-overlay" transition:fade={{ duration: 400 }}>
                <div class="game-card">
                    <h2>Guess the Flood</h2>
                    <p>Round {currentRound + 1} of {totalRounds} — did a flood occur?</p>

                    <div class="graphs-container">
                        <div class="graph">
                            <div class="graph-head">
                                <h4>Precipitation</h4>
                                <span class="axis-unit">mm / day</span>
                            </div>
                            <div class="chart-row">
                                <div class="y-ticks">
                                    <span>{fmtTick(precipMax)}</span>
                                    <span>{fmtTick(precipMax / 2)}</span>
                                    <span>0</span>
                                </div>
                                <div class="plot">
                                    <div class="grid-line" style="top: 0"></div>
                                    <div class="grid-line" style="top: 50%"></div>
                                    <div class="mock-bars">
                                        {#each precipHeights(rounds[currentRound].precip) as b}
                                            {#if b.missing}
                                                <div class="bar missing" title="No data recorded"></div>
                                            {:else}
                                                <div class="bar" style="height: {b.h}%"></div>
                                            {/if}
                                        {/each}
                                    </div>
                                </div>
                            </div>
                            <div class="x-axis-row"><span>Day 1</span><span>Day {roundDays}</span></div>
                        </div>

                        <div class="graph">
                            <div class="graph-head">
                                <h4>
                                    Streamflow
                                    <span class="info" tabindex="0" role="img" aria-label="What is streamflow?">
                                        &#9432;
                                        <span class="info-pop">
                                            <strong>Streamflow</strong> — the volume of water flowing
                                            through a river, channel, or conduit per unit of time
                                            (here, per day).
                                        </span>
                                    </span>
                                </h4>
                                <span class="axis-unit">m³ / s</span>
                            </div>
                            <div class="chart-row">
                                <div class="y-ticks">
                                    <span>{fmtTick(flowMax)}</span>
                                    <span>{fmtTick((flowMax + flowMin) / 2)}</span>
                                    <span>{fmtTick(flowMin)}</span>
                                </div>
                                <div class="plot">
                                    <div class="grid-line" style="top: 0"></div>
                                    <div class="grid-line" style="top: 50%"></div>
                                    <svg class="flow-svg" width="100%" height="100%" viewBox="0 0 100 50" preserveAspectRatio="none">
                                        {#each flowSegments(rounds[currentRound].flow).paths as d}
                                            <path {d} fill="none" stroke="#60a5fa" stroke-width="2.5" vector-effect="non-scaling-stroke"/>
                                        {/each}
                                        {#each flowSegments(rounds[currentRound].flow).points as p}
                                            <circle cx={p.x} cy={p.y} r="1.6" fill="#60a5fa" vector-effect="non-scaling-stroke"/>
                                        {/each}
                                    </svg>
                                </div>
                            </div>
                            <div class="x-axis-row"><span>Day 1</span><span>Day {roundDays}</span></div>
                        </div>
                    </div>

                    {#if missingDays > 0}
                        <p class="gap-note">
                            <span class="gap-swatch"></span>
                            {missingDays} reading{missingDays === 1 ? '' : 's'} missing — the gauge reported nothing on those days.
                        </p>
                    {/if}

                    {#if userGuess === null}
                        <div class="button-group">
                            <button class="btn-yes" onclick={() => submitGuess('Y')}>Flood: Yes</button>
                            <button class="btn-no" onclick={() => submitGuess('N')}>Flood: No</button>
                        </div>
                    {:else}
                        <div class="result-view" transition:fade>
                            {#if rounds[currentRound].flooded === null || rounds[currentRound].flooded === undefined}
                                <h3 class="unknowable">{rounds[currentRound].name ?? 'We may never know.'}</h3>
                                <p>{rounds[currentRound].blurb ?? 'There is no meteorological record for these days, so there is no way to tell whether a flood happened.'}</p>
                            {:else if rounds[currentRound].flooded}
                                <h3>This was a flood.</h3>
                                <p>It was <strong>{rounds[currentRound].event}</strong>.{rounds[currentRound].blurb ? ` ${rounds[currentRound].blurb}` : ''}</p>
                            {:else}
                                <h3>No flood here.</h3>
                                <p>{rounds[currentRound].blurb ?? 'The rivers held, despite the rain.'}</p>
                            {/if}
                            <button class="btn-reset" onclick={nextRound}>
                                {currentRound + 1 >= totalRounds ? 'See how you did' : 'Next day'}
                            </button>
                        </div>
                    {/if}
                </div>
            </div>
        {/if}

        {#if gameVerdict && !gameComplete}
            <div class="game-overlay" transition:fade={{ duration: 400 }}>
                <div class="game-card">
                    <h2>{correctCount} of {totalRounds}, nice job given the data</h2>
                    <p class="unscored">This is not an isolated problem. There have been many other instances.</p>
                    <button class="btn-reset" onclick={() => gameComplete = true}>See the bigger picture</button>
                </div>
            </div>
        {/if}

        {#if gameComplete}
            <div class="black-bg" transition:fade={{ duration: 800 }}></div>
            <div class="map-overlay" transition:fade={{ duration: 800 }}>
                <div class="map-layout">
                    <div class="map-column">
                        <div class="map-inner">
                            <svg viewBox="0 0 1000 700" class="canada-svg" preserveAspectRatio="xMidYMid meet">
                                {#if mapReady}
                                    <path d={canadaPath} fill="rgba(80, 120, 200, 0.25)" stroke="#8ab4ff" stroke-width="1.5" />
                                    {#each monitoringFailures as star}
                                        <text
                                            class="star-svg"
                                            x={star.px}
                                            y={star.py}
                                            text-anchor="middle"
                                            dominant-baseline="central"
                                            role="button"
                                            tabindex="0"
                                            aria-label={`${star.name} — read the reporting`}
                                            onmouseenter={() => hoveredStarId = star.id}
                                            onmouseleave={() => hoveredStarId = null}
                                            onfocus={() => hoveredStarId = star.id}
                                            onblur={() => hoveredStarId = null}
                                            onclick={() => window.open(star.url, '_blank', 'noopener')}
                                            onkeydown={(e) => { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); window.open(star.url, '_blank', 'noopener'); } }}
                                        >★</text>
                                    {/each}
                                {/if}
                            </svg>

                            {#if hoveredStar}
                                <div class="star-tooltip" style="left: {(hoveredStar.px / 1000) * 100}%; top: {(hoveredStar.py / 700) * 100}%;" transition:fade={{ duration: 200 }}>
                                    <strong>{hoveredStar.name}</strong>
                                    <span>{hoveredStar.desc}</span>
                                    <span class="star-source">{hoveredStar.source} · click to read →</span>
                                </div>
                            {/if}
                        </div>
                    </div>

                    <div class="text-column">
                        <h2 class="map-title">Where a lack of monitoring caused harm</h2>
                        <p>Across Canada, gaps in weather and streamflow monitoring have left communities without warning when it mattered most.</p>
                        <p>Each marked location is a real event where a radar gap, a broken station, or a missing gauge delayed detection — or meant no warning came at all.</p>
                        <p>Hover a star to see what happened. Click it to read the reporting.</p>
                    </div>
                </div>
            </div>

            {#if showArrow && !sequenceStarted}
                <button class="advance-arrow" onclick={startSequence} transition:fade aria-label="Continue">→</button>
            {/if}

            {#if sequenceStarted}
                <div class="benefits-overlay" transition:fade={{ duration: 600 }}>
                    <div class="benefits-content">
                        <p class="benefits-intro">{introTyped.text}</p>

                        {#each benefits as benefit, i}
                            {#if benefitStage > i}
                                <div class="benefit-row">
                                    <div class="benefit-text">
                                        <span class="benefit-num">{i + 1}.</span>
                                        <span>{benefit.typed.text}</span>
                                    </div>
                                    {#if benefit.typed.text === benefit.full}
                                        <div class="benefit-media" transition:fade={{ duration: 400 }}>
                                            <img src={benefit.media} alt={benefit.caption} />
                                            <span class="benefit-caption">{benefit.caption}</span>
                                        </div>
                                    {/if}
                                </div>
                            {/if}
                        {/each}

                        <div class="benefits-footer">
                            {#if benefitStage < benefits.length}
                                <button class="advance-arrow inline" onclick={advanceSequence} aria-label="Next">→</button>
                            {:else if scene === 0}
                                <button class="advance-arrow inline pulse" onclick={() => scene = 11} aria-label="Continue">→</button>
                            {/if}
                        </div>
                    </div>
                </div>
            {/if}
        {/if}
    </div>

    <div class="steps">
        <div class="step" data-step="1"></div>
        <div class="step" data-step="2"></div>
        <div class="step" data-step="3"></div>
        <div class="step" data-step="4"></div>
    </div>
</div>
{#if scene === 11}
    <div class="scene-fullscreen">
        <OntarioStations onContinue={() => scene = 12} />
    </div>
{/if}

{#if scene === 12}
    <div class="scene-fullscreen interstitial">
        <p class="interstitial-line" transition:fade={{ duration: 1000 }}>However, if we dig deeper…</p>
        <button class="advance-arrow" onclick={() => scene = 13} aria-label="Continue">→</button>
    </div>
{/if}

{#if scene === 13}
    <div class="scene-fullscreen">
        <WScoreMap onContinue={() => scene = 14} />
    </div>
{/if}

{#if scene === 14}
    <div class="scene-fullscreen">
        <DisasterReports />
    </div>
{/if}

{#if showController}
    <div class="scene-controller" transition:fade={{ duration: 300 }}>
        <button class="ctrl-btn" onclick={goBack} aria-label="Previous scene">←</button>
        <button class="ctrl-btn" onclick={goForward} disabled={!canGoForward} aria-label="Next scene">→</button>
    </div>
{/if}

<style>
    :global(html), :global(body) {
        margin: 0;
        background-color: #1a1a2e;
        color: white;
        font-family: sans-serif;
    }

    .intro-view {
        position: relative;
        width: 100%;
    }

    canvas {
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }

    /* sky-bg / lightning intentionally have NO z-index: as fixed elements they
       sit at stacking level 0 and rely on DOM order (both come before the
       .intro-view subtree), so the game/map overlays still paint above them —
       exactly like the canvas did originally. */
    .sky-bg {
        position: fixed;
        inset: 0;
        background: linear-gradient(to bottom, #5b9bd5 0%, #8fc0e8 55%, #b8dcf2 100%);
    }

    .lightning {
        position: fixed;
        inset: 0;
        pointer-events: none;
        opacity: 0;
        background: radial-gradient(ellipse at 50% -10%, rgba(255,255,255,0.95), rgba(255,255,255,0) 65%);
        animation: lightning 7s infinite;
    }

    @keyframes lightning {
        0%, 46% { opacity: 0; }
        47% { opacity: 0.85; }
        48% { opacity: 0.15; }
        49% { opacity: 0.9; }
        51% { opacity: 0; }
        81% { opacity: 0; }
        82% { opacity: 0.6; }
        84% { opacity: 0; }
        100% { opacity: 0; }
    }

    .intro-title {
        position: fixed;
        inset: 0;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        text-align: center;
        pointer-events: none;
        z-index: 20;
    }

    .intro-title h1 {
        margin: 0;
        font-size: clamp(2.5rem, 7vw, 5rem);
        color: #072840;
        letter-spacing: 0.02em;
        text-shadow: 0 2px 18px rgba(255,255,255,0.5);
    }

    .scroll-cue {
        margin-top: 2rem;
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 0.2rem;
        color: #072840;
        font-size: 1rem;
        opacity: 0.85;
    }

    .scroll-cue .chevron {
        font-size: 2.2rem;
        line-height: 1;
        animation: bounce 1.6s infinite;
    }

    @keyframes bounce {
        0%, 100% { transform: translateY(0); }
        50% { transform: translateY(8px); }
    }

    .scene-controller {
        position: fixed;
        bottom: 1.5rem;
        left: 50%;
        transform: translateX(-50%);
        display: flex;
        gap: 1rem;
        z-index: 120;
    }

    .ctrl-btn {
        width: 3rem;
        height: 3rem;
        padding: 0;
        border-radius: 50%;
        background: rgba(10, 10, 18, 0.75);
        color: #8ab4ff;
        border: 1px solid rgba(138, 180, 255, 0.5);
        font-size: 1.4rem;
        font-weight: bold;
        display: flex;
        align-items: center;
        justify-content: center;
        backdrop-filter: blur(4px);
    }

    .ctrl-btn:disabled {
        opacity: 0.3;
        cursor: default;
        transform: none;
    }

    .interactive-layer {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        pointer-events: none;
        z-index: 10;
    }

    .event-droplet {
        pointer-events: auto;
        cursor: pointer;
    }

    .tooltip {
        position: absolute;
        pointer-events: none;
        background: white;
        color: black;
        padding: 10px;
        border: 1px solid black;
        border-radius: 4px;
        transform: translate(-50%, -100%);
        z-index: 20;
    }

    .step {
        position: relative;
        height: 100vh;
        pointer-events: none;
    }

    .sticky-graphic {
        position: sticky;
        top: 0;
        height: 100vh;
    }

    .game-overlay {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        display: flex;
        justify-content: center;
        align-items: center;
        z-index: 30;
        pointer-events: auto;
    }

    .game-card {
        background: #0a0a12;
        color: #f8fafc;
        padding: 2rem;
        border-radius: 12px;
        text-align: center;
        box-shadow: 0 10px 40px rgba(0,0,0,0.7);
        border: 1px solid rgba(138, 180, 255, 0.25);
        max-width: 500px;
        width: 90%;
    }

    .graphs-container {
        display: flex;
        flex-direction: column;
        gap: 1.5rem;
        margin: 1.5rem 0;
    }

    .graph-head {
        display: flex;
        align-items: baseline;
        justify-content: space-between;
        margin-bottom: 0.4rem;
    }

    .graph-head h4 {
        margin: 0;
        text-align: left;
        font-size: 0.9rem;
        color: #e2e8f0;
        display: flex;
        align-items: center;
        gap: 0.35rem;
    }

    .axis-unit {
        font-size: 0.72rem;
        color: #8ab4ff;
        letter-spacing: 0.02em;
    }

    /* Streamflow definition on hover / focus */
    .info {
        position: relative;
        display: inline-flex;
        width: 1rem;
        height: 1rem;
        align-items: center;
        justify-content: center;
        font-size: 0.8rem;
        color: #8ab4ff;
        border: 1px solid rgba(138,180,255,0.5);
        border-radius: 50%;
        cursor: help;
        line-height: 1;
    }

    .info-pop {
        position: absolute;
        bottom: calc(100% + 8px);
        left: 50%;
        transform: translateX(-50%);
        width: 230px;
        background: #f8fafc;
        color: #1a1a2e;
        padding: 0.6rem 0.7rem;
        border-radius: 6px;
        font-size: 0.78rem;
        font-weight: 400;
        line-height: 1.4;
        text-align: left;
        opacity: 0;
        visibility: hidden;
        transition: opacity 0.15s;
        box-shadow: 0 6px 18px rgba(0,0,0,0.5);
        z-index: 60;
    }

    .info-pop strong { color: #1a1a2e; }
    .info:hover .info-pop,
    .info:focus .info-pop { opacity: 1; visibility: visible; }

    /* chart layout: y-tick column + plot area, x labels underneath */
    .chart-row {
        display: flex;
        gap: 0.4rem;
        align-items: stretch;
    }

    .y-ticks {
        display: flex;
        flex-direction: column;
        justify-content: space-between;
        align-items: flex-end;
        height: 90px;
        min-width: 2.2rem;
        font-size: 0.68rem;
        color: #94a3b8;
        font-variant-numeric: tabular-nums;
    }

    .plot {
        position: relative;
        flex: 1;
        height: 90px;
        background: rgba(138,180,255,0.05);
        border-bottom: 1.5px solid rgba(138,180,255,0.5);
        border-left: 1.5px solid rgba(138,180,255,0.5);
    }

    .grid-line {
        position: absolute;
        left: 0;
        right: 0;
        height: 1px;
        background: rgba(255,255,255,0.06);
        pointer-events: none;
    }

    .mock-bars {
        display: flex;
        align-items: flex-end;
        justify-content: space-around;
        gap: 3px;
        height: 100%;
        padding: 0 4px;
        box-sizing: border-box;
    }

    .mock-bars .bar {
        flex: 1;
        min-width: 2px;
        background-color: #10b981;
        border-radius: 2px 2px 0 0;
    }

    /* A day with no reading. Deliberately NOT a zero-height bar: an absent
       measurement and a measured zero are different facts. */
    .mock-bars .bar.missing {
        align-self: stretch;
        background-color: transparent;
        background-image: repeating-linear-gradient(
            45deg,
            rgba(148,163,184,0.35) 0 2px,
            transparent 2px 5px
        );
        border-radius: 0;
    }

    .gap-note {
        display: flex;
        align-items: center;
        gap: 0.4rem;
        margin: 0 0 0.8rem;
        font-size: 0.74rem;
        color: #94a3b8;
    }
    .gap-swatch {
        width: 14px; height: 12px; flex: none;
        background-image: repeating-linear-gradient(
            45deg,
            rgba(148,163,184,0.5) 0 2px,
            transparent 2px 5px
        );
    }

    .unscored {
        font-size: 0.82rem !important;
        color: #fbbf24 !important;
        line-height: 1.5;
    }

    .flow-svg {
        position: absolute;
        inset: 0;
        display: block;
    }

    .x-axis-row {
        display: flex;
        justify-content: space-between;
        margin-left: 2.6rem;
        margin-top: 0.3rem;
        font-size: 0.68rem;
        color: #94a3b8;
    }

    .tooltip-hint {
        margin: 0.35rem 0 0;
        font-size: 0.72rem;
        color: #64748b;
        font-style: italic;
    }

    .button-group {
        display: flex;
        justify-content: center;
        gap: 1rem;
        margin-top: 1rem;
    }

    button {
        padding: 0.75rem 1.5rem;
        border: none;
        border-radius: 6px;
        font-weight: bold;
        font-size: 1rem;
        cursor: pointer;
        transition: transform 0.1s, opacity 0.2s;
    }

    button:hover {
        opacity: 0.9;
        transform: translateY(-2px);
    }

    .btn-yes { background-color: #ef4444; color: white; }
    .btn-no { background-color: #10b981; color: white; }
    .btn-reset { background-color: #6366f1; color: white; margin-top: 1rem; }

    .result-view {
        margin-top: 1.5rem;
        padding: 1rem;
        background: rgba(255,255,255,0.04);
        border-radius: 8px;
        border: 1px dashed rgba(138,180,255,0.35);
    }

    .result-view strong { color: #8ab4ff; }

    .black-bg {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: #0a0a12;
        z-index: 25;
    }

    .map-overlay {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        display: flex;
        justify-content: center;
        align-items: center;
        z-index: 30;
        pointer-events: auto;
    }

    .map-layout {
        display: flex;
        align-items: center;
        gap: 2rem;
        width: 90%;
        max-width: 1400px;
    }

    .map-column {
        flex: 2;
    }

    .text-column {
        flex: 1;
        text-align: left;
        color: white;
    }

    .map-title {
        color: white;
        margin: 0 0 1rem 0;
        font-size: 1.6rem;
    }

    .text-column p {
        line-height: 1.6;
        color: #cbd5e1;
        margin-bottom: 1rem;
    }

    .map-inner {
        position: relative;
        width: 100%;
        aspect-ratio: 1000 / 700;
    }

    .canada-svg {
        width: 100%;
        height: 100%;
        display: block;
    }

    .star-svg {
        fill: gold;
        font-size: 28px;
        cursor: pointer;
        filter: drop-shadow(0 0 6px rgba(255, 215, 0, 0.8));
        transition: font-size 0.15s;
    }

    .star-svg:hover {
        font-size: 38px;
    }

    .star-source {
        margin-top: 0.4rem;
        font-size: 0.72rem;
        color: #8ab4ff;
        font-weight: 600;
    }

    .star-tooltip {
        position: absolute;
        transform: translate(-50%, calc(-100% - 1.5rem));
        background: white;
        color: #1a1a2e;
        padding: 0.6rem 0.8rem;
        border-radius: 6px;
        width: 200px;
        font-size: 0.85rem;
        display: flex;
        flex-direction: column;
        gap: 0.25rem;
        pointer-events: none;
        box-shadow: 0 6px 18px rgba(0,0,0,0.4);
        z-index: 40;
    }

    .star-tooltip strong { font-size: 0.95rem; }

    .advance-arrow {
        position: absolute;
        bottom: 2rem;
        right: 2rem;
        width: 3.5rem;
        height: 3.5rem;
        border-radius: 50%;
        background: rgba(138, 180, 255, 0.9);
        color: #0a0a12;
        font-size: 1.6rem;
        font-weight: bold;
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 50;
        animation: pulse 2s infinite;
    }

    .advance-arrow.inline {
        position: static;
        margin: 0;
        width: 2.6rem;
        height: 2.6rem;
        font-size: 1.2rem;
        animation: none;
    }

    .advance-arrow.inline.pulse { animation: pulse 2s infinite; }

    @keyframes pulse {
        0%, 100% { transform: scale(1); box-shadow: 0 0 0 0 rgba(138,180,255,0.5); }
        50% { transform: scale(1.08); box-shadow: 0 0 0 12px rgba(138,180,255,0); }
    }

    /* One screen, never scrolls: the content column is a flex box that divides
       the available height between however many benefit rows are revealed, and
       each row's image shrinks to whatever height is left (min-height:0 is what
       lets flex children actually shrink below their content size). */
    .benefits-overlay {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: #0a0a12;
        z-index: 45;
        overflow: hidden;
        display: flex;
        justify-content: center;
        padding: 2rem 2rem 5.5rem;   /* bottom clearance for the scene controller */
        box-sizing: border-box;
    }

    .benefits-content {
        max-width: 860px;
        width: 100%;
        height: 100%;
        min-height: 0;
        color: white;
        display: flex;
        flex-direction: column;
        gap: 0.75rem;
    }

    .benefits-intro {
        flex: none;
        font-size: clamp(0.95rem, 1.5vh, 1.25rem);
        line-height: 1.5;
        margin: 0;
    }

    .benefit-row {
        flex: 1 1 0;
        min-height: 0;              /* allow the row to shrink */
        display: flex;
        gap: 1.25rem;
        align-items: center;
        margin: 0;
        padding-bottom: 0.75rem;
        border-bottom: 1px solid rgba(255,255,255,0.1);
    }

    .benefit-text {
        flex: 1;
        font-size: clamp(0.85rem, 1.4vh, 1.1rem);
        line-height: 1.45;
    }

    .benefit-num {
        color: #8ab4ff;
        font-weight: bold;
        margin-right: 0.5rem;
    }

    .benefit-media {
        flex: 1;
        min-width: 0;
        min-height: 0;
        height: 100%;
        display: flex;
        flex-direction: column;
        gap: 0.35rem;
        justify-content: center;
    }

    .benefit-media img {
        width: 100%;
        flex: 1 1 auto;
        min-height: 0;              /* shrink instead of overflowing the page */
        border-radius: 8px;
        background: #1a1a2e;
        object-fit: cover;
    }

    .benefit-caption {
        flex: none;
        font-size: clamp(0.65rem, 1.05vh, 0.8rem);
        line-height: 1.3;
        color: #94a3b8;
    }

    .benefits-footer {
        flex: none;
        display: flex;
        justify-content: center;
        min-height: 2.75rem;
        align-items: center;
    }

    .scene-fullscreen {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    z-index: 100;
    background: #0a0a12;
}

    .interstitial {
        display: flex;
        align-items: center;
        justify-content: center;
    }

    .interstitial-line {
        color: #f8fafc;
        font-size: clamp(1.8rem, 4vw, 3rem);
        font-weight: 600;
        text-align: center;
        max-width: 60%;
        line-height: 1.3;
    }
</style>