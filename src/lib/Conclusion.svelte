<script>
    import { base } from '$app/paths';
    import { fade } from 'svelte/transition';

    const ORGS = [
        {
            name: 'Climate Risk Institute (CRI)',
            tag: 'Sudbury, ON',
            desc: "Leads the Northern Climate Change Network (NCCN), pushing for technical climate data collection, vulnerability assessments, and infrastructure tracking.",
            url: 'https://climateriskinstitute.ca/',
            logo: '/orgs/cri.png'
        },
        {
            name: 'Straight Up North',
            tag: 'Laurentian University',
            desc: 'Explores sustainability and land-planning shifts in the face of a changing climate, encouraging data collection.',
            url: 'https://straightupnorth.ca/',
            logo: '/orgs/straight-up-north.png'
        },
        {
            name: 'Up North on Climate',
            tag: 'Laurentian University',
            desc: 'Spearheads climate tracking initiatives, especially along critical infrastructure routes.',
            url: 'https://www.upnorthonclimate.ca/',
            logo: '/orgs/up-north-on-climate.webp'
        },
        {
            name: 'Ontario Nature',
            tag: 'Province-wide',
            desc: 'Focuses on nature-based climate solutions and data monitoring — actively pushing for tracking of the stark contrast between warming in Southern Ontario and the much-needed carbon sinks of Northern Ontario.',
            url: 'https://ontarionature.org/',
            logo: '/orgs/ontario-nature.svg'
        },
        {
            name: 'Indigenous Climate Hub',
            tag: 'National',
            desc: 'Spearheads regional climate monitoring networks across data-sparse communities.',
            url: 'https://indigenousclimatehub.ca/',
            logo: '/orgs/indigenous-climate-hub.png'
        },
        {
            name: 'SmartICE',
            tag: 'Arctic & coastal Canada',
            desc: 'A social enterprise integrating Inuit traditional knowledge with real-time sea-ice monitoring technology, helping northern communities safely track rapidly changing ice conditions.',
            url: 'https://smartice.org/',
            logo: '/orgs/smartice.png'
        },
        {
            name: 'ECCC (Environment and Climate Change Canada)',
            tag: 'Federal',
            desc: "The federal organization in charge of Canada's climate monitoring, infrastructure and response.",
            url: 'https://www.canada.ca/en/environment-climate-change.html',
            logo: '/orgs/eccc.svg'
        }
    ];
</script>

<div class="conclusion-scene">
    <div class="conclusion-inner">
        <div class="intro" transition:fade={{ duration: 700 }}>
            <p>
                It is important that more people are aware of this problem. In our
                changing climate, it is imperative that we have the most accurate
                information of as many climatic factors as we can. This will make
                all of us, both rural and urban, more resilient in the face of
                severe weather events.
            </p>
            <p class="lede">Here are organisations in Ontario fighting for this exact cause.</p>
        </div>

        <div class="org-grid" transition:fade={{ duration: 700, delay: 200 }}>
            {#each ORGS as org}
                <a class="org-card" href={org.url} target="_blank" rel="noopener noreferrer">
                    <div class="org-logo-tile">
                        <img src={`${base}${org.logo}`} alt="{org.name} logo" loading="lazy" />
                    </div>
                    <div class="org-body">
                        <h3>{org.name}</h3>
                        <span class="org-tag">{org.tag}</span>
                        <p>{org.desc}</p>
                        <span class="org-link">Visit website &rarr;</span>
                    </div>
                </a>
            {/each}
        </div>
    </div>
</div>

<style>
    .conclusion-scene {
        width: 100%;
        height: 100vh;
        overflow-y: auto;
        background: #0a0a12;
        color: white;
        font-family: sans-serif;
        box-sizing: border-box;
        padding: 4rem 1.5rem 6rem;
    }

    .conclusion-inner {
        max-width: 1000px;
        margin: 0 auto;
    }

    .intro {
        max-width: 700px;
        margin: 0 auto 3rem;
        text-align: center;
    }

    .intro p {
        line-height: 1.6;
        font-size: 1.05rem;
        color: #cbd5e1;
        margin: 0 0 1rem;
    }

    .intro .lede {
        font-size: 1.2rem;
        font-weight: 700;
        color: #f8fafc;
        margin: 0;
    }

    .org-grid {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
        gap: 1.25rem;
    }

    .org-card {
        display: flex;
        gap: 1rem;
        background: rgba(255,255,255,0.04);
        border: 1px solid rgba(138,180,255,0.2);
        border-radius: 12px;
        padding: 1.1rem;
        text-decoration: none;
        color: inherit;
        transition: transform 0.15s, border-color 0.15s, background 0.15s;
    }

    .org-card:hover {
        transform: translateY(-2px);
        border-color: rgba(138,180,255,0.55);
        background: rgba(255,255,255,0.07);
    }

    .org-logo-tile {
        flex: none;
        width: 64px;
        height: 64px;
        border-radius: 10px;
        background: #f8fafc;
        display: flex;
        align-items: center;
        justify-content: center;
        overflow: hidden;
        padding: 8px;
        box-sizing: border-box;
    }

    .org-logo-tile img {
        max-width: 100%;
        max-height: 100%;
        object-fit: contain;
    }

    .org-body {
        display: flex;
        flex-direction: column;
        min-width: 0;
    }

    .org-body h3 {
        margin: 0;
        font-size: 0.98rem;
        line-height: 1.3;
        color: #f8fafc;
    }

    .org-tag {
        font-size: 0.66rem;
        letter-spacing: 0.05em;
        text-transform: uppercase;
        color: #8ab4ff;
        margin: 0.2rem 0 0.4rem;
    }

    .org-body p {
        margin: 0 0 0.6rem;
        font-size: 0.8rem;
        line-height: 1.5;
        color: #94a3b8;
    }

    .org-link {
        margin-top: auto;
        font-size: 0.76rem;
        font-weight: 600;
        color: #8ab4ff;
    }
</style>
