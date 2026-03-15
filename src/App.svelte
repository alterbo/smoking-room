<script lang="ts">
    import { onDestroy, onMount } from 'svelte';

    type SceneState = 'idle' | 'smoking';

    type SmokeParticle = {
        id: number;
        progress: number;
        speed: number;
        size: number;
        driftAmp: number;
        driftFreq: number;
        driftPhase: number;
        colorPhase: number;
    };

    type BurnBlock = {
        id: number;
        flickerPhase: number;
        colorPhase: number;
    };

    type PathPoint = { x: number; y: number; angle: number };

    let sceneState: SceneState = 'idle';
    let smokePathEl: SVGPathElement;
    let smokePathLength = 1;
    let smokeParticles: SmokeParticle[] = [];
    let burnBlocks: BurnBlock[] = [];
    let pathPoints: PathPoint[] = [];
    let animationsEnabled = true;
    let staticSmokeSlots: { progress: number; colorPhase: number }[] = [];
    let staticColorTimer = 0;
    const staticColorInterval = 1.5;

    let frameId = 0;
    let particleId = 0;
    let spawnAccumulator = 0;
    let lastTs = 0;
    let simAccumulator = 0;
    let animPhase = 0;
    let smokingElapsedMs = 0;

    let idleStartedAt = 0;
    const idleEmitLimitMs = 30_000;

    let smokingStartedAt = 0;
    const smokingDurationMs = 3000;
    const burnSampleLabelX = 0;
    const burnSampleLabelY = 19.273;
    const burnSampleRotateCx = -202.5;
    const burnSampleRotateCy = 330.5;
    const burnStartX = burnSampleRotateCx - (burnSampleLabelY - burnSampleRotateCy);
    const burnY = burnSampleRotateCy + (burnSampleLabelX - burnSampleRotateCx);
    const burnBlocksMax = 9;
    const burnTextStep = 20;
    const burnFontSize = 20;
    const smokeSpacing = 0.026;

    // No-anim: show all blocks instantly on click, nothing in between
    $: burnActiveCountNow = sceneState === 'smoking'
        ? (animationsEnabled ? burnActiveCount(smokingElapsedMs) : burnBlocksMax)
        : 0;

    function clamp(value: number, min: number, max: number): number {
        return Math.min(max, Math.max(min, value));
    }

    function toHex(value: number): string {
        return clamp(Math.round(value), 0, 255).toString(16).padStart(2, '0').toUpperCase();
    }

    function grayHex(value: number): string {
        const hex = toHex(value);
        return `#${hex}${hex}${hex}`;
    }

    function initBurnBlocks(): BurnBlock[] {
        return Array.from({ length: burnBlocksMax }, (_, index) => ({
            id: index,
            flickerPhase: Math.random() * Math.PI * 2,
            colorPhase: Math.random() * Math.PI * 2
        }));
    }

    function spawnSmokeParticle(): void {
        const latestProgress = smokeParticles.length > 0 ? smokeParticles[smokeParticles.length - 1].progress : 1;
        if (latestProgress < smokeSpacing) {
            return;
        }

        smokeParticles = [
            ...smokeParticles,
            {
                id: particleId++,
                progress: 0,
                speed: 0.11 + Math.random() * 0.03,
                size: 18 + Math.random() * 2,
                driftAmp: 2.5 + Math.random() * 2,
                driftFreq: 0.35 + Math.random() * 0.35,
                driftPhase: Math.random() * Math.PI * 2,
                colorPhase: Math.random() * 0.8
            }
        ];
    }

    function buildPathLookup(): void {
        const samples = 300;
        pathPoints = [];
        for (let i = 0; i <= samples; i++) {
            const t = i / samples;
            const len = smokePathLength * t;
            const point = smokePathEl.getPointAtLength(len);
            const next = smokePathEl.getPointAtLength(Math.min(len + 2, smokePathLength));
            const angle = (Math.atan2(next.y - point.y, next.x - point.x) * 180) / Math.PI;
            pathPoints.push({ x: point.x, y: point.y, angle });
        }
    }

    function smokeColor(progress: number, colorPhase: number): string {
        const stepped = Math.round(clamp(progress + colorPhase * 0.05, 0, 1) * 18) / 18;
        const gray = Math.round(85 + (210 - 85) * stepped);
        return grayHex(gray);
    }

    function smokeOpacity(progress: number): number {
        return Math.pow(1 - clamp(progress, 0, 1), 1.4);
    }

    function burnColor(index: number, block: BurnBlock): string {
        const edge = index / (burnBlocksMax - 1);
        // In no-anim mode use a frozen phase derived from block.colorPhase so colors are static
        const wave = animationsEnabled ? animPhase * 7.2 + block.colorPhase : block.colorPhase;
        const heatWave = (Math.sin(wave) + 1) * 0.5;
        const heat = clamp((1 - edge * 0.84) * 0.72 + heatWave * 0.28, 0, 1);
        const r = Math.round(clamp(160 + heat * 95, 0, 255));
        const g = Math.round(clamp(heat * 215, 0, 255));
        const b = Math.round(clamp(heat * heat * 25, 0, 255));
        return `#${toHex(r)}${toHex(g)}${toHex(b)}`;
    }

    function burnOpacity(index: number, block: BurnBlock): number {
        if (!animationsEnabled) return 1;
        const edge = index / (burnBlocksMax - 1);
        const flicker = (Math.sin(animPhase * 13 + block.flickerPhase) + 1) * 0.5;
        return clamp(0.58 + (1 - edge * 0.52) * 0.34 + flicker * 0.1, 0, 1);
    }

    function burnActiveCount(elapsedMs: number): number {
        const t = clamp(elapsedMs / smokingDurationMs, 0, 1);
        const ignite = t < 0.18 ? t / 0.18 : 1;
        const sustain = t < 0.62 ? 1 : clamp(1 - (t - 0.62) / 0.38, 0, 1);
        // Dual-oscillator flicker makes the block count rise and fall dynamically (±1-2 blocks)
        const flicker = (Math.sin(animPhase * 11.3) + Math.sin(animPhase * 7.7 + 1.3)) * 0.75;
        return Math.max(0, Math.min(burnBlocksMax, Math.round(ignite * sustain * burnBlocksMax + flicker)));
    }

    function canEmitSmoke(now: number): boolean {
        if (sceneState === 'smoking') {
            return true;
        }
        return now - idleStartedAt < idleEmitLimitMs;
    }

    function animate(ts: number): void {
        if (!lastTs) {
            lastTs = ts;
        }

        const dt = (ts - lastTs) / 1000;
        lastTs = ts;

        const step = animationsEnabled ? 1 / 24 : 1 / 4;

        simAccumulator += dt;
        while (simAccumulator >= step) {
            animPhase += step;

            if (sceneState === 'smoking' && ts - smokingStartedAt >= smokingDurationMs) {
                sceneState = 'idle';
                idleStartedAt = ts;
                smokingElapsedMs = 0;
            }

            if (sceneState === 'smoking') {
                smokingElapsedMs = ts - smokingStartedAt;
            }

            if (animationsEnabled) {
                const emitRate = sceneState === 'smoking' ? 9.5 : 7.5;
                if (canEmitSmoke(ts)) {
                    spawnAccumulator += step * emitRate;
                    while (spawnAccumulator >= 1) {
                        spawnSmokeParticle();
                        spawnAccumulator -= 1;
                    }
                } else {
                    spawnAccumulator = 0;
                }
                const nextSmokeParticles: SmokeParticle[] = [];
                for (const particle of smokeParticles) {
                    particle.progress += particle.speed * step;
                    if (particle.progress < 1) {
                        nextSmokeParticles.push(particle);
                    }
                }
                smokeParticles = nextSmokeParticles;
            } else {
                // Clear any leftover particles from a previous animated session
                if (smokeParticles.length > 0) smokeParticles = [];
                // Update static slot colors on a slow cadence
                staticColorTimer += step;
                if (staticColorTimer >= staticColorInterval) {
                    staticColorTimer = 0;
                    staticSmokeSlots = staticSmokeSlots.map(s => ({
                        ...s,
                        colorPhase: Math.random() * 0.8
                    }));
                }
            }

            simAccumulator -= step;
        }

        frameId = requestAnimationFrame(animate);
    }

    function triggerSmoke(): void {
        const now = performance.now();
        sceneState = 'smoking';
        smokingStartedAt = now;
        burnBlocks = initBurnBlocks();
        smokingElapsedMs = 0;
    }

    function pointOnSmokePath(progress: number): PathPoint {
        const i = Math.round(clamp(progress, 0, 1) * (pathPoints.length - 1));
        return pathPoints[i];
    }

    onMount(() => {
        smokePathLength = smokePathEl.getTotalLength();
        buildPathLookup();
        // 10 static slots evenly distributed along the smoke path
        staticSmokeSlots = Array.from({ length: 10 }, (_, i) => ({
            progress: (i + 0.5) / 10,
            colorPhase: Math.random() * 0.8
        }));
        const params = new URLSearchParams(window.location.search);
        if (params.get('animations') === 'false') {
            animationsEnabled = false;
        }
        idleStartedAt = performance.now();
        frameId = requestAnimationFrame(animate);
    });

    onDestroy(() => {
        cancelAnimationFrame(frameId);
    });
</script>

<main>
    <hgroup class="container">
        <h1>Smoking room</h1>
        <p>I have quit smoking! Alberto Ferrero, March 2026.</p>
    </hgroup>

    <div class="scene">
        <svg
            class:smoking={sceneState === 'smoking'}
            class:anim-off={!animationsEnabled}
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 1000 1000"
        >
            <path
                stroke="currentColor"
                d="M629.507 751C622.981 791.698 681.304 919.889 695.599 935.277C709.895 950.666 710.879 985.504 709.584 1001"
            />
            <path
                stroke="currentColor"
                d="M855.897 738C857.425 710.573 859.688 666.334 861 631.429M793 407.412C801.675 385.662 841.438 390.012 848.668 401.612C853.385 409.181 856.564 476.328 858.806 526.307"
            />
            <path
                stroke="currentColor"
                d="M808 701C803.655 653.218 803.076 586.356 803.655 577.668C804.38 566.808 800.759 411.879 789.897 407.535C779.035 403.191 738.483 393.055 735.586 419.842C732.69 446.629 736.311 571.876 738.483 574.772C740.655 577.668 742.828 668.421 724 683.625"
            />
            <path
                stroke="currentColor"
                d="M694.316 678.193C702.977 674.724 716.692 679.639 722.466 682.53C738.587 709.272 771.26 764.779 772.993 772.874C775.158 782.993 785.263 842.259 780.933 852.377C776.602 862.496 737.624 856.714 732.571 852.377C727.519 848.041 722.466 792.389 715.97 786.607C709.474 780.824 673.383 712.885 671.218 705.658C669.052 698.43 683.489 682.53 694.316 678.193Z"
            />
            <path
                stroke="currentColor"
                d="M670.908 703.757C642.034 683.768 617.448 712.926 618.027 729.227C616.868 735.631 652.798 782.836 670.908 805.638C675.012 824.558 684.526 862.837 689.742 864.583C696.261 866.766 720.89 871.133 739 857.306"
            />

            <g class="cigarette">
                <path
                    stroke="currentColor"
                    d="M736.915 525H77.9999C48.3999 567.4 65.6666 614 77.9999 632C193.156 631.747 593.278 631.443 736.915 631.353M801.507 525H982.302C1005 571 982.302 632 982.302 632C982.302 632 954.908 632 936.497 631.475M804.41 631.328C836.199 631.323 851.83 631.332 877.711 631.36"
                />
            </g>

            {#if sceneState === 'smoking'}
                <g class="burn-layer">
                    {#each burnBlocks.slice(0, burnActiveCountNow) as block, index}
                        {@const x = burnStartX + index * burnTextStep}
                        {@const y = burnY}
                        {@const color = burnColor(index, block)}
                        <text
                            x={x}
                            y={y}
                            fill={color}
                            font-family="monospace"
                            font-size={burnFontSize}
                            letter-spacing="-0.02em"
                            opacity={burnOpacity(index, block)}
                            transform={`rotate(90 ${x} ${y})`}
                            style="white-space:pre"
                        >
                            {color}
                        </text>
                    {/each}
                </g>
            {/if}

            <path
                stroke="currentColor"
                d="M847.667 1003C848.386 985.814 851.119 949.119 856.298 939.824C862.772 928.205 922.473 867.933 931.105 800.4C939.736 732.866 942.613 712.534 941.894 698.737C941.175 684.939 934.701 587.633 931.105 582.55C927.508 577.467 893.701 570.931 880.754 617.406C870.396 654.586 875 669.69 875 698.737C864.93 724.152 839.754 764.092 834 800.4"
            />
            <path stroke="currentColor" d="M717 525C721.528 546.177 727.867 597.225 717 632" />
            <path stroke="currentColor" d="M78 525C91.6667 545 110.8 594.2 78 631" />

            <g class="smoke-guide" aria-hidden="true">
                <path
                    bind:this={smokePathEl}
                    stroke="currentColor"
                    d="M69 502C69 497 50 428 63 392C76 356 92.0004 281 69 234C45.9996 187 63 30 63 25"
                />
            </g>

            <g class="smoke-particles" opacity={animationsEnabled && sceneState === 'smoking' ? 0.35 : 1}>
                {#if animationsEnabled}
                    {#each smokeParticles as particle (particle.id)}
                        {@const pathPoint = pointOnSmokePath(particle.progress)}
                        {@const drift =
                            Math.sin(animPhase * particle.driftFreq * Math.PI * 2 + particle.driftPhase) *
                            particle.driftAmp *
                            (0.25 + particle.progress)}
                        {@const color = smokeColor(particle.progress, particle.colorPhase)}
                        <text
                            x={pathPoint.x + drift}
                            y={pathPoint.y}
                            fill={color}
                            font-family="monospace"
                            font-size={particle.size}
                            letter-spacing="0em"
                            opacity={smokeOpacity(particle.progress)}
                            transform={`rotate(${pathPoint.angle + 90} ${pathPoint.x + drift} ${pathPoint.y})`}
                            style="white-space:pre"
                        >
                            {color}
                        </text>
                    {/each}
                {:else}
                    {#each staticSmokeSlots as slot}
                        {@const pathPoint = pointOnSmokePath(slot.progress)}
                        {@const color = smokeColor(slot.progress, slot.colorPhase)}
                        <text
                            x={pathPoint.x}
                            y={pathPoint.y}
                            fill={color}
                            font-family="monospace"
                            font-size="18"
                            letter-spacing="0em"
                            opacity={smokeOpacity(slot.progress)}
                            transform={`rotate(${Math.round((pathPoint.angle + 90) / 15) * 15} ${pathPoint.x} ${pathPoint.y})`}
                            style="white-space:pre"
                        >
                            {color}
                        </text>
                    {/each}
                {/if}
            </g>

            <text
                class="sample-label"
                xml:space="preserve"
                fill="#FFD54D"
                font-family="monospace"
                font-size="20"
                letter-spacing="0em"
                style="white-space:pre"
                transform={`rotate(90 ${burnSampleRotateCx} ${burnSampleRotateCy})`}
            >
                <tspan x={burnSampleLabelX} y={burnSampleLabelY}>#FFD54D</tspan>
            </text>
            <text
                class="sample-label"
                xml:space="preserve"
                fill="#646464"
                font-family="monospace"
                font-size="20"
                letter-spacing="0em"
                style="white-space:pre"
                transform="rotate(-90 286.5 213.5)"
            >
                <tspan x="0" y="19.273">#646464</tspan>
            </text>
        </svg>
    </div>

    <div class="container">
        <label class="anim-toggle">
            <input type="checkbox" bind:checked={animationsEnabled} />
            Animations
        </label>
        <button type="button" on:click={triggerSmoke}>Smoke</button>
    </div>
</main>

<style>
    .container {
        padding-inline: 1rem;
    }

    .scene {
        width: min(920px, 96vw);
        margin-left: auto;
        margin-right: 0;
        contain: layout paint;
    }

    .scene svg {
        display: block;
        width: 100%;
        height: auto;
        transform-origin: 24% 58%;
        color: var(--fg);
    }

    .scene svg.smoking {
        animation: handPull 3s ease-in-out 1;
    }

    .scene svg.smoking.anim-off {
        animation: none;
    }

    .smoke-guide path {
        opacity: 0;
        pointer-events: none;
    }

    .smoke-particles,
    .burn-layer {
        pointer-events: none;
    }

    .sample-label {
        opacity: 0;
        pointer-events: none;
    }

    main {
        display: grid;
        gap: 1rem;
        grid-template-rows: auto 1fr auto;
        justify-items: end;
        align-items: center;
        min-height: 100svh;
        padding: 1rem;
    }

    .anim-toggle {
        display: flex;
        align-items: center;
        gap: 0.4rem;
        cursor: pointer;
        user-select: none;
    }

    button {
        margin-block: 1rem;
        padding: 0.5rem 1rem;
        cursor: pointer;
        background: var(--btn-bg);
        color: var(--fg);
        border: 1px solid var(--btn-border);
    }

    @keyframes handPull {
        0%, 100% {
            transform: translateX(0);
        }
        35%, 65% {
            transform: translateX(56px);
        }
    }
</style>
