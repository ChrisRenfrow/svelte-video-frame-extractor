<script lang="ts">
	import { confetti } from '@neoconfetti/svelte';
	import { tweened } from 'svelte/motion';
	import { fade } from 'svelte/transition';
	import { FFmpeg } from '@ffmpeg/ffmpeg';
	import { onMount } from 'svelte';

	type State = 'loading' | 'loaded' | 'convert.start' | 'convert.error' | 'convert.done';

	let state: State = 'loading';
	let error = '';
	let ffmpeg: FFmpeg;
	let progress = tweened(0);
	let output = [''];

	async function fetchFile(file: File): Promise<Uint8Array> {
		return new Promise((resolve) => {
			const fileReader = new FileReader();

			fileReader.onload = () => {
				const { result } = fileReader;
				if (result instanceof ArrayBuffer) {
					resolve(new Uint8Array(result));
				}
			};

			fileReader.onerror = () => {
				error = 'Could not read file';
			};

			fileReader.readAsArrayBuffer(file);
		});
	}

	async function convertVideo(file: File) {
		state = 'convert.start';

		const videoData = await fetchFile(file);
		await ffmpeg.writeFile('input.webm', videoData);
		await ffmpeg.exec(['-i', 'input.webm', 'output.mp4']);
		const data = await ffmpeg.readFile('output.mp4');

		state = 'convert.done';

		return data as Uint8Array;
	}

	function downloadVideo(data: Uint8Array) {
		const a = document.createElement('a');
		a.href = URL.createObjectURL(new Blob([data.buffer], { type: 'video/mp4' }));
		a.download = 'video.mp4';

		setTimeout(() => {
			a.click();
		}, 1000);
	}

	async function handleDrop(event: DragEvent) {
		if (state !== 'loaded' || !event.dataTransfer) return;
		if (event.dataTransfer.files.length > 1) {
			error = 'One file at a time please.';
		}

		const [file] = event.dataTransfer.files;

		if (file.type === 'video/webm') {
			error = '';
			const data = await convertVideo(file);
			downloadVideo(data);
		} else {
			error = `Unsupported format: ${file.type}`;
		}
	}

	async function loadFFmpeg() {
		const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.5/dist/esm';
		ffmpeg = new FFmpeg();

		ffmpeg.on('log', (msg) => {
			output = [...output, msg.message];
		});

		ffmpeg.on('progress', (event) => {
			$progress = event.progress * 100;
		});

		await ffmpeg.load({
			coreURL: `${baseURL}/ffmpeg-core.js`,
			wasmURL: `${baseURL}/ffmpeg-core.wasm`
		});

		state = 'loaded';
	}

	onMount(() => {
		loadFFmpeg();
	});

	$: console.log(state);
</script>

<h1 class="title">Converter</h1>

<div class="container">
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<div
		on:drop|preventDefault={handleDrop}
		on:dragover|preventDefault={() => {}}
		data-state={state}
		class="drop"
	>
		{#if state === 'loading'}
			<p in:fade>Loading FFmpeg...</p>
		{/if}
		{#if state === 'loaded'}
			<p in:fade>Drag video here</p>
		{/if}
		{#if state === 'convert.start'}
			<p in:fade>Converting...</p>
			<div class="progress-bar">
				<div class="progress" style:--progress={$progress}>
					{$progress.toFixed(0)}%
				</div>
			</div>
		{/if}
		{#if state === 'convert.done'}
			<div use:confetti />
			<p in:fade>Done!</p>
		{/if}

		{#if error}
			<p in:fade class="error">{error}</p>
		{/if}
	</div>

	<div class="terminal">
		<div class="terminal-output">
			{#each output as line}
				<div class="terminal-line">
					<span class="terminal-text">{line}</span>
				</div>
			{/each}
		</div>
	</div>
</div>

<style>
	.title {
		text-align: center;
	}

	.container {
		width: 600px;
		display: grid;
		place-content: center;
	}

	.drop {
		width: 600px;
		height: 200px;
		display: grid;
		place-content: center;
		margin-block-start: 2rem;
		border: 3px dashed hsl(0 0% 20%);
		border-radius: 8px;
		margin-bottom: 20px;

		& p {
			font-size: 2rem;
			text-align: center;

			&.error {
				color: hsl(9, 100%, 64%);
			}
		}
	}

	.progress-bar {
		--progress-bar-clr: hsl(180deg 100% 50%);
		--progress-txt-clr: hsl(0 0% 0%);

		width: 100%;
		height: 40px;
		position: relative;
		font-weight: 700;
		background-color: hsl(220deg 10% 14%);
		border-radius: 8px;

		& .progress {
			width: var(--progress);
			height: 100%;
			position: absolute;
			left: 0%;
			display: grid;
			place-content: center;
			background: var(--progress-bar-clr);
			color: var(--progress-txt-clr);
			border-radius: 8px;
		}
	}

	.terminal {
		background-color: hsl(0, 0%, 10%);
		color: hsl(300, 50%, 40%);
		font-family: 'Iosevka', monospace;
		padding: 10px;
		margin-bottom: 30px;
		overflow-y: auto;
		height: 300px;
		width: 600px;
		border: 1px solid hsl(0 0% 30%);
		border-radius: 5px;
		& .terminal-output {
			white-space: pre;
			& .terminal-line {
				margin: 0;
				& .terminal-text {
					display: block;
				}
			}
		}
	}
</style>
