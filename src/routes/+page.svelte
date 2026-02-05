<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import gsap from 'gsap';

	let sigLines: string[] = [];
	let maxLen = 0;

	let displayedSig = '';
	let reveal = 0; // 0..1

	const charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789#$%&@*+-=<>?/\\|[]{}()';
	const fps = 30;

	// base grid size (we scale the whole block)
	const baseFontSize = 16;
	const baseLineHeight = 16;
	let scale = 1;

	let intervalId: ReturnType<typeof setInterval> | null = null;
	let revealRaf = 0;

	let stackEl: HTMLDivElement;
	let sigEl: HTMLPreElement;

	let cleanupTilt: (() => void) | null = null;

	function randomChar() {
		return charset[(Math.random() * charset.length) | 0];
	}

	function parseSignature(text: string) {
		const lines = text.replace(/\r/g, '').split('\n');
		while (lines.length && lines[0].trim().length === 0) lines.shift();
		while (lines.length && lines[lines.length - 1].trim().length === 0) lines.pop();

		sigLines = lines;
		maxLen = Math.max(...sigLines.map((l) => l.length), 0);
	}

	function buildFrame(t: number) {
		const p = Math.max(0, Math.min(1, t));
		const lock = p * p;

		let out = '';
		for (let r = 0; r < sigLines.length; r++) {
			const line = sigLines[r].padEnd(maxLen, ' ');
			for (let c = 0; c < maxLen; c++) {
				const ch = line[c];
				out += ch === ' ' ? ' ' : Math.random() < lock ? ch : randomChar();
			}
			if (r !== sigLines.length - 1) out += '\n';
		}
		return out;
	}

	function startRenderLoop() {
		if (intervalId) clearInterval(intervalId);
		intervalId = setInterval(
			() => {
				if (!sigLines.length) return;
				displayedSig = buildFrame(reveal);
			},
			Math.floor(1000 / fps)
		);
	}

	function animateReveal(durationMs = 4500) {
		const start = performance.now();

		const step = (now: number) => {
			const t = Math.min(1, (now - start) / durationMs);
			const eased = 1 - Math.pow(1 - t, 3);
			reveal = eased;

			if (t < 1) revealRaf = window.requestAnimationFrame(step);
		};

		revealRaf = window.requestAnimationFrame(step);
	}

	function computeScale() {
		const pad = 24;

		const vw = Math.max(1, window.innerWidth - pad * 2);
		const vh = Math.max(1, window.innerHeight - pad * 2);

		const sigW = Math.max(1, maxLen) * baseFontSize;
		const sigH = Math.max(1, sigLines.length) * baseLineHeight;

		const s = Math.min(vw / sigW, vh / sigH, 1);
		scale = Math.max(0.5, s);
	}

	function setupTilt() {
		if (!stackEl || !sigEl) return;
		if (window.matchMedia?.('(prefers-reduced-motion: reduce)')?.matches) return;

		gsap.set(stackEl, {
			transformPerspective: 650,
			transformStyle: 'preserve-3d',
			rotationX: 0,
			rotationY: 0
		});
		gsap.set(sigEl, { x: 0, y: 0 });

		const rotX = gsap.quickTo(stackEl, 'rotationX', { ease: 'power3', duration: 0.2 });
		const rotY = gsap.quickTo(stackEl, 'rotationY', { ease: 'power3', duration: 0.2 });
		const innerX = gsap.quickTo(sigEl, 'x', { ease: 'power3', duration: 0.2 });
		const innerY = gsap.quickTo(sigEl, 'y', { ease: 'power3', duration: 0.2 });

		const onMove = (e: PointerEvent) => {
			const nx = e.clientX / window.innerWidth;
			const ny = e.clientY / window.innerHeight;

			rotX(gsap.utils.interpolate(10, -10, ny));
			rotY(gsap.utils.interpolate(-10, 10, nx));

			innerX(gsap.utils.interpolate(-10, 10, nx));
			innerY(gsap.utils.interpolate(-10, 10, ny));
		};

		const onLeave = () => {
			rotX(0);
			rotY(0);
			innerX(0);
			innerY(0);
		};

		window.addEventListener('pointermove', onMove, { passive: true });
		window.addEventListener('pointerleave', onLeave, { passive: true });

		cleanupTilt = () => {
			window.removeEventListener('pointermove', onMove);
			window.removeEventListener('pointerleave', onLeave);
		};
	}

	onMount(async () => {
		const text = await fetch('/brav3.txt').then((r) => r.text());
		parseSignature(text);

		displayedSig = buildFrame(0);

		computeScale();
		window.addEventListener('resize', computeScale);

		startRenderLoop();
		animateReveal(6500);

		setupTilt();
	});

	onDestroy(() => {
		if (typeof window === 'undefined') return;

		window.removeEventListener('resize', computeScale);

		if (intervalId) clearInterval(intervalId);
		intervalId = null;

		window.cancelAnimationFrame(revealRaf);

		cleanupTilt?.();
		cleanupTilt = null;
	});
</script>

<div class="page">
	<div class="stage">
		<div class="stack" bind:this={stackEl} style:transform={`scale(${scale})`} aria-label="brav3">
			<pre class="sig" bind:this={sigEl}>{displayedSig}</pre>

			<!-- command-line style, bottom-right of the signature box -->
			<div class="cli" aria-label="coming soon terminal">
				<span class="prompt">&gt;</span>
				<span class="cmd">coming soon</span>
				<span class="cursor" aria-hidden="true">█</span>
			</div>
		</div>
	</div>
</div>

<style>
	.page {
		position: fixed;
		inset: 0;
		background: #000;
	}

	.stage {
		position: fixed;
		inset: 0;
		display: grid;
		place-items: center;
	}

	.stack {
		position: relative; /* cli'yi içeride konumlamak için */
		transform-origin: center;
		will-change: transform;
	}

	.sig {
		margin: 0;
		padding: 0;
		white-space: pre;
		font-family:
			ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', 'Courier New',
			monospace;
		font-size: 24px;
		line-height: 24px;
		color: rgba(200, 255, 200, 0.95);
		text-shadow:
			0 0 10px rgba(120, 255, 120, 0.45),
			0 0 22px rgba(120, 255, 120, 0.25);
		user-select: none;
		will-change: transform;
	}
	.cli {
		position: absolute;
		right: 0;
		top: 100%;
		margin-top: 10px;
		display: inline-flex;
		align-items: center;
		gap: 8px;

		font-family:
			ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', 'Courier New',
			monospace;
		font-size: 16px;
		letter-spacing: 0.08em;
		color: rgba(170, 255, 170, 0.78);
		text-shadow: 0 0 10px rgba(120, 255, 120, 0.25);
		user-select: none;
		white-space: nowrap;
	}
	.prompt {
		color: rgba(200, 255, 200, 0.9);
	}
	.cmd {
		text-transform: lowercase;
	}
	.cursor {
		display: inline-block;
		transform: translateY(-1px);
		animation: blink 1s steps(1, end) infinite;
		color: rgba(200, 255, 200, 0.9);
	}

	@keyframes blink {
		50% {
			opacity: 0;
		}
	}
</style>
