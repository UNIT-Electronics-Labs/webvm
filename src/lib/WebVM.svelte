<script>
	import { onMount, tick } from 'svelte';
	import { get } from 'svelte/store';
	import Nav from 'labs/packages/global-navbar/src/Nav.svelte';
	import SideBar from '$lib/SideBar.svelte';
	import '$lib/global.css';
	import '@xterm/xterm/css/xterm.css'
	import '@fortawesome/fontawesome-free/css/all.min.css'
	import { networkInterface, startLogin } from '$lib/network.js'
	import { cpuActivity, diskActivity, cpuPercentage, diskLatency } from '$lib/activities.js'
	import { errorMessage, unexpectedErrorMessage } from '$lib/messages.js'
	import { displayConfig, handleToolImpl } from '$lib/anthropic.js'
	import { tryPlausible } from '$lib/plausible.js'

	export let configObj = null;
	export let processCallback = null;
	export let cacheId = null;
	export let cpuActivityEvents = [];
	export let diskLatencies = [];
	export let activityEventsInterval = 0;

	var term = null;
	var cx = null;
	var dataDevice = null;
	var fitAddon = null;
	var cxReadFunc = null;
	var blockCache = null;
	var processCount = 0;
	var curVT = 0;
	var sideBarPinned = false;
	var startupLogs = [];
	var editorOpen = false;
	var editorContent = '';
	var editorStatus = '';
	var editorBusy = false;
	var directoryOpen = false;
	var editorDirectories = [];
	var editorDirectory = '';
	var editorStageCounter = 0;
	const examplesRoot = '/home/user/ch552/examples';
	function writeData(buf, vt)
	{
		if(vt != 1)
			return;
		term.write(new Uint8Array(buf));
	}
	function readData(str)
	{
		if(cxReadFunc == null)
			return;
		for(var i=0;i<str.length;i++)
			cxReadFunc(str.charCodeAt(i));
	}
	function printMessage(msg)
	{
		for(var i=0;i<msg.length;i++)
			term.write(msg[i] + "\n");
	}
	function appendLog(message)
	{
		startupLogs = [...startupLogs, message.replace(/\x1b\[[0-9;]*m/g, '')];
	}
	function linuxOptions(cwd = editorDirectory || examplesRoot)
	{
		return {
			...configObj.opts,
			cwd,
			uid: 1000,
			gid: 1000
		};
	}
	function selectedFile()
	{
		return `${editorDirectory}/main.c`;
	}
	async function runCapture(fileName, args, forwardOutput = true)
	{
		var output = '';
		const decoder = new TextDecoder();
		cx.setCustomConsole((buf, vt) => {
			if(vt == 1)
			{
				if(forwardOutput)
					writeData(buf, vt);
				output += decoder.decode(new Uint8Array(buf), {stream:true});
			}
		}, term.cols, term.rows);
		var result;
		try
		{
			result = await cx.run(fileName, args, linuxOptions());
		}
		finally
		{
			cxReadFunc = cx.setCustomConsole(writeData, term.cols, term.rows);
		}
		return {status: result.status, output};
	}
	async function chooseDirectory()
	{
		if(cx == null || editorBusy)
			return;
		editorBusy = true;
		editorStatus = 'Buscando proyectos...';
		try
		{
			const result = await runCapture('/usr/bin/find', [examplesRoot, '-mindepth', '2', '-type', 'f', '-name', 'main.c', '-printf', '%h\\n']);
			if(result.status != 0)
				throw new Error('No se pudieron leer los proyectos');
			editorDirectories = result.output.split(/\r?\n/).map(path => path.trim()).filter(Boolean);
			if(editorDirectories.length == 0)
				throw new Error('No se encontraron directorios con main.c');
			directoryOpen = true;
			editorStatus = 'Elige un directorio';
		}
		catch(e)
		{
			editorStatus = e.toString();
		}
		finally
		{
			editorBusy = false;
		}
	}
	async function selectDirectory(directory)
	{
		editorDirectory = directory;
		directoryOpen = false;
		await openEditor();
	}
	async function openEditor()
	{
		if(cx == null || editorBusy)
			return;
		editorBusy = true;
		editorStatus = 'Abriendo main.c...';
		try
		{
			const result = await runCapture('/bin/cat', [selectedFile()]);
			if(result.status != 0)
				throw new Error('No se pudo abrir main.c');
			editorContent = result.output;
			editorOpen = true;
			editorStatus = 'Archivo cargado';
		}
		catch(e)
		{
			editorStatus = e.toString();
		}
		finally
		{
			editorBusy = false;
		}
	}
	async function saveEditorFile()
	{
		if(cx == null)
			return false;
		editorStatus = 'Guardando...';
		try
		{
			const stageFile = `/main-${Date.now()}-${editorStageCounter++}.c`;
			await dataDevice.writeFile(stageFile, editorContent);
			const result = await cx.run('/bin/cp', [`/data${stageFile}`, selectedFile()], linuxOptions());
			if(result.status != 0)
				throw new Error('No se pudo guardar main.c');
			editorStatus = 'Guardado en Linux';
			return true;
		}
		catch(e)
		{
			editorStatus = e.toString();
			return false;
		}
	}
	async function saveEditor()
	{
		if(cx == null || editorBusy)
			return;
		editorBusy = true;
		await saveEditorFile();
		editorBusy = false;
	}
	async function compileEditor()
	{
		if(cx == null || editorBusy)
			return;
		editorBusy = true;
		editorStatus = 'Compilando BIN... revisa la terminal';
		try
		{
			if(!await saveEditorFile())
				return;
			const result = await cx.run('/usr/bin/make', ['bin'], linuxOptions());
			editorStatus = result.status == 0 ? 'BIN generado correctamente' : `Error de compilación (${result.status})`;
		}
		catch(e)
		{
			editorStatus = e.toString();
		}
		finally
		{
			editorBusy = false;
		}
	}
	async function downloadBin()
	{
		if(cx == null || editorBusy)
			return;
		editorBusy = true;
		editorStatus = 'Preparando descarga...';
		try
		{
			const result = await runCapture('/usr/bin/base64', ['-w', '0', `${editorDirectory}/build/main.bin`], false);
			if(result.status != 0)
				throw new Error('No existe build/main.bin; compila primero');
			const encoded = result.output.replace(/\s/g, '');
			const raw = atob(encoded);
			const bytes = new Uint8Array(raw.length);
			for(let i = 0; i < raw.length; i++)
				bytes[i] = raw.charCodeAt(i);
			const url = URL.createObjectURL(new Blob([bytes], {type: 'application/octet-stream'}));
			const link = document.createElement('a');
			link.href = url;
			link.download = 'main.bin';
			link.click();
			URL.revokeObjectURL(url);
			editorStatus = 'main.bin descargado';
		}
		catch(e)
		{
			editorStatus = e.toString();
		}
		finally
		{
			editorBusy = false;
		}
	}
	function expireEvents(list, curTime, limitTime)
	{
		while(list.length > 1)
		{
			if(list[1].t < limitTime)
			{
				list.shift();
			}
			else
			{
				break;
			}
		}
	}
	function cleanupEvents()
	{
		var curTime = Date.now();
		var limitTime = curTime - 10000;
		expireEvents(cpuActivityEvents, curTime, limitTime);
		computeCpuActivity(curTime, limitTime);
		if(cpuActivityEvents.length == 0)
		{
			clearInterval(activityEventsInterval);
			activityEventsInterval = 0;
		}
	}
	function computeCpuActivity(curTime, limitTime)
	{
		var totalActiveTime = 0;
		var lastActiveTime = limitTime;
		var lastWasActive = false;
		for(var i=0;i<cpuActivityEvents.length;i++)
		{
			var e = cpuActivityEvents[i];
			// NOTE: The first event could be before the limit,
			//       we need at least one event to correctly mark
			//       active time when there is long time under load
			var eTime = e.t;
			if(eTime < limitTime)
				eTime = limitTime;
			if(e.state == "ready")
			{
				// Inactive state, add the time from lastActiveTime
				totalActiveTime += (eTime - lastActiveTime);
				lastWasActive = false;
			}
			else
			{
				// Active state
				lastActiveTime = eTime;
				lastWasActive = true;
			}
		}
		// Add the last interval if needed
		if(lastWasActive)
		{
			totalActiveTime += (curTime - lastActiveTime);
		}
		cpuPercentage.set(Math.ceil((totalActiveTime / 10000) * 100));
	}
	function hddCallback(state)
	{
		diskActivity.set(state != "ready");
	}
	function latencyCallback(latency)
	{
		diskLatencies.push(latency);
		if(diskLatencies.length > 30)
			diskLatencies.shift();
		// Average the latency over at most 30 blocks
		var total = 0;
		for(var i=0;i<diskLatencies.length;i++)
			total += diskLatencies[i];
		var avg = total / diskLatencies.length;
		diskLatency.set(Math.ceil(avg));
	}
	function cpuCallback(state)
	{
		cpuActivity.set(state != "ready");
		var curTime = Date.now();
		var limitTime = curTime - 10000;
		expireEvents(cpuActivityEvents, curTime, limitTime);
		cpuActivityEvents.push({t: curTime, state: state});
		computeCpuActivity(curTime, limitTime);
		// Start an interval timer to cleanup old samples when no further activity is received
		if(activityEventsInterval != 0)
			clearInterval(activityEventsInterval);
		activityEventsInterval = setInterval(cleanupEvents, 2000);
	}
	function computeXTermFontSize()
	{
		return parseInt(getComputedStyle(document.body).fontSize);
	}
	function setScreenSize(display)
	{
		var internalMult = 1.0;
		var displayWidth = display.offsetWidth;
		var displayHeight = display.offsetHeight;
		var minWidth = 1024;
		var minHeight = 768;
		if(displayWidth < minWidth)
			internalMult = minWidth / displayWidth;
		if(displayHeight < minHeight)
			internalMult = Math.max(internalMult, minHeight / displayHeight);
		var internalWidth = Math.floor(displayWidth * internalMult);
		var internalHeight = Math.floor(displayHeight * internalMult);
		cx.setKmsCanvas(display, internalWidth, internalHeight);
		// Compute the size to be used for AI screenshots
		var screenshotMult = 1.0;
		var maxWidth = 1024;
		var maxHeight = 768;
		if(internalWidth > maxWidth)
			screenshotMult = maxWidth / internalWidth;
		if(internalHeight > maxHeight)
			screenshotMult = Math.min(screenshotMult, maxHeight / internalHeight);
		var screenshotWidth = Math.floor(internalWidth * screenshotMult);
		var screenshotHeight = Math.floor(internalHeight * screenshotMult);
		// Track the state of the mouse as requested by the AI, to avoid losing the position due to user movement
		displayConfig.set({width: screenshotWidth, height: screenshotHeight, mouseMult: internalMult * screenshotMult});
	}
	var curInnerWidth = 0;
	var curInnerHeight = 0;
	function handleResize()
	{
		// Avoid spurious resize events caused by the soft keyboard
		if(curInnerWidth == window.innerWidth && curInnerHeight == window.innerHeight)
			return;
		curInnerWidth = window.innerWidth;
		curInnerHeight = window.innerHeight;
		triggerResize();
	}
	function triggerResize()
	{
		term.options.fontSize = computeXTermFontSize();
		fitAddon.fit();
		const display = document.getElementById("display");
		if(display)
			setScreenSize(display);
	}
	async function initTerminal()
	{
		const { Terminal } = await import('@xterm/xterm');
		const { FitAddon } = await import('@xterm/addon-fit');
		const { WebLinksAddon } = await import('@xterm/addon-web-links');
		term = new Terminal({cursorBlink:true, convertEol:true, fontFamily:"monospace", fontWeight: 400, fontWeightBold: 700, fontSize: computeXTermFontSize()});
		fitAddon = new FitAddon();
		term.loadAddon(fitAddon);
		var linkAddon = new WebLinksAddon();
		term.loadAddon(linkAddon);
		const consoleDiv = document.getElementById("console");
		term.open(consoleDiv);
		term.scrollToTop();
		fitAddon.fit();
		window.addEventListener("resize", handleResize);
		term.focus();
		term.onData(readData);
		// Avoid undesired default DnD handling
		function preventDefaults (e) {
			e.preventDefault()
			e.stopPropagation()
		}
		consoleDiv.addEventListener("dragover", preventDefaults, false);
		consoleDiv.addEventListener("dragenter", preventDefaults, false);
		consoleDiv.addEventListener("dragleave", preventDefaults, false);
		consoleDiv.addEventListener("drop", preventDefaults, false);
		curInnerWidth = window.innerWidth;
		curInnerHeight = window.innerHeight;
		appendLog('Devlab Laboratory · UNIT Electronics · CH552 / SDCC');
		appendLog('Inicializando Linux y herramientas SDCC...');
		try
		{
			await initCheerpX();
		}
		catch(e)
		{
			appendLog(e.toString());
			printMessage(unexpectedErrorMessage);
			printMessage([e.toString()]);
			return;
		}
	}
	function handleActivateConsole(vt)
	{
		if(curVT == vt)
			return;
		curVT = vt;
		if(vt != 7)
			return;
		// Raise the display to the foreground
		const display = document.getElementById("display");
		display.parentElement.style.zIndex = 5;
		tryPlausible("Display activated");
	}
	function handleProcessCreated()
	{
		processCount++;
		if(processCallback)
			processCallback(processCount);
	}
	async function initCheerpX()
	{
		const CheerpX = await import('@leaningtech/cheerpx');
		var blockDevice = null;
		switch(configObj.diskImageType)
		{
			case "cloud":
				try
				{
					blockDevice = await CheerpX.CloudDevice.create(configObj.diskImageUrl);
				}
				catch(e)
				{
					// Report the failure and try again with plain HTTP
					var wssProtocol = "wss:";
					if(configObj.diskImageUrl.startsWith(wssProtocol))
					{
						// WebSocket protocol failed, try agin using plain HTTP
						tryPlausible("WS Disk failure");
						blockDevice = await CheerpX.CloudDevice.create("https:" + configObj.diskImageUrl.substr(wssProtocol.length));
					}
					else
					{
						// No other recovery option
						throw e;
					}
				}
				break;
			case "bytes":
				blockDevice = await CheerpX.HttpBytesDevice.create(configObj.diskImageUrl);
				break;
			case "github":
				blockDevice = await CheerpX.GitHubDevice.create(configObj.diskImageUrl);
				break;
			default:
				throw new Error("Unrecognized device type");
		}
		blockCache = await CheerpX.IDBDevice.create(cacheId);
		var overlayDevice = await CheerpX.OverlayDevice.create(blockDevice, blockCache);
		dataDevice = await CheerpX.DataDevice.create();
		var mountPoints = [
			// The root filesystem, as an Ext2 image
			{type:"ext2", dev:overlayDevice, path:"/"},
			// Access to read-only data coming from JavaScript
			{type:"dir", dev:dataDevice, path:"/data"},
			// Automatically created device files
			{type:"devs", path:"/dev"},
			// Pseudo-terminals
			{type:"devpts", path:"/dev/pts"},
			// The Linux 'proc' filesystem which provides information about running processes
			{type:"proc", path:"/proc"},
			// The Linux 'sysfs' filesystem which is used to enumerate emulated devices
			{type:"sys", path:"/sys"},
		];
		try
		{
			cx = await CheerpX.Linux.create({mounts: mountPoints, networkInterface: networkInterface});
		}
		catch(e)
		{
			appendLog(e.toString());
			printMessage(errorMessage);
			printMessage([e.toString()]);
			return;
		}
		cx.registerCallback("cpuActivity", cpuCallback);
		cx.registerCallback("diskActivity", hddCallback);
		cx.registerCallback("diskLatency", latencyCallback);
		cx.registerCallback("processCreated", handleProcessCreated);
		term.scrollToBottom();
		cxReadFunc = cx.setCustomConsole(writeData, term.cols, term.rows);
		const display = document.getElementById("display");
		if(display)
		{
			setScreenSize(display);
			cx.setActivateConsole(handleActivateConsole);
		}
		// Run the command in a loop, in case the user exits
		while (true)
		{
			await cx.run(configObj.cmd, configObj.args, configObj.opts);
		}
	}
	onMount(initTerminal);
	async function handleConnect()
	{
		const w = window.open("login.html", "_blank");
		cx.networkLogin();
		try
		{
			w.location.href = await startLogin();
		}
		catch(e)
		{
			w.close();
			console.warn(e);
		}
	}
	async function handleReset()
	{
		// Be robust before initialization
		if(blockCache == null)
			return;
		await blockCache.reset();
		location.reload();
	}
	async function handleTool(tool)
	{
		return await handleToolImpl(tool, term);
	}
	async function handleSidebarPinChange(event)
	{
		sideBarPinned = event.detail;
		// Make sure the pinning state of reflected in the layout
		await tick();
		// Adjust the layout based on the new sidebar state
		triggerResize();
	}
</script>

<main class="relative w-full h-full">
	<Nav />
	<div class="absolute top-10 bottom-0 left-0 right-0">
		<SideBar logs={startupLogs} on:connect={handleConnect} on:reset={handleReset} handleTool={!configObj.needsDisplay || curVT == 7 ? handleTool : null} on:sidebarPinChange={handleSidebarPinChange}>
			<slot></slot>
		</SideBar>
		{#if configObj.needsDisplay}
			<div class="absolute top-0 bottom-0 {sideBarPinned ? 'left-[23.5rem]' : 'left-14'} right-0">
				<canvas class="w-full h-full cursor-none" id="display"></canvas>
			</div>
		{/if}
		<div class="absolute top-0 bottom-0 {sideBarPinned ? 'left-[23.5rem]' : 'left-14'} right-0 p-1 scrollbar" id="console">
		</div>
		{#if !configObj.needsDisplay}
			<div class="absolute top-2 right-3 z-10 flex gap-2">
				<button class="rounded bg-emerald-500 px-3 py-2 text-sm font-bold text-slate-950 shadow" on:click={chooseDirectory} disabled={editorBusy || cx == null}>
					Abrir editor
				</button>
			</div>
		{/if}
		</div>
	{#if directoryOpen}
		<div class="fixed inset-0 z-20 flex items-center justify-center bg-slate-950/80 p-4">
			<section class="w-full max-w-xl rounded-lg border border-emerald-400/40 bg-slate-900 p-5 shadow-2xl">
				<h2 class="font-bold text-emerald-300">Elegir proyecto</h2>
				<p class="mb-4 text-sm text-slate-400">Selecciona el directorio que contiene el main.c.</p>
				<div class="grid max-h-[60vh] gap-2 overflow-y-auto">
					{#each editorDirectories as directory}
						<button class="rounded border border-slate-700 px-3 py-2 text-left text-sm text-slate-200 hover:border-emerald-400 hover:bg-slate-800" on:click={() => selectDirectory(directory)}>
							{directory.replace(`${examplesRoot}/`, '')}
						</button>
					{/each}
				</div>
				<button class="mt-4 rounded border border-slate-600 px-3 py-2 text-sm text-slate-300" on:click={() => directoryOpen = false}>Cancelar</button>
			</section>
		</div>
	{/if}
	{#if editorOpen}
		<div class="pointer-events-none fixed inset-0 z-20">
			<section class="pointer-events-auto absolute right-2 top-2 bottom-2 flex w-[calc(100%-1rem)] flex-col rounded-lg border border-emerald-400/40 bg-slate-900 shadow-2xl md:w-[min(68vw,56rem)]">
				<header class="flex items-center justify-between border-b border-slate-700 px-4 py-3">
					<div>
						<h2 class="font-bold text-emerald-300">Editor CH552</h2>
						<p class="text-xs text-slate-400">{selectedFile()}</p>
					</div>
					<button class="text-xl text-slate-300 hover:text-white" on:click={() => editorOpen = false} aria-label="Cerrar editor">×</button>
				</header>
				<textarea class="min-h-0 flex-1 resize-none bg-slate-950 p-4 font-mono text-sm leading-6 text-slate-100 outline-none" bind:value={editorContent} spellcheck="false"></textarea>
				<footer class="flex items-center justify-between gap-3 border-t border-slate-700 px-4 py-3">
					<span class="text-sm text-slate-400">{editorStatus}</span>
					<div class="flex gap-2">
						<button class="rounded border border-slate-600 px-3 py-2 text-sm text-slate-200" on:click={() => editorOpen = false}>Cerrar</button>
						<button class="rounded bg-slate-700 px-3 py-2 text-sm font-bold text-white" on:click={saveEditor} disabled={editorBusy}>Guardar</button>
						<button class="rounded bg-emerald-500 px-3 py-2 text-sm font-bold text-slate-950" on:click={compileEditor} disabled={editorBusy}>Guardar y compilar BIN</button>
						<button class="rounded border border-emerald-500 px-3 py-2 text-sm font-bold text-emerald-300" on:click={downloadBin} disabled={editorBusy}>Descargar BIN</button>
					</div>
				</footer>
			</section>
		</div>
	{/if}
</main>
