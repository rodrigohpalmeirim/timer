<script lang="ts">
	import { onDestroy } from 'svelte';
	import { fly } from 'svelte/transition';

	// ─── Theme definitions ──────────────────────────────────────────────────────

	type ThemeId = 'void' | 'terminal' | 'chalk' | 'dusk' | 'ice';
	type Theme = {
		id: ThemeId;
		name: string;
		bg: string;
		text: string;
		textMuted: string;
		textDim: string;
		accent: string;
		accentHover: string;
		overflow: string;
		overflowHover: string;
		surface: string;
		surfaceHover: string;
		track: string;
		dot: string;
	};

	const themes: Theme[] = [
		{
			id: 'void',
			name: 'Void',
			bg: '#0a0a0a',
			text: '#ffffff',
			textMuted: '#444444',
			textDim: '#202020',
			accent: '#6366f1',
			accentHover: '#818cf8',
			overflow: '#ef4444',
			overflowHover: '#f87171',
			surface: 'rgba(255,255,255,0.06)',
			surfaceHover: 'rgba(255,255,255,0.11)',
			track: '#181818',
			dot: '#6366f1',
		},
		{
			id: 'terminal',
			name: 'Terminal',
			bg: '#080808',
			text: '#00ff41',
			textMuted: '#006e1c',
			textDim: '#003c0f',
			accent: '#00c032',
			accentHover: '#00ff41',
			overflow: '#ff3c3c',
			overflowHover: '#ff7070',
			surface: 'rgba(0,255,65,0.06)',
			surfaceHover: 'rgba(0,255,65,0.13)',
			track: '#001408',
			dot: '#00ff41',
		},
		{
			id: 'chalk',
			name: 'Chalk',
			bg: '#f3ede2',
			text: '#1c1614',
			textMuted: '#9c8e80',
			textDim: '#c0b8ac',
			accent: '#1d4ed8',
			accentHover: '#2563eb',
			overflow: '#b91c1c',
			overflowHover: '#dc2626',
			surface: '#e6ddd0',
			surfaceHover: '#d8cfc2',
			track: '#d4ccc0',
			dot: '#1d4ed8',
		},
		{
			id: 'dusk',
			name: 'Dusk',
			bg: '#07051a',
			text: '#f0e8ff',
			textMuted: '#7060a0',
			textDim: '#281840',
			accent: '#a855f7',
			accentHover: '#c084fc',
			overflow: '#f97316',
			overflowHover: '#fb923c',
			surface: 'rgba(255,255,255,0.07)',
			surfaceHover: 'rgba(255,255,255,0.13)',
			track: 'rgba(168,85,247,0.18)',
			dot: '#a855f7',
		},
		{
			id: 'ice',
			name: 'Ice',
			bg: '#010c18',
			text: '#c8e8ff',
			textMuted: '#1e6080',
			textDim: '#091820',
			accent: '#0ea5e9',
			accentHover: '#38bdf8',
			overflow: '#e879f9',
			overflowHover: '#f0abfc',
			surface: 'rgba(14,165,233,0.07)',
			surfaceHover: 'rgba(14,165,233,0.14)',
			track: '#061420',
			dot: '#0ea5e9',
		},
	];

	let themeIndex = $state(0);
	let theme = $derived(themes[themeIndex]);
	function cycleTheme() { themeIndex = (themeIndex + 1) % themes.length; }

	// ─── Timer sequence ───────────────────────────────────────────────────────

	type TimerEntry = { id: number; name: string; seconds: number };

	const STORAGE_KEY = 'timer-sequence';
	const DEFAULT_SEQUENCE: TimerEntry[] = [
		{ id: 1, name: 'Focus', seconds: 1500 },
		{ id: 2, name: 'Short break', seconds: 300 },
	];

	function loadFromStorage(): { sequence: TimerEntry[]; nextEntryId: number } {
		try {
			const raw = localStorage.getItem(STORAGE_KEY);
			if (raw) {
				const parsed = JSON.parse(raw);
				if (Array.isArray(parsed.sequence) && parsed.sequence.length > 0) {
					return { sequence: parsed.sequence, nextEntryId: parsed.nextEntryId ?? parsed.sequence.length + 1 };
				}
			}
		} catch {}
		return { sequence: DEFAULT_SEQUENCE, nextEntryId: 3 };
	}

	const stored = loadFromStorage();

	let sequence = $state<TimerEntry[]>(stored.sequence);
	let seqIndex = $state(0);
	let nextEntryId = $state(stored.nextEntryId);
	let sidebarOpen = $state(false);
	let uiHidden = $state(false);

	$effect(() => {
		localStorage.setItem(STORAGE_KEY, JSON.stringify({ sequence, nextEntryId }));
	});

	// ─── Timer logic ──────────────────────────────────────────────────────────

	let totalSeconds = $state(stored.sequence[0].seconds);
	let remaining = $state(stored.sequence[0].seconds);
	let running = $state(false);
	let overflowSeconds = $state(0);

	let intervalId: ReturnType<typeof setInterval> | null = null;

	function startInterval() {
		if (intervalId) return;
		intervalId = setInterval(() => {
			if (remaining > 0) remaining -= 1;
			else overflowSeconds += 1;
		}, 1000);
	}
	function stopInterval() { if (intervalId) { clearInterval(intervalId); intervalId = null; } }
	function start() { running = true; startInterval(); }
	function pause() { running = false; stopInterval(); }
	function reset() { pause(); remaining = totalSeconds; overflowSeconds = 0; }
	function toggleRunning() { if (running) pause(); else start(); }

	function loadTimer(index: number, autoStart = false) {
		const i = ((index % sequence.length) + sequence.length) % sequence.length;
		pause();
		seqIndex = i;
		totalSeconds = sequence[i].seconds;
		remaining = sequence[i].seconds;
		overflowSeconds = 0;
		if (autoStart) start();
	}

	function nextTimer() { loadTimer(seqIndex + 1, running); }

	function pad(n: number) { return String(n).padStart(2, '0'); }

	function formatDuration(s: number): string {
		const h = Math.floor(s / 3600), m = Math.floor((s % 3600) / 60), sec = s % 60;
		return h > 0 ? `${pad(h)}:${pad(m)}:${pad(sec)}` : `${pad(m)}:${pad(sec)}`;
	}

	function parseDuration(str: string): number {
		const parts = str.trim().split(':').map(p => parseInt(p) || 0);
		if (parts.length === 3) return Math.max(1, parts[0] * 3600 + parts[1] * 60 + parts[2]);
		if (parts.length === 2) return Math.max(1, parts[0] * 60 + parts[1]);
		return Math.max(1, parts[0]);
	}

	function addTimer() {
		sequence = [...sequence, { id: nextEntryId++, name: 'Timer', seconds: 300 }];
	}

	function removeTimer(id: number) {
		if (sequence.length <= 1) return;
		const idx = sequence.findIndex(t => t.id === id);
		const newSeq = sequence.filter(t => t.id !== id);
		sequence = newSeq;
		let newIdx = seqIndex;
		if (idx < seqIndex) newIdx = seqIndex - 1;
		else if (idx === seqIndex) {
			newIdx = Math.min(seqIndex, newSeq.length - 1);
			totalSeconds = newSeq[newIdx].seconds;
			remaining = newSeq[newIdx].seconds;
			overflowSeconds = 0;
		}
		seqIndex = newIdx;
	}

	function updateTimerName(id: number, name: string) {
		sequence = sequence.map(t => t.id === id ? { ...t, name } : t);
	}

	function updateTimerDuration(id: number, durStr: string) {
		const isActive = sequence[seqIndex]?.id === id;
		const seconds = parseDuration(durStr);
		sequence = sequence.map(t => t.id === id ? { ...t, seconds } : t);
		if (isActive) { totalSeconds = seconds; remaining = seconds; overflowSeconds = 0; }
	}

	let displayTime = $derived.by(() => {
		if (remaining > 0) {
			const h = Math.floor(remaining / 3600), m = Math.floor((remaining % 3600) / 60), s = remaining % 60;
			return h > 0 ? `${pad(h)}:${pad(m)}:${pad(s)}` : `${pad(m)}:${pad(s)}`;
		} else {
			const h = Math.floor(overflowSeconds / 3600), m = Math.floor((overflowSeconds % 3600) / 60), s = overflowSeconds % 60;
			return h > 0 ? `+${pad(h)}:${pad(m)}:${pad(s)}` : `+${pad(m)}:${pad(s)}`;
		}
	});

	let isOverflow = $derived(remaining === 0 && overflowSeconds > 0);
	let barProgress = $derived(isOverflow ? Math.min(overflowSeconds / totalSeconds, 1) : remaining / totalSeconds);

	let ca = $derived(isOverflow ? theme.overflow : theme.accent);
	let caHover = $derived(isOverflow ? theme.overflowHover : theme.accentHover);
	let ct = $derived(isOverflow ? theme.overflow : theme.text);

	const DOT_COUNT = 24;
	let filledDots = $derived(Math.round(barProgress * DOT_COUNT));

	const BLOCK_COUNT = 30;
	let filledBlocks = $derived(Math.round(barProgress * BLOCK_COUNT));

	let termBar = $derived.by(() => {
		const w = 22, f = Math.round(barProgress * w);
		return `[${'█'.repeat(f)}${'░'.repeat(w - f)}] ${Math.round(barProgress * 100)}%`;
	});

	let durationStr = $derived(formatDuration(totalSeconds));
	let currentName = $derived(sequence[seqIndex]?.name ?? '');

	// ─── Sidebar theme helpers ────────────────────────────────────────────────

	let sidebarBg = $derived(
		theme.id === 'chalk' ? '#faf5ec' :
		theme.id === 'terminal' ? '#090909' :
		theme.id === 'dusk' ? '#0e0c28' :
		theme.id === 'ice' ? '#050e1a' :
		'#0e0e0e'
	);
	let sidebarBorder = $derived(
		theme.id === 'chalk' ? '#ddd5c8' :
		theme.id === 'terminal' ? '#1a2a1a' :
		theme.id === 'dusk' ? 'rgba(168,85,247,0.25)' :
		theme.id === 'ice' ? 'rgba(14,165,233,0.18)' :
		'#1c1c1c'
	);
	let sidebarFont = $derived(theme.id === 'terminal' ? "'Courier New',Courier,monospace" : "inherit");

	function handleKeydown(e: KeyboardEvent) {
		if ((e.target as HTMLElement).tagName === 'INPUT') return;
		if (e.key === ' ' || e.key === 'k') { e.preventDefault(); toggleRunning(); }
		if (e.key === 'r') { e.preventDefault(); reset(); }
		if (e.key === 'n') { e.preventDefault(); nextTimer(); }
		if (e.key === 's') { e.preventDefault(); sidebarOpen = !sidebarOpen; }
		if (e.key === 'h') { e.preventDefault(); uiHidden = !uiHidden; }
		if (e.key === 'q') { e.preventDefault(); cycleTheme(); }
		if (e.key === 'Escape' && sidebarOpen) { e.preventDefault(); sidebarOpen = false; }
	}

	onDestroy(() => stopInterval());
</script>

<svelte:window onkeydown={handleKeydown} />

<!-- ─── Sidebar ──────────────────────────────────────────────────────────── -->
{#if sidebarOpen && !uiHidden}
<aside
	transition:fly={{ x: 300, duration: 220, opacity: 1 }}
	style="
		position:fixed; right:0; top:0; bottom:0; width:280px; z-index:200;
		background:{sidebarBg}; border-left:1px solid {sidebarBorder};
		display:flex; flex-direction:column; font-family:{sidebarFont};
		box-shadow:-8px 0 32px rgba(0,0,0,0.4);
	"
>
	<!-- Header -->
	<div style="display:flex; align-items:center; justify-content:space-between; padding:16px 16px 12px; border-bottom:1px solid {sidebarBorder};">
		<span style="font-size:11px; letter-spacing:0.12em; text-transform:uppercase; color:{theme.textMuted}; font-weight:500;">Timer Sequence</span>
		<button
			onclick={() => sidebarOpen = false}
			style="background:none; border:none; cursor:pointer; color:{theme.textMuted}; padding:2px; display:flex; align-items:center; justify-content:center; border-radius:4px; transition:color 0.15s;"
			onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.color=theme.text;}}
			onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.color=theme.textMuted;}}
			title="Close (S or Esc)"
		>
			<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
				<line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/>
			</svg>
		</button>
	</div>

	<!-- Timer list -->
	<div style="flex:1; overflow-y:auto; padding:8px 0;">
		{#each sequence as timer, i}
			<div style="
				display:flex; align-items:center; gap:8px; padding:8px 12px;
				border-left:2px solid {i === seqIndex ? ca : 'transparent'};
				background:{i === seqIndex ? (theme.id === 'chalk' ? 'rgba(0,0,0,0.04)' : 'rgba(255,255,255,0.04)') : 'transparent'};
				transition:all 0.15s;
			">
				<!-- Active indicator -->
				<div style="width:6px; height:6px; border-radius:50%; flex-shrink:0; background:{i === seqIndex ? ca : 'transparent'}; border:1px solid {i === seqIndex ? ca : theme.textMuted}; opacity:{i === seqIndex ? 1 : 0.4};"></div>

				<!-- Name input -->
				<input
					value={timer.name}
					oninput={(e) => updateTimerName(timer.id, (e.target as HTMLInputElement).value)}
					style="
						flex:1; min-width:0; background:transparent; border:none; outline:none;
						color:{i === seqIndex ? theme.text : theme.textMuted}; font-size:13px;
						font-family:{sidebarFont}; padding:2px 0;
						border-bottom:1px solid transparent; transition:border-color 0.15s;
					"
					onfocus={(e)=>{(e.target as HTMLInputElement).style.borderBottomColor=theme.textMuted;}}
					onblur={(e)=>{(e.target as HTMLInputElement).style.borderBottomColor='transparent';}}
					title="Timer name"
				/>

				<!-- Duration input -->
				<input
					value={formatDuration(timer.seconds)}
					onblur={(e) => { updateTimerDuration(timer.id, (e.target as HTMLInputElement).value); (e.target as HTMLInputElement).style.borderBottomColor='transparent'; }}
					onkeydown={(e) => { if (e.key === 'Enter') (e.target as HTMLInputElement).blur(); }}
					onfocus={(e) => { (e.target as HTMLInputElement).select(); (e.target as HTMLInputElement).style.borderBottomColor=ca; }}
					style="
						width:52px; flex-shrink:0; background:transparent; border:none; outline:none;
						color:{i === seqIndex ? ca : theme.textMuted}; font-size:12px; font-family:monospace;
						text-align:right; padding:2px 0;
						border-bottom:1px solid transparent; transition:border-color 0.15s;
					"
					title="Duration (MM:SS or H:MM:SS)"
				/>

				<!-- Remove button -->
				<button
					onclick={() => removeTimer(timer.id)}
					disabled={sequence.length <= 1}
					style="
						background:none; border:none; cursor:pointer; color:{theme.textMuted}; padding:2px;
						opacity:{sequence.length <= 1 ? 0.2 : 0.5}; display:flex; align-items:center;
						flex-shrink:0; transition:opacity 0.15s;
					"
					onmouseenter={(e)=>{if(sequence.length>1)(e.currentTarget as HTMLElement).style.opacity='1';}}
					onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.opacity=sequence.length<=1?'0.2':'0.5';}}
					title="Remove"
				>
					<svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
						<line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/>
					</svg>
				</button>
			</div>
		{/each}
	</div>

	<!-- Add timer button -->
	<div style="padding:12px 16px; border-top:1px solid {sidebarBorder};">
		<button
			onclick={addTimer}
			style="
				width:100%; padding:8px 12px; background:transparent; border:1px solid {sidebarBorder};
				color:{theme.textMuted}; font-size:12px; font-family:{sidebarFont}; cursor:pointer;
				border-radius:6px; letter-spacing:0.04em; transition:all 0.15s;
				display:flex; align-items:center; justify-content:center; gap:6px;
			"
			onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.borderColor=ca;(e.currentTarget as HTMLElement).style.color=ca;}}
			onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.borderColor=sidebarBorder;(e.currentTarget as HTMLElement).style.color=theme.textMuted;}}
		>
			<svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
				<line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/>
			</svg>
			Add timer
		</button>
	</div>
</aside>
{/if}

<!-- ─── Shared snippets ──────────────────────────────────────────────────── -->

{#snippet dotSwitcher(dimColor: string)}
	<div style="display:flex; flex-direction:column; align-items:center; gap:8px;">
		<div style="display:flex; align-items:center; gap:6px;">
			{#each themes as t, i}
				<button
					onclick={() => { themeIndex = i; }}
					style="height:7px; width:{themeIndex===i ? '22px' : '7px'}; border-radius:999px; background:{themeIndex===i ? t.dot : dimColor}; opacity:{themeIndex===i ? 1 : 0.45}; transition:all 0.25s ease; border:none; cursor:pointer; padding:0;"
					title={t.name}
				></button>
			{/each}
		</div>
		<span style="font-size:10px; letter-spacing:0.12em; text-transform:uppercase; color:{dimColor}; opacity:0.55;">{theme.name}</span>
	</div>
{/snippet}

{#snippet iconControls()}
	<div style="display:flex; align-items:center; gap:14px;">
		<button
			onclick={reset}
			style="width:44px; height:44px; border-radius:50%; background:{theme.surface}; border:none; cursor:pointer; display:flex; align-items:center; justify-content:center; transition:background 0.15s; color:{theme.textMuted};"
			onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surfaceHover;}}
			onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surface;}}
			title="Reset (R)"
		>
			<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
				<path d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8"/><path d="M3 3v5h5"/>
			</svg>
		</button>
		<button
			onclick={toggleRunning}
			style="width:56px; height:56px; border-radius:50%; background:{ca}; border:none; cursor:pointer; display:flex; align-items:center; justify-content:center; color:white; transition:background 0.15s;"
			onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=caHover;}}
			onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=ca;}}
			title="{running ? 'Pause' : 'Start'} (Space)"
		>
			{#if running}
				<svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><rect x="6" y="4" width="4" height="16" rx="1"/><rect x="14" y="4" width="4" height="16" rx="1"/></svg>
			{:else}
				<svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor" style="transform:translateX(1px)"><polygon points="5,3 19,12 5,21"/></svg>
			{/if}
		</button>
		<button
			onclick={nextTimer}
			style="width:44px; height:44px; border-radius:50%; background:{theme.surface}; border:none; cursor:pointer; display:flex; align-items:center; justify-content:center; transition:background 0.15s; color:{theme.textMuted};"
			onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surfaceHover;}}
			onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surface;}}
			title="Next timer (N)"
		>
			<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
				<polygon points="5,4 15,12 5,20" fill="currentColor" stroke="none"/><line x1="19" y1="5" x2="19" y2="19"/>
			</svg>
		</button>
		<button
			onclick={() => sidebarOpen = !sidebarOpen}
			style="width:44px; height:44px; border-radius:50%; background:{sidebarOpen ? theme.surfaceHover : theme.surface}; border:none; cursor:pointer; display:flex; align-items:center; justify-content:center; transition:background 0.15s; color:{sidebarOpen ? ca : theme.textMuted};"
			onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surfaceHover;}}
			onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=sidebarOpen?theme.surfaceHover:theme.surface;}}
			title="Timer sequence (S)"
		>
			<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
				<line x1="8" y1="6" x2="21" y2="6"/><line x1="8" y1="12" x2="21" y2="12"/><line x1="8" y1="18" x2="21" y2="18"/>
				<line x1="3" y1="6" x2="3.01" y2="6"/><line x1="3" y1="12" x2="3.01" y2="12"/><line x1="3" y1="18" x2="3.01" y2="18"/>
			</svg>
		</button>
	</div>
{/snippet}

<!-- ─── Hidden / minimal view ───────────────────────────────────────────── -->
{#if uiHidden}
<div style="min-height:100vh; background:{theme.bg}; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:20px; user-select:none;">
	{#if currentName}
		<span style="font-size:11px; letter-spacing:0.16em; text-transform:uppercase; color:{theme.textMuted}; opacity:0.6;">{currentName}</span>
	{/if}
	<div style="font-size:clamp(5rem,22vw,11rem); font-family:monospace; font-weight:200; letter-spacing:-0.02em; color:{ct}; line-height:1;">{displayTime}</div>
	<div style="width:min(320px,80vw); height:2px; background:{theme.track}; border-radius:999px; overflow:hidden;">
		<div style="height:100%; width:{barProgress*100}%; background:{ca}; border-radius:999px; transition:width 1s linear, background 0.4s;"></div>
	</div>
	{#if isOverflow}<span style="font-size:10px; color:{theme.overflow}; opacity:0.6; letter-spacing:0.12em; text-transform:uppercase;">overtime</span>{/if}
</div>

<!-- ─── VOID ─────────────────────────────────────────────────────────────── -->
{:else if theme.id === 'void'}
<div class="void-root" style="min-height:100vh; background:radial-gradient(ellipse at 50% 40%, #111 0%, #080808 80%); display:flex; flex-direction:column; align-items:center; justify-content:center; gap:40px; user-select:none; padding:24px;">

	<div style="display:flex; flex-direction:column; align-items:center; gap:20px; width:100%; max-width:320px;">
		{#if currentName}
			<span style="font-size:11px; letter-spacing:0.14em; text-transform:uppercase; color:{theme.textMuted}; opacity:0.7;">{currentName} <span style="opacity:0.4;">· {seqIndex + 1}/{sequence.length}</span></span>
		{/if}
		<div style="font-size:clamp(4rem,20vw,8rem); font-family:monospace; font-weight:200; letter-spacing:-0.02em; color:{ct}; line-height:1;">{displayTime}</div>

		<div style="width:100%; height:1px; background:{theme.track}; border-radius:999px; overflow:hidden;">
			<div style="height:100%; width:{barProgress*100}%; background:{ca}; border-radius:999px; transition:width 1s linear, background 0.4s;"></div>
		</div>
		{#if isOverflow}<span style="font-size:10px; color:{theme.overflow}; opacity:0.6; letter-spacing:0.12em; text-transform:uppercase; margin-top:-8px;">overtime</span>{/if}
	</div>

	{@render iconControls()}
	{@render dotSwitcher(theme.textDim)}

	<div style="display:flex; gap:16px; font-size:11px; color:{theme.textDim};">
		{#each [['Space','play/pause'],['R','reset'],['N','next'],['S','sequence'],['H','hide'],['Q','theme']] as [k,l]}
			<span><kbd style="font-family:monospace; background:#111; border:1px solid #222; color:#555; padding:2px 5px; border-radius:3px;">{k}</kbd> {l}</span>
		{/each}
	</div>
</div>

<!-- ─── TERMINAL ──────────────────────────────────────────────────────────── -->
{:else if theme.id === 'terminal'}
<div style="min-height:100vh; background:#080808; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:28px; user-select:none; font-family:'Courier New',Courier,monospace; padding:24px;">

	<!-- Window frame -->
	<div style="width:min(580px,96vw); border:1px solid #1e1e1e; border-radius:8px; overflow:hidden; box-shadow:0 24px 80px rgba(0,0,0,0.85);">
		<!-- Titlebar -->
		<div style="background:#141414; padding:10px 14px; display:flex; align-items:center; gap:6px; border-bottom:1px solid #1e1e1e;">
			<span style="width:11px;height:11px;border-radius:50%;background:#ff5f57;display:inline-block;flex-shrink:0;"></span>
			<span style="width:11px;height:11px;border-radius:50%;background:#febc2e;display:inline-block;flex-shrink:0;"></span>
			<span style="width:11px;height:11px;border-radius:50%;background:#28c840;display:inline-block;flex-shrink:0;"></span>
			<span style="flex:1;text-align:center;color:#3a3a3a;font-size:11px;font-family:'Courier New',monospace;">timer.sh — bash — 80×24</span>
		</div>
		<!-- Body with scanlines -->
		<div class="scanlines" style="position:relative;background:#000;padding:22px 26px;color:#00ff41;">
			<!-- Prompt line -->
			<div style="font-size:12px;margin-bottom:14px;line-height:1.5;">
				<span style="color:#00a82b;">user@localhost</span><span style="color:#555;">:</span><span style="color:#5555dd;">~</span><span style="color:#555;">$</span>
				<span style="color:#00ff41;"> ./timer.sh --duration {durationStr}</span>
			</div>
			<!-- Status + name -->
			<div style="font-size:11px;color:{isOverflow?'#ff3c3c':running?'#00ff41':'#006e1c'};letter-spacing:0.08em;margin-bottom:6px;">
				{isOverflow ? '⚠  OVERTIME' : running ? '▶  RUNNING' : '⏸  PAUSED'}
				{#if currentName}<span style="color:#006e1c; margin-left:12px;"># {currentName} ({seqIndex+1}/{sequence.length})</span>{/if}
			</div>
			<!-- Time display -->
			<div style="font-size:clamp(3rem,12vw,5.5rem);line-height:1;margin:8px 0;color:{isOverflow?'#ff3c3c':'#00ff41'};text-shadow:0 0 24px {isOverflow?'rgba(255,60,60,0.5)':'rgba(0,255,65,0.5)'};letter-spacing:0.04em;">
				{displayTime}<span class="cursor">█</span>
			</div>
			<!-- Text progress bar -->
			<div style="font-size:12px;color:{isOverflow?'#ff3c3c':'#006e1c'};margin:12px 0;letter-spacing:0.01em;">{termBar}</div>
			<!-- Bracketed buttons -->
			<div style="display:flex;gap:10px;flex-wrap:wrap;margin-top:16px;">
				<button onclick={toggleRunning}
					style="background:{running?'rgba(0,255,65,0.1)':'transparent'}; border:1px solid {running?'#00ff41':'#006e1c'}; color:{running?'#00ff41':ca}; font-family:inherit; font-size:12px; padding:5px 14px; cursor:pointer; border-radius:2px; letter-spacing:0.05em; transition:all 0.15s;"
					onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background='rgba(0,255,65,0.15)';}}
					onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=running?'rgba(0,255,65,0.1)':'transparent';}}
				>[{running?'pause':'start ▶'}]</button>
				<button onclick={reset}
					style="background:transparent; border:1px solid #006e1c; color:#00a82b; font-family:inherit; font-size:12px; padding:5px 14px; cursor:pointer; border-radius:2px; letter-spacing:0.05em; transition:all 0.15s;"
					onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background='rgba(0,255,65,0.08)';}}
					onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background='transparent';}}
				>[reset]</button>
				<button onclick={nextTimer}
					style="background:transparent; border:1px solid #006e1c; color:#00a82b; font-family:inherit; font-size:12px; padding:5px 14px; cursor:pointer; border-radius:2px; letter-spacing:0.05em; transition:all 0.15s;"
					onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background='rgba(0,255,65,0.08)';}}
					onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background='transparent';}}
				>[next ▶|]</button>
				<button onclick={() => sidebarOpen = !sidebarOpen}
					style="background:{sidebarOpen?'rgba(0,255,65,0.1)':'transparent'}; border:1px solid {sidebarOpen?'#00ff41':'#006e1c'}; color:{sidebarOpen?'#00ff41':'#00a82b'}; font-family:inherit; font-size:12px; padding:5px 14px; cursor:pointer; border-radius:2px; letter-spacing:0.05em; transition:all 0.15s;"
					onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background='rgba(0,255,65,0.08)';}}
					onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=sidebarOpen?'rgba(0,255,65,0.1)':'transparent';}}
				>[sequence]</button>
			</div>
		</div>
	</div>

	{@render dotSwitcher('#003c0f')}
	<div style="display:flex;gap:16px;font-size:11px;color:#003c0f;font-family:'Courier New',monospace;">
		{#each [['space','play/pause'],['r','reset'],['n','next'],['s','sequence'],['h','hide'],['q','theme']] as [k,l]}
			<span style="color:#006e1c;">{k} <span style="color:#003c0f;">→ {l}</span></span>
		{/each}
	</div>
</div>

<!-- ─── CHALK ──────────────────────────────────────────────────────────────── -->
{:else if theme.id === 'chalk'}
<div style="min-height:100vh; background:#f3ede2; display:flex; flex-direction:column; align-items:center; justify-content:center; padding:32px; user-select:none;">

	<!-- Big card -->
	<div style="background:#fffcf5; border-radius:24px; padding:clamp(32px,6vw,64px) clamp(28px,8vw,80px); box-shadow:0 2px 0 #e0d8cc, 0 8px 24px rgba(0,0,0,0.06), 0 32px 80px rgba(0,0,0,0.09); display:flex; flex-direction:column; align-items:center; gap:32px; width:min(520px,96vw);">

		<!-- Time display -->
		<div style="display:flex; flex-direction:column; align-items:center; gap:4px; width:100%;">
			{#if currentName}
				<span style="font-size:11px; letter-spacing:0.1em; text-transform:uppercase; color:{theme.textMuted}; margin-bottom:4px;">{currentName} <span style="opacity:0.5;">· {seqIndex+1}/{sequence.length}</span></span>
			{/if}
			<div style="font-size:clamp(5rem,22vw,9.5rem); font-family:Georgia,'Times New Roman',serif; font-weight:400; color:{isOverflow?'#b91c1c':'#1c1614'}; line-height:1; letter-spacing:-0.02em;">{displayTime}</div>
			{#if isOverflow}<span style="font-size:11px; color:#b91c1c; letter-spacing:0.1em; text-transform:uppercase; margin-top:4px;">overtime</span>{/if}
		</div>

		<!-- Dot progress -->
		<div style="display:flex; gap:6px; align-items:center; flex-wrap:wrap; justify-content:center;">
			{#each {length: DOT_COUNT} as _, i}
				<div style="width:8px; height:8px; border-radius:50%; background:{i < filledDots ? ca : theme.track}; transition:background 0.3s;"></div>
			{/each}
		</div>

		<!-- Physical buttons -->
		<div style="display:flex; align-items:center; gap:12px; flex-wrap:wrap; justify-content:center;">
			<button onclick={reset}
				style="padding:10px 20px; border-radius:10px; background:{theme.surface}; border:1px solid #d0c8bc; box-shadow:0 2px 0 #cac2b6; color:{theme.textMuted}; font-size:13px; cursor:pointer; letter-spacing:0.04em; transition:all 0.1s; display:flex;align-items:center;gap:6px;"
				onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surfaceHover;}}
				onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surface;}}
			>
				<svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8"/><path d="M3 3v5h5"/></svg>
				Reset
			</button>
			<button onclick={toggleRunning}
				style="padding:12px 28px; border-radius:12px; background:{ca}; border:1px solid {ca}; box-shadow:0 3px 0 {isOverflow?'#7f1d1d':'#1e3a8a'}; color:white; font-size:14px; font-weight:500; cursor:pointer; letter-spacing:0.04em; transition:all 0.1s; display:flex;align-items:center;gap:7px;"
				onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=caHover;}}
				onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=ca;}}
			>
				{#if running}
					<svg width="13" height="13" viewBox="0 0 24 24" fill="currentColor"><rect x="6" y="4" width="4" height="16" rx="1"/><rect x="14" y="4" width="4" height="16" rx="1"/></svg>
					Pause
				{:else}
					<svg width="13" height="13" viewBox="0 0 24 24" fill="currentColor" style="transform:translateX(1px)"><polygon points="5,3 19,12 5,21"/></svg>
					Start
				{/if}
			</button>
			<button onclick={nextTimer}
				style="padding:10px 20px; border-radius:10px; background:{theme.surface}; border:1px solid #d0c8bc; box-shadow:0 2px 0 #cac2b6; color:{theme.textMuted}; font-size:13px; cursor:pointer; letter-spacing:0.04em; transition:all 0.1s; display:flex;align-items:center;gap:6px;"
				onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surfaceHover;}}
				onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surface;}}
			>
				<svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
					<polygon points="4,4 14,12 4,20" fill="currentColor" stroke="none"/><line x1="18" y1="5" x2="18" y2="19"/>
				</svg>
				Next
			</button>
			<button onclick={() => sidebarOpen = !sidebarOpen}
				style="padding:10px 20px; border-radius:10px; background:{sidebarOpen?theme.surfaceHover:theme.surface}; border:1px solid {sidebarOpen?ca+'80':'#d0c8bc'}; box-shadow:0 2px 0 #cac2b6; color:{sidebarOpen?ca:theme.textMuted}; font-size:13px; cursor:pointer; letter-spacing:0.04em; transition:all 0.1s; display:flex;align-items:center;gap:6px;"
				onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surfaceHover;}}
				onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=sidebarOpen?theme.surfaceHover:theme.surface;}}
			>
				<svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
					<line x1="8" y1="6" x2="21" y2="6"/><line x1="8" y1="12" x2="21" y2="12"/><line x1="8" y1="18" x2="21" y2="18"/>
					<line x1="3" y1="6" x2="3.01" y2="6"/><line x1="3" y1="12" x2="3.01" y2="12"/><line x1="3" y1="18" x2="3.01" y2="18"/>
				</svg>
				Sequence
			</button>
		</div>
	</div>

	<div style="margin-top:28px; display:flex; flex-direction:column; align-items:center; gap:14px;">
		{@render dotSwitcher(theme.textDim)}
		<div style="display:flex; gap:14px; font-size:11px; color:{theme.textDim};">
			{#each [['Space','play/pause'],['R','reset'],['N','next'],['S','sequence'],['H','hide'],['Q','theme']] as [k,l]}
				<span><kbd style="font-family:monospace; background:white; border:1px solid #d8d0c8; color:#9c8e80; padding:2px 5px; border-radius:3px; box-shadow:0 1px 0 #d0c8bc;">{k}</kbd> {l}</span>
			{/each}
		</div>
	</div>
</div>

<!-- ─── DUSK ───────────────────────────────────────────────────────────────── -->
{:else if theme.id === 'dusk'}
<div style="min-height:100vh; background:#07051a; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:36px; user-select:none; padding:24px; position:relative; overflow:hidden;">

	<!-- Aurora blobs -->
	<div style="position:absolute; inset:0; pointer-events:none; overflow:hidden;">
		<div style="position:absolute; width:70vw; height:60vh; top:-15%; left:-15%; background:radial-gradient(circle, rgba(168,85,247,0.45) 0%, transparent 65%); filter:blur(70px); animation:aurora1 9s ease-in-out infinite;"></div>
		<div style="position:absolute; width:60vw; height:55vh; bottom:-10%; right:-10%; background:radial-gradient(circle, rgba(251,146,60,0.3) 0%, transparent 65%); filter:blur(70px); animation:aurora2 11s ease-in-out infinite;"></div>
		<div style="position:absolute; width:50vw; height:50vh; top:50%; left:50%; transform:translate(-50%,-50%); background:radial-gradient(circle, rgba(139,92,246,0.25) 0%, transparent 65%); filter:blur(80px); animation:aurora3 14s ease-in-out infinite;"></div>
	</div>

	<!-- Glass card -->
	<div style="position:relative; z-index:1; background:rgba(255,255,255,0.05); backdrop-filter:blur(28px); -webkit-backdrop-filter:blur(28px); border:1px solid rgba(255,255,255,0.1); border-radius:28px; padding:clamp(36px,6vw,56px) clamp(32px,7vw,72px); box-shadow:0 8px 48px rgba(0,0,0,0.5), inset 0 1px 0 rgba(255,255,255,0.1); display:flex; flex-direction:column; align-items:center; gap:28px; width:min(480px,94vw);">

		{#if currentName}
			<span style="font-size:11px; letter-spacing:0.14em; text-transform:uppercase; color:rgba(168,85,247,0.5);">{currentName} <span style="opacity:0.5;">· {seqIndex+1}/{sequence.length}</span></span>
		{/if}
		<div style="font-size:clamp(4rem,18vw,7.5rem); font-family:monospace; font-weight:200; letter-spacing:-0.02em; color:{ct}; line-height:1; text-shadow:0 0 40px {ca}80, 0 0 80px {ca}40;">{displayTime}</div>

		<!-- Glowing bar -->
		<div style="width:100%; height:2px; background:{theme.track}; border-radius:999px; overflow:hidden; position:relative;">
			<div style="height:100%; width:{barProgress*100}%; background:{ca}; border-radius:999px; box-shadow:0 0 8px {ca}, 0 0 18px {ca}80; transition:width 1s linear, background 0.4s;"></div>
		</div>
		{#if isOverflow}<span style="font-size:10px; color:{theme.overflow}; opacity:0.6; letter-spacing:0.12em; text-transform:uppercase; margin-top:-8px;">overtime</span>{/if}

		{@render iconControls()}
	</div>

	<div style="position:relative; z-index:1; display:flex; flex-direction:column; align-items:center; gap:14px;">
		{@render dotSwitcher(theme.textDim)}
		<div style="display:flex; gap:16px; font-size:11px; color:{theme.textDim};">
			{#each [['Space','play/pause'],['R','reset'],['N','next'],['S','sequence'],['H','hide'],['Q','theme']] as [k,l]}
				<span><kbd style="font-family:monospace; background:rgba(255,255,255,0.04); border:1px solid rgba(255,255,255,0.08); color:rgba(168,85,247,0.5); padding:2px 5px; border-radius:3px;">{k}</kbd> {l}</span>
			{/each}
		</div>
	</div>
</div>

<!-- ─── ICE ────────────────────────────────────────────────────────────────── -->
{:else if theme.id === 'ice'}
<div style="min-height:100vh; background:#010c18; background-image:linear-gradient(rgba(14,165,233,0.04) 1px, transparent 1px),linear-gradient(90deg, rgba(14,165,233,0.04) 1px, transparent 1px); background-size:36px 36px; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:36px; user-select:none; padding:24px; font-family:'Courier New',Courier,monospace;">

	<div style="display:flex; flex-direction:column; align-items:center; gap:28px; width:100%; max-width:500px;">

		{#if currentName}
			<span style="font-size:10px; letter-spacing:0.14em; text-transform:uppercase; color:{theme.textMuted};">{currentName} <span style="opacity:0.5;">· {seqIndex+1}/{sequence.length}</span></span>
		{/if}

		<!-- Segment digit display -->
		<div style="display:flex; align-items:center; gap:4px;">
			{#each displayTime.split('') as char}
				{#if /\d/.test(char)}
					<span style="
						font-size:clamp(2.8rem,13vw,6rem); font-weight:200; line-height:1;
						color:{isOverflow?theme.overflow:theme.text};
						border:1px solid {isOverflow?theme.overflow+'60':theme.textMuted+'80'};
						border-radius:5px;
						padding:0.04em 0.12em;
						background:rgba(14,165,233,0.03);
						display:inline-block;
						text-shadow:{isOverflow?'0 0 16px '+theme.overflow:'0 0 16px rgba(14,165,233,0.3)'};
					">{char}</span>
				{:else if char === ':'}
					<span style="font-size:clamp(2.8rem,13vw,6rem); font-weight:200; color:{theme.textMuted}; line-height:1; padding:0 2px; margin-top:-4px;">:</span>
				{:else}
					<span style="font-size:clamp(2.8rem,13vw,6rem); font-weight:200; color:{theme.overflow}; line-height:1; padding:0 2px;">{char}</span>
				{/if}
			{/each}
		</div>

		<!-- Block progress -->
		<div style="display:flex; align-items:flex-end; gap:3px; height:24px;">
			{#each {length: BLOCK_COUNT} as _, i}
				{@const filled = i < filledBlocks}
				{@const heightPct = 40 + Math.sin((i / (BLOCK_COUNT - 1)) * Math.PI) * 60}
				<div style="
					width:4px;
					height:{heightPct}%;
					background:{filled ? ca : theme.track};
					border-radius:2px;
					box-shadow:{filled ? '0 0 6px '+ca+'99' : 'none'};
					transition:background 0.4s;
				"></div>
			{/each}
		</div>
		{#if isOverflow}<span style="font-size:10px; color:{theme.overflow}; letter-spacing:0.12em; text-transform:uppercase; margin-top:-8px; opacity:0.65;">overtime</span>{/if}

		<!-- Technical labels -->
		<div style="display:flex; gap:24px; font-size:10px; color:{theme.textMuted}; letter-spacing:0.1em; text-transform:uppercase;">
			<span>ELAPSED <span style="color:{ca};">{Math.round((1-barProgress)*100)}%</span></span>
			<span>|</span>
			<span>REMAINING <span style="color:{ca};">{Math.round(barProgress*100)}%</span></span>
		</div>
	</div>

	{@render iconControls()}
	{@render dotSwitcher(theme.textDim)}

	<div style="display:flex; gap:16px; font-size:11px; color:{theme.textDim}; font-family:inherit;">
		{#each [['Space','play/pause'],['R','reset'],['N','next'],['S','sequence'],['H','hide'],['Q','theme']] as [k,l]}
			<span><kbd style="font-family:inherit; background:rgba(14,165,233,0.04); border:1px solid rgba(14,165,233,0.12); color:{theme.textMuted}; padding:2px 5px; border-radius:3px;">{k}</kbd> {l}</span>
		{/each}
	</div>
</div>
{/if}

<style>
	@keyframes blink {
		0%, 49% { opacity: 1; }
		50%, 100% { opacity: 0; }
	}
	:global(.cursor) {
		animation: blink 1.1s step-start infinite;
	}
	@keyframes aurora1 {
		0%, 100% { transform: translate(0%, 0%) scale(1); }
		33% { transform: translate(6%, -10%) scale(1.14); }
		66% { transform: translate(-8%, 6%) scale(0.9); }
	}
	@keyframes aurora2 {
		0%, 100% { transform: translate(0%, 0%) scale(1); }
		40% { transform: translate(-9%, 7%) scale(0.94); }
		80% { transform: translate(5%, -5%) scale(1.1); }
	}
	@keyframes aurora3 {
		0%, 100% { transform: translate(-50%, -50%) scale(1); }
		50% { transform: translate(-50%, -50%) scale(1.18); }
	}
	:global(.scanlines::after) {
		content: '';
		position: absolute;
		inset: 0;
		background: repeating-linear-gradient(
			to bottom,
			transparent 0px,
			transparent 3px,
			rgba(0, 0, 0, 0.13) 3px,
			rgba(0, 0, 0, 0.13) 4px
		);
		pointer-events: none;
		z-index: 2;
	}
</style>
