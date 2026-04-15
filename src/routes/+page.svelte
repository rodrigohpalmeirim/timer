<script lang="ts">
	import { onDestroy } from 'svelte';

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

	// ─── Timer logic ───────────────────────────────────────────────────────────

	let totalSeconds = $state(300);
	let remaining = $state(300);
	let running = $state(false);
	let overflowSeconds = $state(0);
	let editing = $state(false);
	let editHours = $state(0);
	let editMinutes = $state(5);
	let editSeconds = $state(0);

	let intervalId: ReturnType<typeof setInterval> | null = null;

	function startInterval() {
		if (intervalId) return;
		intervalId = setInterval(() => {
			if (remaining > 0) remaining -= 1;
			else overflowSeconds += 1;
		}, 1000);
	}
	function stopInterval() { if (intervalId) { clearInterval(intervalId); intervalId = null; } }
	function start() { if (editing) commitEdit(); running = true; startInterval(); }
	function pause() { running = false; stopInterval(); }
	function reset() { pause(); remaining = totalSeconds; overflowSeconds = 0; }
	function toggleRunning() { if (running) pause(); else start(); }

	function openEdit() {
		pause();
		editHours = Math.floor(totalSeconds / 3600);
		editMinutes = Math.floor((totalSeconds % 3600) / 60);
		editSeconds = totalSeconds % 60;
		editing = true;
	}
	function commitEdit() {
		totalSeconds = Math.max(1, editHours * 3600 + editMinutes * 60 + editSeconds);
		remaining = totalSeconds;
		overflowSeconds = 0;
		editing = false;
	}
	function cancelEdit() { editing = false; }
	function pad(n: number) { return String(n).padStart(2, '0'); }

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

	// Contextual accent (flips to overflow color)
	let ca = $derived(isOverflow ? theme.overflow : theme.accent);
	let caHover = $derived(isOverflow ? theme.overflowHover : theme.accentHover);
	let ct = $derived(isOverflow ? theme.overflow : theme.text);

	// Progress variants
	const DOT_COUNT = 24;
	let filledDots = $derived(Math.round(barProgress * DOT_COUNT));

	const BLOCK_COUNT = 30;
	let filledBlocks = $derived(Math.round(barProgress * BLOCK_COUNT));

	let termBar = $derived.by(() => {
		const w = 22, f = Math.round(barProgress * w);
		return `[${'█'.repeat(f)}${'░'.repeat(w - f)}] ${Math.round(barProgress * 100)}%`;
	});

	let durationStr = $derived.by(() => {
		const h = Math.floor(totalSeconds / 3600), m = Math.floor((totalSeconds % 3600) / 60), s = totalSeconds % 60;
		return h > 0 ? `${pad(h)}:${pad(m)}:${pad(s)}` : `${pad(m)}:${pad(s)}`;
	});

	function clampField(field: 'h' | 'm' | 's') {
		if (field === 'h') editHours = Math.max(0, Math.min(99, editHours));
		if (field === 'm') editMinutes = Math.max(0, Math.min(59, editMinutes));
		if (field === 's') editSeconds = Math.max(0, Math.min(59, editSeconds));
	}

	function handleKeydown(e: KeyboardEvent) {
		if (editing) {
			if (e.key === 'Enter') { e.preventDefault(); commitEdit(); }
			if (e.key === 'Escape') { e.preventDefault(); cancelEdit(); }
			return;
		}
		if (e.key === ' ' || e.key === 'k') { e.preventDefault(); toggleRunning(); }
		if (e.key === 'r') { e.preventDefault(); reset(); }
		if (e.key === 'e') { e.preventDefault(); openEdit(); }
		if (e.key === 'q') { e.preventDefault(); cycleTheme(); }
	}

	onDestroy(() => stopInterval());
</script>

<svelte:window onkeydown={handleKeydown} />

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
			onclick={openEdit}
			style="width:44px; height:44px; border-radius:50%; background:{theme.surface}; border:none; cursor:pointer; display:flex; align-items:center; justify-content:center; transition:background 0.15s; color:{theme.textMuted};"
			onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surfaceHover;}}
			onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surface;}}
			title="Edit (E)"
		>
			<svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
				<path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/>
				<path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4Z"/>
			</svg>
		</button>
	</div>
{/snippet}

<!-- ─── VOID ─────────────────────────────────────────────────────────────── -->
{#if theme.id === 'void'}
<div class="void-root" style="min-height:100vh; background:radial-gradient(ellipse at 50% 40%, #111 0%, #080808 80%); display:flex; flex-direction:column; align-items:center; justify-content:center; gap:40px; user-select:none; padding:24px;">

	<div style="display:flex; flex-direction:column; align-items:center; gap:20px; width:100%; max-width:320px;">
		{#if editing}
			<form onsubmit={(e)=>{e.preventDefault();commitEdit();}} style="display:flex; align-items:center; gap:8px;">
				{#each [{v: editHours, f:'h', max:99},{v:editMinutes,f:'m',max:59},{v:editSeconds,f:'s',max:59}] as field, i}
					{#if i > 0}<span style="font-size:3rem; color:#333; font-family:monospace; font-weight:300;">:</span>{/if}
					<input type="number" min="0" max={field.max} value={field.v}
						oninput={(e)=>{ const v=parseInt((e.target as HTMLInputElement).value)||0; if(field.f==='h')editHours=v; else if(field.f==='m')editMinutes=v; else editSeconds=v; clampField(field.f as 'h'|'m'|'s'); }}
						style="width:56px; background:transparent; border:none; border-bottom:2px solid #333; color:white; font-size:3rem; font-family:monospace; font-weight:300; text-align:center; outline:none; padding-bottom:2px;"
						onfocus={(e)=>{(e.target as HTMLInputElement).style.borderBottomColor='#6366f1';}}
						onblur={(e)=>{(e.target as HTMLInputElement).style.borderBottomColor='#333';}}
						placeholder="00"
					/>
				{/each}
			</form>
			<span style="font-size:11px; color:#333; letter-spacing:0.1em; text-transform:uppercase;">Enter to set · Esc to cancel</span>
		{:else}
			<button onclick={openEdit} style="font-size:clamp(4rem,20vw,8rem); font-family:monospace; font-weight:200; letter-spacing:-0.02em; color:{ct}; background:none; border:none; cursor:pointer; line-height:1; opacity:1; transition:opacity 0.15s; padding:0;"
				onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.opacity='0.65';}}
				onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.opacity='1';}}
			>{displayTime}</button>
		{/if}

		<div style="width:100%; height:1px; background:{theme.track}; border-radius:999px; overflow:hidden;">
			<div style="height:100%; width:{barProgress*100}%; background:{ca}; border-radius:999px; transition:width 1s linear, background 0.4s;"></div>
		</div>
		{#if isOverflow}<span style="font-size:10px; color:{theme.overflow}; opacity:0.6; letter-spacing:0.12em; text-transform:uppercase; margin-top:-8px;">overtime</span>{/if}
	</div>

	{@render iconControls()}
	{@render dotSwitcher(theme.textDim)}

	<div style="display:flex; gap:16px; font-size:11px; color:{theme.textDim};">
		{#each [['Space','play/pause'],['R','reset'],['E','edit'],['Q','theme']] as [k,l]}
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
			<!-- Status -->
			<div style="font-size:11px;color:{isOverflow?'#ff3c3c':running?'#00ff41':'#006e1c'};letter-spacing:0.08em;margin-bottom:6px;">
				{isOverflow ? '⚠  OVERTIME' : running ? '▶  RUNNING' : '⏸  PAUSED'}
			</div>
			<!-- Time display -->
			{#if editing}
				<form onsubmit={(e)=>{e.preventDefault();commitEdit();}} style="display:flex; align-items:baseline; gap:6px; margin:10px 0;">
					<span style="color:#006e1c; font-size:12px;">$ set</span>
					{#each [{v:editHours,f:'h',max:99},{v:editMinutes,f:'m',max:59},{v:editSeconds,f:'s',max:59}] as field, i}
						{#if i>0}<span style="font-size:clamp(1.8rem,7vw,2.8rem);color:#006e1c;">:</span>{/if}
						<input type="number" min="0" max={field.max} value={field.v}
							oninput={(e)=>{ const v=parseInt((e.target as HTMLInputElement).value)||0; if(field.f==='h')editHours=v; else if(field.f==='m')editMinutes=v; else editSeconds=v; clampField(field.f as 'h'|'m'|'s'); }}
							style="width:52px; background:rgba(0,255,65,0.08); border:1px solid #006e1c; color:#00ff41; font-size:clamp(1.8rem,7vw,2.8rem); font-family:inherit; text-align:center; outline:none; border-radius:2px; padding:2px;"
							onfocus={(e)=>{(e.target as HTMLInputElement).style.borderColor='#00ff41';}}
							onblur={(e)=>{(e.target as HTMLInputElement).style.borderColor='#006e1c';}}
						/>
					{/each}
				</form>
				<div style="font-size:11px;color:#006e1c;margin-top:4px;">↵ enter to set &nbsp;·&nbsp; esc to cancel</div>
			{:else}
				<div style="font-size:clamp(3rem,12vw,5.5rem);line-height:1;margin:8px 0;color:{isOverflow?'#ff3c3c':'#00ff41'};text-shadow:0 0 24px {isOverflow?'rgba(255,60,60,0.5)':'rgba(0,255,65,0.5)'};letter-spacing:0.04em;">
					<button onclick={openEdit} style="background:none;border:none;color:inherit;font:inherit;cursor:pointer;padding:0;text-shadow:inherit;letter-spacing:inherit;">{displayTime}<span class="cursor">█</span></button>
				</div>
			{/if}
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
				<button onclick={openEdit}
					style="background:transparent; border:1px solid #006e1c; color:#00a82b; font-family:inherit; font-size:12px; padding:5px 14px; cursor:pointer; border-radius:2px; letter-spacing:0.05em; transition:all 0.15s;"
					onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background='rgba(0,255,65,0.08)';}}
					onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background='transparent';}}
				>[edit]</button>
			</div>
		</div>
	</div>

	{@render dotSwitcher('#003c0f')}
	<div style="display:flex;gap:16px;font-size:11px;color:#003c0f;font-family:'Courier New',monospace;">
		{#each [['space','play/pause'],['r','reset'],['e','edit'],['q','theme']] as [k,l]}
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
			{#if editing}
				<form onsubmit={(e)=>{e.preventDefault();commitEdit();}} style="display:flex; flex-direction:column; align-items:center; gap:16px;">
					<div style="display:flex; align-items:center; gap:8px;">
						{#each [{v:editHours,f:'h',max:99,lbl:'HH'},{v:editMinutes,f:'m',max:59,lbl:'MM'},{v:editSeconds,f:'s',max:59,lbl:'SS'}] as field, i}
							{#if i>0}<span style="font-size:3rem;color:#c0b8ac;font-family:Georgia,serif;margin-bottom:18px;">:</span>{/if}
							<div style="display:flex; flex-direction:column; align-items:center; gap:6px;">
								<input type="number" min="0" max={field.max} value={field.v}
									oninput={(e)=>{ const v=parseInt((e.target as HTMLInputElement).value)||0; if(field.f==='h')editHours=v; else if(field.f==='m')editMinutes=v; else editSeconds=v; clampField(field.f as 'h'|'m'|'s'); }}
									style="width:70px; background:#f5f0e8; border:2px solid #d8d0c4; border-radius:8px; color:#1c1614; font-size:2.8rem; font-family:Georgia,serif; text-align:center; outline:none; padding:6px 4px; transition:border-color 0.15s;"
									onfocus={(e)=>{(e.target as HTMLInputElement).style.borderColor='#1d4ed8';}}
									onblur={(e)=>{(e.target as HTMLInputElement).style.borderColor='#d8d0c4';}}
								/>
								<span style="font-size:10px; letter-spacing:0.12em; color:#c0b8ac; text-transform:uppercase;">{field.lbl}</span>
							</div>
						{/each}
					</div>
					<span style="font-size:11px; color:#c0b8ac; letter-spacing:0.08em;">Enter to set · Esc to cancel</span>
				</form>
			{:else}
				<button onclick={openEdit}
					style="font-size:clamp(5rem,22vw,9.5rem); font-family:Georgia,'Times New Roman',serif; font-weight:400; color:{isOverflow?'#b91c1c':'#1c1614'}; background:none; border:none; cursor:pointer; line-height:1; letter-spacing:-0.02em; transition:opacity 0.15s; padding:0;"
					onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.opacity='0.6';}}
					onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.opacity='1';}}
				>{displayTime}</button>
				{#if isOverflow}<span style="font-size:11px; color:#b91c1c; letter-spacing:0.1em; text-transform:uppercase; margin-top:4px;">overtime</span>{/if}
			{/if}
		</div>

		<!-- Dot progress -->
		<div style="display:flex; gap:6px; align-items:center; flex-wrap:wrap; justify-content:center;">
			{#each {length: DOT_COUNT} as _, i}
				<div style="width:8px; height:8px; border-radius:50%; background:{i < filledDots ? ca : theme.track}; transition:background 0.3s;"></div>
			{/each}
		</div>

		<!-- Physical buttons -->
		<div style="display:flex; align-items:center; gap:12px;">
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
			<button onclick={openEdit}
				style="padding:10px 20px; border-radius:10px; background:{theme.surface}; border:1px solid #d0c8bc; box-shadow:0 2px 0 #cac2b6; color:{theme.textMuted}; font-size:13px; cursor:pointer; letter-spacing:0.04em; transition:all 0.1s; display:flex;align-items:center;gap:6px;"
				onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surfaceHover;}}
				onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.background=theme.surface;}}
			>
				<svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/><path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4Z"/></svg>
				Edit
			</button>
		</div>
	</div>

	<div style="margin-top:28px; display:flex; flex-direction:column; align-items:center; gap:14px;">
		{@render dotSwitcher(theme.textDim)}
		<div style="display:flex; gap:14px; font-size:11px; color:{theme.textDim};">
			{#each [['Space','play/pause'],['R','reset'],['E','edit'],['Q','theme']] as [k,l]}
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

		{#if editing}
			<form onsubmit={(e)=>{e.preventDefault();commitEdit();}} style="display:flex; align-items:center; gap:8px;">
				{#each [{v:editHours,f:'h',max:99},{v:editMinutes,f:'m',max:59},{v:editSeconds,f:'s',max:59}] as field, i}
					{#if i>0}<span style="font-size:3rem; color:rgba(168,85,247,0.4); font-family:monospace; font-weight:200;">:</span>{/if}
					<input type="number" min="0" max={field.max} value={field.v}
						oninput={(e)=>{ const v=parseInt((e.target as HTMLInputElement).value)||0; if(field.f==='h')editHours=v; else if(field.f==='m')editMinutes=v; else editSeconds=v; clampField(field.f as 'h'|'m'|'s'); }}
						style="width:60px; background:rgba(168,85,247,0.1); border:1px solid rgba(168,85,247,0.3); border-radius:8px; color:#f0e8ff; font-size:3rem; font-family:monospace; font-weight:200; text-align:center; outline:none; padding:4px;"
						onfocus={(e)=>{(e.target as HTMLInputElement).style.borderColor='rgba(168,85,247,0.8)'; (e.target as HTMLInputElement).style.boxShadow='0 0 12px rgba(168,85,247,0.3)';}}
						onblur={(e)=>{(e.target as HTMLInputElement).style.borderColor='rgba(168,85,247,0.3)'; (e.target as HTMLInputElement).style.boxShadow='none';}}
					/>
				{/each}
			</form>
			<span style="font-size:11px; color:rgba(168,85,247,0.4); letter-spacing:0.1em; text-transform:uppercase;">Enter to set · Esc to cancel</span>
		{:else}
			<button onclick={openEdit}
				style="font-size:clamp(4rem,18vw,7.5rem); font-family:monospace; font-weight:200; letter-spacing:-0.02em; color:{ct}; background:none; border:none; cursor:pointer; line-height:1; padding:0; text-shadow:0 0 40px {ca}80, 0 0 80px {ca}40; transition:opacity 0.15s;"
				onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.opacity='0.7';}}
				onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.opacity='1';}}
			>{displayTime}</button>
		{/if}

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
			{#each [['Space','play/pause'],['R','reset'],['E','edit'],['Q','theme']] as [k,l]}
				<span><kbd style="font-family:monospace; background:rgba(255,255,255,0.04); border:1px solid rgba(255,255,255,0.08); color:rgba(168,85,247,0.5); padding:2px 5px; border-radius:3px;">{k}</kbd> {l}</span>
			{/each}
		</div>
	</div>
</div>

<!-- ─── ICE ────────────────────────────────────────────────────────────────── -->
{:else if theme.id === 'ice'}
<div style="min-height:100vh; background:#010c18; background-image:linear-gradient(rgba(14,165,233,0.04) 1px, transparent 1px),linear-gradient(90deg, rgba(14,165,233,0.04) 1px, transparent 1px); background-size:36px 36px; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:36px; user-select:none; padding:24px; font-family:'Courier New',Courier,monospace;">

	<div style="display:flex; flex-direction:column; align-items:center; gap:28px; width:100%; max-width:500px;">

		<!-- Segment digit display -->
		{#if editing}
			<form onsubmit={(e)=>{e.preventDefault();commitEdit();}} style="display:flex; align-items:center; gap:6px;">
				{#each [{v:editHours,f:'h',max:99},{v:editMinutes,f:'m',max:59},{v:editSeconds,f:'s',max:59}] as field, i}
					{#if i>0}<span style="font-size:clamp(2rem,8vw,3.5rem); color:#1e6080; font-weight:200; margin:0 2px;">:</span>{/if}
					<input type="number" min="0" max={field.max} value={field.v}
						oninput={(e)=>{ const v=parseInt((e.target as HTMLInputElement).value)||0; if(field.f==='h')editHours=v; else if(field.f==='m')editMinutes=v; else editSeconds=v; clampField(field.f as 'h'|'m'|'s'); }}
						style="width:64px; background:rgba(14,165,233,0.06); border:1px solid #1e6080; border-radius:4px; color:#c8e8ff; font-size:clamp(2rem,8vw,3.5rem); font-family:inherit; font-weight:200; text-align:center; outline:none; padding:4px 2px;"
						onfocus={(e)=>{(e.target as HTMLInputElement).style.borderColor='#0ea5e9'; (e.target as HTMLInputElement).style.boxShadow='0 0 8px rgba(14,165,233,0.3)';}}
						onblur={(e)=>{(e.target as HTMLInputElement).style.borderColor='#1e6080'; (e.target as HTMLInputElement).style.boxShadow='none';}}
					/>
				{/each}
			</form>
			<span style="font-size:11px; color:#1e6080; letter-spacing:0.1em; text-transform:uppercase; margin-top:-8px;">Enter to set · Esc to cancel</span>
		{:else}
			<div style="display:flex; align-items:center; gap:4px; cursor:pointer;" onclick={openEdit}
				onmouseenter={(e)=>{(e.currentTarget as HTMLElement).style.opacity='0.7';}}
				onmouseleave={(e)=>{(e.currentTarget as HTMLElement).style.opacity='1';}}
				role="button" tabindex="0" onkeydown={(e)=>{if(e.key==='Enter')openEdit();}} title="Click or E to edit"
			>
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
		{/if}

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
		{#each [['Space','play/pause'],['R','reset'],['E','edit'],['Q','theme']] as [k,l]}
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
