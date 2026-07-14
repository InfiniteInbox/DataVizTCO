<script>
    import { onMount } from 'svelte';
    import { fade } from 'svelte/transition';
    import scrollama from 'scrollama';
    import * as d3 from 'd3';
    import OntarioStations from '$lib/OntarioStationsOld.svelte';
    import WScoreMap from '$lib/WScoreMapOld.svelte';

    let canvas;
    let animationFrameId;

    let eventDroplets = $state([
        { id: 'hazel', name: 'Hurricane Hazel', year: 1954, x: 200, y: -100, vy: 3, a: 8 },
        { id: 'july-2013', name: 'July 2013 Flood', year: 2013, x: 500, y: -300, vy: 2, a: 8 },
        { id: 'lake-ontario', name: 'Lake Ontario Freshet', year: 2017, x: 800, y: -200, vy: 3, a: 8 }
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

    // Precip bars, scaled so the tallest day in this round fills the axis.
    /** @param {number[]} precip */
    function precipHeights(precip) {
        const max = Math.max(...precip, 1);
        return precip.map(v => (v / max) * 100);
    }

    // Build an SVG hydrograph path (0..100 x, 0..50 y viewBox) from raw flow.
    /** @param {number[]} flow */
    function flowPath(flow) {
        if (!flow || flow.length < 2) return '';
        const max = Math.max(...flow);
        const min = Math.min(...flow);
        const range = (max - min) || 1;
        const n = flow.length;
        return flow.map((v, i) => {
            const x = (i / (n - 1)) * 100;
            const y = 48 - ((v - min) / range) * 44; // higher flow -> higher line
            return `${i === 0 ? 'M' : 'L'} ${x.toFixed(1)} ${y.toFixed(1)}`;
        }).join(' ');
    }

    // Canada map state
    let canadaPath = $state('');
    let mapReady = $state(false);

    let monitoringFailures = $state([
        { id: 'f1', lon: -125, lat: 54, px: 0, py: 0, name: 'Northern BC, 1990', desc: 'Flash flood undetected — no upstream gauges.' },
        { id: 'f2', lon: -106, lat: 52, px: 0, py: 0, name: 'Prairie Event, 2005', desc: 'Radar gap delayed warnings by hours.' },
        { id: 'f3', lon: -80,  lat: 43, px: 0, py: 0, name: 'Southern Ontario, 2013', desc: 'Urban flooding underestimated by sparse sensors.' },
        { id: 'f4', lon: -63,  lat: 45, px: 0, py: 0, name: 'Atlantic Coast, 2019', desc: 'Storm surge unmonitored in remote inlet.' }
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

    function submitGuess(guess) {
        userGuess = guess;
        if ((guess === 'Y') === rounds[currentRound].flooded) {
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
    let showController = $derived(gameComplete || scene !== 0);
    let canGoForward = $derived(scene !== 13);

    function goForward() {
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

    function goBack() {
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

            function handleStepProgress(response) {
                if (!chosenDrop) return;
                if (response.index < 2) {
                    gameVisible = false;
                    return;
                }
                const globalProgress = (response.index - 2 + response.progress) / 2;

                let maxSize = Math.max(canvas.width, canvas.height);
                chosenDrop.a = 4 + (globalProgress * maxSize);

                if (globalProgress > 0.6) {
                    gameStarted = true;
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
                <h3>{hoveredEventData.name}</h3>
                <p>Interactive event data triggered!</p>
            </div>
        {/if}

        {#if gameVisible && !gameVerdict && !gameComplete && rounds[currentRound]}
            <div class="game-overlay" transition:fade={{ duration: 400 }}>
                <div class="game-card">
                    <h2>Guess the Flood</h2>
                    <p>Round {currentRound + 1} of {totalRounds} — did a flood occur?</p>

                    <div class="graphs-container">
                        <div class="graph">
                            <h4>Precipitation</h4>
                            <div class="mock-bars">
                                {#each precipHeights(rounds[currentRound].precip) as h}
                                    <div class="bar" style="height: {h}%"></div>
                                {/each}
                            </div>
                        </div>
                        <div class="graph">
                            <h4>Streamflow</h4>
                            <div class="mock-line-container">
                                <svg width="100%" height="100%" viewBox="0 0 100 50" preserveAspectRatio="none">
                                    <path d={flowPath(rounds[currentRound].flow)} fill="none" stroke="#60a5fa" stroke-width="3"/>
                                </svg>
                            </div>
                        </div>
                    </div>

                    {#if userGuess === null}
                        <div class="button-group">
                            <button class="btn-yes" onclick={() => submitGuess('Y')}>Flood: Yes</button>
                            <button class="btn-no" onclick={() => submitGuess('N')}>Flood: No</button>
                        </div>
                    {:else}
                        <div class="result-view" transition:fade>
                            {#if rounds[currentRound].flooded}
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
                    {#if correctCount >= 2}
                        <h2>You might be a meteorologist!</h2>
                        <p>You correctly called {correctCount} of {totalRounds} events.</p>
                    {:else}
                        <h2>Hard to tell, right?</h2>
                        <p>You got {correctCount} of {totalRounds}. Even the experts struggle without good data.</p>
                    {/if}
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
                                            onmouseenter={() => hoveredStarId = star.id}
                                            onmouseleave={() => hoveredStarId = null}
                                        >★</text>
                                    {/each}
                                {/if}
                            </svg>

                            {#if hoveredStar}
                                <div class="star-tooltip" style="left: {(hoveredStar.px / 1000) * 100}%; top: {(hoveredStar.py / 700) * 100}%;" transition:fade={{ duration: 200 }}>
                                    <strong>{hoveredStar.name}</strong>
                                    <span>{hoveredStar.desc}</span>
                                </div>
                            {/if}
                        </div>
                    </div>

                    <div class="text-column">
                        <h2 class="map-title">Where a lack of monitoring caused harm</h2>
                        <p>Across Canada, gaps in weather and streamflow monitoring have left communities without warning when it mattered most.</p>
                        <p>Each marked location represents a flood event where sparse sensor coverage, radar gaps, or missing upstream gauges delayed detection and response.</p>
                        <p>Hover over a star to see what happened.</p>
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

                        {#if benefitStage < benefits.length}
                            <button class="advance-arrow inline" onclick={advanceSequence} aria-label="Next">→</button>
                        {/if}
                        {#if benefitStage >= benefits.length && scene === 0}
                            <button class="advance-arrow" onclick={() => scene = 11} aria-label="Continue">→</button>
                        {/if}
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
        <WScoreMap />
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

    .graph h4 {
        margin: 0 0 0.5rem 0;
        text-align: left;
        font-size: 0.9rem;
        color: #cbd5e1;
    }

    .mock-bars {
        display: flex;
        align-items: flex-end;
        justify-content: space-around;
        gap: 3px;
        height: 80px;
        background: #f3f4f6;
        border-bottom: 2px solid #9ca3af;
        border-left: 2px solid #9ca3af;
        padding: 10px 5px 0 5px;
    }

    .mock-bars .bar {
        flex: 1;
        min-width: 2px;
        background-color: #3b82f6;
        border-radius: 2px 2px 0 0;
    }

    .mock-line-container {
        height: 80px;
        background: #f3f4f6;
        border-bottom: 2px solid #9ca3af;
        border-left: 2px solid #9ca3af;
        position: relative;
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
        margin-top: 1.5rem;
        animation: none;
    }

    @keyframes pulse {
        0%, 100% { transform: scale(1); box-shadow: 0 0 0 0 rgba(138,180,255,0.5); }
        50% { transform: scale(1.08); box-shadow: 0 0 0 12px rgba(138,180,255,0); }
    }

    .benefits-overlay {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: #0a0a12;
        z-index: 45;
        overflow-y: auto;
        display: flex;
        justify-content: center;
        padding: 3rem 2rem;
        box-sizing: border-box;
    }

    .benefits-content {
        max-width: 800px;
        width: 100%;
        color: white;
    }

    .benefits-intro {
        font-size: 1.3rem;
        line-height: 1.6;
        margin-bottom: 2rem;
        min-height: 2rem;
    }

    .benefit-row {
        display: flex;
        gap: 1.5rem;
        align-items: center;
        margin-bottom: 2rem;
        padding-bottom: 2rem;
        border-bottom: 1px solid rgba(255,255,255,0.1);
    }

    .benefit-text {
        flex: 1;
        font-size: 1.1rem;
        line-height: 1.5;
    }

    .benefit-num {
        color: #8ab4ff;
        font-weight: bold;
        margin-right: 0.5rem;
    }

    .benefit-media {
        flex: 1;
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
    }

    .benefit-media img {
        width: 100%;
        border-radius: 8px;
        background: #1a1a2e;
        min-height: 120px;
        object-fit: cover;
    }

    .benefit-caption {
        font-size: 0.8rem;
        color: #94a3b8;
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