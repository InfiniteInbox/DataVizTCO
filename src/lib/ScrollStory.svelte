<script>
    import { onMount } from 'svelte';
    import { fade } from 'svelte/transition';
    import scrollama from 'scrollama';
    import * as d3 from 'd3';
    import OntarioStations from '$lib/OntarioStations.svelte';
    import WScoreMap from '$lib/WScoreMap.svelte';
    import DisasterReports from '$lib/DisasterReports.svelte';
    import Conclusion from '$lib/Conclusion.svelte';

    let canvas;
    let animationFrameId;

    const COST_M = { hazel: 1300, 'july-2013': 1000, 'lake-ontario': 550 };
    const RADIUS_SCALE = 26 / Math.sqrt(1300);
    const costRadius = (id) => Math.sqrt(COST_M[id]) * RADIUS_SCALE;

    let hoveredEventId = $state(null);
    let eventDroplets = $state([
        { id: 'hazel', name: 'Hurricane Hazel', year: 1954, damage: '$1.3 billion in damage', x: 200, y: -100, vy: 3, a: costRadius('hazel') },
        { id: 'july-2013', name: 'July 2013 Flood', year: 2013, damage: '$1 billion in damage', x: 500, y: -300, vy: 2, a: costRadius('july-2013') },
        { id: 'lake-ontario', name: 'Lake Ontario Freshet', year: 2017, damage: '$550 million in damage', x: 800, y: -200, vy: 3, a: costRadius('lake-ontario') }
    ]);

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
    let harmIntroDone = $state(false);
    let gameIntroDismissed = $state(false);

    const EXAMPLE_PRECIP = [5, 12, 60, 88, 38, 8, 0, 4];
    const EXAMPLE_FLOW = [8, 9, 11, 22, 55, 78, 52, 28];

    /** @type {Array<{ name?: string, flooded: boolean, event: string|null, blurb?: string, dates?: string[], precip: number[], flow: number[], precipAvg?: number, flowAvg?: number }>} */
    let rounds = $state([]);
    let totalRounds = $derived(rounds.length);

    const finite = (arr) => (arr ?? []).filter((v) => typeof v === 'number' && isFinite(v));

    let round = $derived(rounds[currentRound]);
    let precipMax = $derived(round ? Math.max(...finite(round.precip), round.precipAvg ?? 0, 1) : 1);
    let flowVals = $derived(round ? finite(round.flow) : []);
    let flowMax = $derived(flowVals.length ? Math.max(...flowVals, round?.flowAvg ?? -Infinity) : 1);
    let flowMin = $derived(flowVals.length ? Math.min(...flowVals, round?.flowAvg ?? Infinity) : 0);
    let precipAvgPct = $derived(round?.precipAvg != null ? (round.precipAvg / precipMax) * 100 : null);
    let flowAvgY = $derived(round?.flowAvg != null ? 48 - ((round.flowAvg - flowMin) / ((flowMax - flowMin) || 1)) * 44 : null);
    let roundDays = $derived(round ? round.precip.length : 0);
    let missingDays = $derived(round
        ? round.precip.filter(v => v == null).length + round.flow.filter(v => v == null).length
        : 0);

    const fmtTick = (v) => (v >= 100 ? Math.round(v).toString()
        : v >= 10 ? v.toFixed(0)
        : v.toFixed(1));

    /** @param {Array<number|null>} precip */
    function precipHeights(precip) {
        const max = Math.max(...finite(precip), 1);
        return (precip ?? []).map(v =>
            v == null ? { missing: true, h: 0 } : { missing: false, h: (v / max) * 100 }
        );
    }

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
            y: 48 - ((v - min) / range) * 44
        });

        const paths = [];
        const points = [];
        let run = [];
        const flush = () => {
            if (run.length >= 2) {
                paths.push(run.map((p, k) => `${k === 0 ? 'M' : 'L'} ${p.x.toFixed(1)} ${p.y.toFixed(1)}`).join(' '));
            } else if (run.length === 1) {
                points.push(run[0]);
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

    let canadaPath = $state('');
    let mapReady = $state(false);

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

    let showArrow = $state(false);
    let sequenceStarted = $state(false);
    let benefitStage = $state(0);
    let introTyped = $state({ text: '' });

    let benefits = $state([
        { full: 'improve early warning systems for storms', typed: { text: '' }, caption: 'Rain cells on urban radar; the Northern Mesonet Project', media: '/radar-cells.gif', link: 'https://www.uwo.ca/nmp/index.html' },
        { full: 'inform smarter city planning', typed: { text: '' }, caption: 'Cities adapting rural flood-mitigation techniques', media: '/city-planning.jpg', link: 'https://www.sciencedirect.com/science/article/pii/S221458182500285X' },
        { full: 'learn more about the micro-scale changes of climate change', typed: { text: '' }, caption: 'The Canadian Severe Storms Lab, where the Northern Hail and Tornado Project are in', media: '/hail-project.jpg', link: 'https://www.uwo.ca/cssl/index.html' },
        { full: 'advance weather forecasting knowledge and techniques', typed: { text: '' }, caption: 'AI forecasting: GraphCast, NVIDIA Earth-2, GenCast, FourCastNet', media: '/ai-forecasting.jpg', link: 'https://wmo.int/media/magazine-article/forecasting-future-role-of-artificial-intelligence-transforming-weather-prediction-and-policy' }
    ]);

    let typeTimers = [];

    function typewrite(full, target, speed = 25) {
        let i = 0;
        target.text = '';
        const id = setInterval(() => {
            target.text = full.slice(0, i + 1);
            i++;
            if (i >= full.length) clearInterval(id);
        }, speed);
        typeTimers.push(id);
        return id;
    }

    function clearTypewriters() {
        typeTimers.forEach(clearInterval);
        typeTimers = [];
    }

    function startSequence() {
        clearTypewriters();
        scene = 15;
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

    $effect(() => {
        if (gameComplete && scene === 14 && harmIntroDone) {
            const t = setTimeout(() => { showArrow = true; }, 5000);
            return () => clearTimeout(t);
        }
        showArrow = false;
    });

    function submitGuess(guess) {
        userGuess = guess;
        const truth = rounds[currentRound].flooded;
        if (truth == null) return;
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

    let droplets = [];
    let chosenDrop = null;

    function resetAllDroplets() {
        chosenDrop = null;
        for (const drop of droplets) {
            drop.a = 4;
            drop.targetA = 4;
            if (drop.vy === 0) {
                drop.vy = Math.random() * 2 + 5;
            }
        }
    }

    function ensureChosenDrop() {
        if (chosenDrop || !droplets.length || !canvas) return;
        let mindist = Infinity;
        for (const drop of droplets) {
            const dist = Math.hypot(drop.x - (canvas.width / 2), drop.y - (canvas.height / 2));
            if (dist < mindist) {
                mindist = dist;
                chosenDrop = drop;
            }
        }
        if (chosenDrop) {
            chosenDrop.vy = 0;
            chosenDrop.x = canvas.width / 2;
            chosenDrop.y = canvas.height / 2;
        }
    }

    function openBiggerPicture() {
        resetAllDroplets();
        showArrow = false;
        sequenceStarted = false;
        benefitStage = 0;
        gameComplete = true;
        scene = 11;
    }

    let showController = $derived(gameComplete || gameVerdict || scene !== 0);

    const chapters = [
        { key: 'rain', label: 'The Cost of Flooding' },
        { key: 'stations', label: 'The Bigger Picture' },
        { key: 'wscore', label: 'What the Data Shows' },
        { key: 'harm', label: 'Where Monitoring Failed' },
        { key: 'benefits', label: 'Why It Matters' },
        { key: 'reports', label: 'Who Reports a Disaster?' },
        { key: 'conclusion', label: 'What You Can Do' }
    ];

    let currentChapterKey = $derived(
        scene === 0 ? ((gameVisible || gameVerdict) ? 'game' : 'rain')
        : scene === 11 ? 'stations'
        : scene === 12 || scene === 13 ? 'wscore'
        : scene === 14 ? 'harm'
        : scene === 15 ? 'benefits'
        : scene === 18 ? 'conclusion'
        : 'reports'
    );
    let chaptersOpen = $state(false);
    const toggleChapters = () => chaptersOpen = !chaptersOpen;
    const closeChapters = () => chaptersOpen = false;

    function resetIntroState() {
        resetAllDroplets();
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
        gameIntroDismissed = false;
    }

    function resetToRain() {
        if (typeof window !== 'undefined') window.scrollTo({ top: 0, behavior: 'auto' });
        resetIntroState();
        scene = 0;
    }

    function scrollToGame() {
        if (typeof window === 'undefined' || typeof document === 'undefined') return;
        const steps = document.querySelectorAll('.step');
        const target = steps[2];
        if (!target) return;
        const top = target.getBoundingClientRect().top + window.scrollY;
        window.scrollTo({ top: top + window.innerHeight * 1.5, behavior: 'auto' });
    }

    function jumpToChapter(key) {
        closeChapters();
        if (key === currentChapterKey) return;
        if (key === 'rain') { resetToRain(); return; }
        if (key === 'game') {
            resetIntroState();
            scene = 0;
            scrollToGame();
            return;
        }

        resetAllDroplets();
        gameComplete = true;
        gameVerdict = false;
        showArrow = false;

        if (key === 'stations') { scene = 11; return; }
        if (key === 'wscore') { scene = 13; return; }
        if (key === 'harm') { harmIntroDone = true; scene = 14; return; }
        if (key === 'benefits') { harmIntroDone = true; startSequence(); return; }
        if (key === 'reports') { scene = 17; return; }
        if (key === 'conclusion') { scene = 18; return; }
    }

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
            droplets = Array.from({ length: num_drops }, () => ({
                x: Math.random() * canvas.width,
                y: Math.random() * canvas.height,
                vy: Math.random() * 2 + 5,
                a: a,
                targetA: a,
            }));

            function handleStepEnter(response) {
                currentStep = response.index;
                if (scene !== 0) return;

                if (response.index === 2 && !chosenDrop) {
                    ensureChosenDrop();
                }
            }

            const GAME_THRESHOLD = 0.6;

            function handleStepProgress(response) {
                if (scene !== 0) return;

                const scrollY = typeof window !== 'undefined' ? window.scrollY : 0;
                if (response.index < 2 || scrollY < window.innerHeight * 0.5) {
                    resetAllDroplets();
                    if (!gameVerdict && !gameComplete) {
                        currentRound = 0;
                        correctCount = 0;
                        userGuess = null;
                        showArrow = false;
                    }
                    gameVisible = false;
                    gameStarted = false;
                    return;
                }

                if (!chosenDrop) {
                    ensureChosenDrop();
                }
                if (!chosenDrop) return;

                const globalProgress = (response.index - 2 + response.progress) / 2;

                let maxSize = Math.max(canvas.width, canvas.height);
                chosenDrop.targetA = 4 + (globalProgress * maxSize);

                if (gameVerdict || gameComplete) return;

                const past = globalProgress > GAME_THRESHOLD;

                if (past !== gameStarted) {
                    gameStarted = past;
                    if (!past) {
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
                    chosenDrop.a += (chosenDrop.targetA - chosenDrop.a) * 0.15;

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
        <h2>An exploration into the inequity of weather monitoring across Ontario</h2>
        <p>Yash Jain</p>
        <div class="scroll-cue">
            <span>Scroll to begin</span>
            <span class="chevron">⌄</span>
        </div>
    </div>
{/if}

<div bind:this={introView} id="intro-view" class="intro-view">
    <div class="sticky-graphic">

        {#if gameVisible && !gameIntroDismissed && !gameVerdict && !gameComplete}
            <div class="game-overlay" transition:fade={{ duration: 400 }}>
                <div class="game-card game-intro-card">
                    <h2>Guess the Flood</h2>
                    <p class="game-intro-text">
                        The following game will display the streamflow data of a nearby station
                        along with the precipitation measured at a nearby station over the course
                        of 8-10 days. Your goal is to determine if a flood occurred within that
                        timeframe.
                    </p>

                    <div class="example-graphic">
                        <span class="example-label">Example</span>
                        <div class="graphs-container example-graphs">
                            <div class="graph">
                                <div class="graph-head">
                                    <h4>Precipitation</h4>
                                    <span class="axis-unit">mm / day</span>
                                </div>
                                <div class="plot example-plot">
                                    <div class="mock-bars">
                                        {#each precipHeights(EXAMPLE_PRECIP) as b}
                                            <div class="bar" style="height: {b.h}%"></div>
                                        {/each}
                                    </div>
                                </div>
                            </div>
                            <div class="graph">
                                <div class="graph-head">
                                    <h4>Streamflow</h4>
                                    <span class="axis-unit">m³ / s</span>
                                </div>
                                <div class="plot example-plot">
                                    <svg class="flow-svg" width="100%" height="100%" viewBox="0 0 100 50" preserveAspectRatio="none">
                                        {#each flowSegments(EXAMPLE_FLOW).paths as d}
                                            <path {d} fill="none" stroke="#60a5fa" stroke-width="2.5" vector-effect="non-scaling-stroke"/>
                                        {/each}
                                    </svg>
                                </div>
                            </div>
                        </div>
                        <p class="example-caption">Rain arrives, then the river responds — often a day or more later.</p>
                    </div>

                    <button class="btn-reset" onclick={() => gameIntroDismissed = true}>Start</button>
                </div>
            </div>
        {/if}

        {#if gameVisible && gameIntroDismissed && !gameVerdict && !gameComplete && rounds[currentRound]}
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
                            {#if precipAvgPct != null}
                                <p class="avg-legend"><span class="avg-swatch-line"></span>historical average ({fmtTick(round.precipAvg)} mm/day)</p>
                            {/if}
                            <div class="chart-row">
                                <div class="y-ticks">
                                    <span>{fmtTick(precipMax)}</span>
                                    <span>{fmtTick(precipMax / 2)}</span>
                                    <span>0</span>
                                </div>
                                <div class="plot">
                                    <div class="grid-line" style="top: 0"></div>
                                    <div class="grid-line" style="top: 50%"></div>
                                    {#if precipAvgPct != null}
                                        <div class="avg-line" style="bottom: {precipAvgPct}%" title="Historical average: {round.precipAvg} mm/day"></div>
                                    {/if}
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
                            {#if flowAvgY != null}
                                <p class="avg-legend"><span class="avg-swatch-line"></span>historical average ({fmtTick(round.flowAvg)} m³/s)</p>
                            {/if}
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
                                        {#if flowAvgY != null}
                                            <line x1="0" x2="100" y1={flowAvgY} y2={flowAvgY} stroke="#fbbf24" stroke-width="1.4" stroke-dasharray="3 2.5" vector-effect="non-scaling-stroke"/>
                                        {/if}
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
                    <h2>{correctCount} of {totalRounds}</h2>
                    <p class="unscored" transition:fade={{ delay: 500, duration: 700 }}>The importance of ground truth data for weather forecasting cannot be overstated.</p>
                    <button class="btn-reset" onclick={openBiggerPicture}>See the station distribution</button>
                </div>
            </div>
        {/if}

        {#if gameComplete && scene === 14 && harmIntroDone}
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
                        <p>Across Canada, gaps in weather and streamflow monitoring have left communities without warning when it mattered most.</p>
                        <p>Each marked location is a situation or expressed concern of this specific instance</p>
                        <p>Hover a star to see what happened. Click it to read the reporting.</p>
                    </div>
                </div>
            </div>
            {#if showArrow && !sequenceStarted}
                <button class="advance-arrow" onclick={startSequence} transition:fade aria-label="Continue">→</button>
            {/if}
        {/if}

        {#if gameComplete && scene === 15}
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
                                        <a href={benefit.link} target="_blank" rel="noopener noreferrer" class="benefit-media-link">
                                            <img src={benefit.media} alt={benefit.caption} />
                                        </a>
                                        <span class="benefit-caption">{benefit.caption}</span>
                                    </div>
                                {/if}
                            </div>
                        {/if}
                    {/each}

                    <div class="benefits-footer">
                        {#if benefitStage < benefits.length}
                            <button class="advance-arrow inline" onclick={advanceSequence} aria-label="Next">→</button>
                        {:else}
                            <button class="advance-arrow inline pulse" onclick={() => scene = 16} aria-label="Continue">→</button>
                        {/if}
                    </div>
                </div>
            </div>
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
        <p class="interstitial-line" transition:fade={{ duration: 1000 }}>However, its not what it seems</p>
        <button class="advance-arrow" onclick={() => scene = 13} aria-label="Continue">→</button>
    </div>
{/if}

{#if scene === 13}
    <div class="scene-fullscreen">
        <WScoreMap onContinue={() => scene = 14} />
    </div>
{/if}

{#if scene === 14 && !harmIntroDone}
    <div class="scene-fullscreen interstitial">
        <p class="interstitial-line" transition:fade={{ duration: 1000 }}>Where a lack of monitoring caused harm</p>
        <button class="advance-arrow" onclick={() => harmIntroDone = true} aria-label="Continue">→</button>
    </div>
{/if}

{#if scene === 16}
    <div class="scene-fullscreen interstitial">
        <p class="interstitial-line" transition:fade={{ duration: 1000 }}>Dont just take my work for it, the disaster databases clearly reflect where the most monitored regions are</p>
        <button class="advance-arrow" onclick={() => scene = 17} aria-label="Continue">→</button>
    </div>
{/if}

{#if scene === 17}
    <div class="scene-fullscreen">
        <DisasterReports onContinue={() => scene = 18} />
    </div>
{/if}

{#if scene === 18}
    <div class="scene-fullscreen">
        <Conclusion />
    </div>
{/if}

{#if showController}
    <div class="scene-controller" transition:fade={{ duration: 300 }}>
        <div class="chapters-wrap">
            {#if chaptersOpen}
                <button class="chapters-backdrop" onclick={closeChapters} aria-label="Close chapters"></button>
                <div class="chapters-panel" transition:fade={{ duration: 150 }}>
                    <span class="chapters-title">Chapters</span>
                    <ul>
                        {#each chapters as ch}
                            <li>
                                <button
                                    class="chapter-item"
                                    class:active={ch.key === currentChapterKey}
                                    onclick={() => jumpToChapter(ch.key)}
                                    aria-current={ch.key === currentChapterKey ? 'true' : undefined}
                                >
                                    {ch.label}
                                </button>
                            </li>
                        {/each}
                    </ul>
                </div>
            {/if}
            <button
                class="ctrl-btn chapters-btn"
                onclick={toggleChapters}
                aria-label="Chapters"
                aria-expanded={chaptersOpen}
                title="Chapters"
            >
                <svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">
                    <circle cx="4" cy="6" r="1.8" fill="currentColor" />
                    <rect x="9" y="4.6" width="12" height="2.8" rx="1.4" fill="currentColor" />
                    <circle cx="4" cy="12" r="1.8" fill="currentColor" />
                    <rect x="9" y="10.6" width="12" height="2.8" rx="1.4" fill="currentColor" />
                    <circle cx="4" cy="18" r="1.8" fill="currentColor" />
                    <rect x="9" y="16.6" width="12" height="2.8" rx="1.4" fill="currentColor" />
                </svg>
            </button>
        </div>
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

    .chapters-wrap {
        position: relative;
    }

    .chapters-btn {
        color: #f8fafc;
    }

    .chapters-backdrop {
        position: fixed;
        inset: 0;
        background: transparent;
        border: none;
        padding: 0;
        margin: 0;
        z-index: 119;
        cursor: default;
    }

    .chapters-panel {
        position: absolute;
        bottom: calc(100% + 0.75rem);
        left: 50%;
        transform: translateX(-50%);
        width: 15rem;
        background: rgba(10, 10, 18, 0.92);
        border: 1px solid rgba(138, 180, 255, 0.3);
        border-radius: 10px;
        padding: 0.75rem 0.9rem;
        backdrop-filter: blur(6px);
        box-shadow: 0 10px 30px rgba(0,0,0,0.6);
        z-index: 121;
    }

    .chapters-title {
        display: block;
        font-size: 0.68rem;
        letter-spacing: 0.08em;
        text-transform: uppercase;
        color: #8ab4ff;
        margin-bottom: 0.4rem;
    }

    .chapters-panel ul {
        list-style: none;
        margin: 0;
        padding: 0;
        display: flex;
        flex-direction: column;
        gap: 0.3rem;
    }

    .chapters-panel li {
        margin: 0;
        padding: 0;
    }

    .chapter-item {
        display: block;
        width: 100%;
        text-align: left;
        background: none;
        border: none;
        border-radius: 5px;
        font-size: 0.85rem;
        font-weight: 400;
        color: #94a3b8;
        line-height: 1.4;
        padding: 0.3rem 0.5rem 0.3rem 0.7rem;
        cursor: pointer;
        position: relative;
        transition: background 0.15s, color 0.15s;
    }

    .chapter-item:hover {
        color: #e2e8f0;
        background: rgba(138, 180, 255, 0.1);
        transform: none;
    }

    .chapter-item.active {
        color: #f8fafc;
        font-weight: 700;
    }

    .chapter-item.active::before {
        content: '';
        position: absolute;
        left: 0;
        top: 50%;
        transform: translateY(-50%);
        width: 5px;
        height: 5px;
        border-radius: 50%;
        background: #8ab4ff;
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
        pointer-events: none;
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

    .game-intro-card {
        max-width: 460px;
    }

    .game-intro-text {
        text-align: left;
        line-height: 1.55;
        color: #cbd5e1;
        font-size: 0.92rem;
        margin: 0.75rem 0 1.25rem;
    }

    .example-graphic {
        position: relative;
        background: rgba(255,255,255,0.03);
        border: 1px dashed rgba(138,180,255,0.3);
        border-radius: 10px;
        padding: 1.1rem 1rem 0.8rem;
    }

    .example-label {
        position: absolute;
        top: -0.6rem;
        left: 0.9rem;
        background: #0a0a12;
        padding: 0 0.5rem;
        font-size: 0.66rem;
        letter-spacing: 0.08em;
        text-transform: uppercase;
        color: #8ab4ff;
    }

    .example-graphs {
        margin: 0;
        gap: 1rem;
    }

    .example-plot {
        height: 60px;
    }

    .example-caption {
        margin: 0.7rem 0 0;
        font-size: 0.74rem;
        color: #94a3b8;
        text-align: left;
        line-height: 1.4;
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

    /* Historical-average reference line, drawn across the precipitation bars. */
    .avg-line {
        position: absolute;
        left: 0;
        right: 0;
        height: 0;
        border-top: 1.4px dashed #fbbf24;
        z-index: 2;
        pointer-events: none;
    }

    .avg-legend {
        display: flex;
        align-items: center;
        gap: 0.35rem;
        margin: -0.15rem 0 0.35rem;
        font-size: 0.68rem;
        color: #fbbf24;
    }
    .avg-swatch-line {
        width: 14px; height: 0; flex: none;
        border-top: 1.4px dashed #fbbf24;
    }

    .unscored {
        font-size: 1.05rem !important;
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

    .benefit-media-link {
    display: flex;
    align-items: center;
    justify-content: center;
    flex: 1 1 auto;
    min-height: 0;
    width: 100%;
    border-radius: 8px;
    overflow: hidden;
    background: #10101a; /* Dark background behind letterboxed edges if aspect ratios differ */
    }

    .benefit-media-link img {
        max-width: 100%;
        max-height: 100%;
        width: auto;
        height: auto;
        object-fit: contain;
        display: block;
        transition: transform 0.2s ease, opacity 0.2s ease;
    }

    .benefit-media-link:hover img {
        transform: scale(1.02);
        opacity: 0.9;
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