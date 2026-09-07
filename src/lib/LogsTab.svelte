<script>
	export let logs = [];
</script>

<p class="devlab-kicker">SYSTEM LOG</p>
<h1 class="text-lg font-bold">Logs</h1>
<div class="min-h-0 flex-1 overflow-y-auto rounded bg-slate-950 p-2 font-mono text-xs text-slate-300">
	{#if logs.length == 0}
		<p class="text-slate-500">Esperando inicialización...</p>
	{:else}
		{#each logs as entry}
			<p>{entry}</p>
		{/each}
	{/if}
</div>
