<script lang="ts">
	import { confetti } from '@neoconfetti/svelte';
	import { tweened } from 'svelte/motion';
	import { fade } from 'svelte/transition';
	import { FFmpeg } from '@ffmpeg/ffmpeg';
	import { onMount } from 'svelte';
	import JSZip from 'jszip';
	import TermOutput from './components/TermOutput.svelte';

	type State = 'loading' | 'loaded' | 'convert.start' | 'convert.error' | 'convert.done';

	interface FrameFile {
		name: string;
		data: Uint8Array;
	}

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

	async function getVideoFrames(file: File): Promise<FrameFile[]> {
		state = 'convert.start';

		const videoData = await fetchFile(file);
		await ffmpeg.writeFile(file.name, videoData);
		await ffmpeg.createDir('frames');
		await ffmpeg.exec(['-i', file.name, '-vf', 'fps=1', 'frames/f%04d.png']);
		const frameNodes = await ffmpeg.listDir('frames');
		console.log(frameNodes);
		let frameDataP = frameNodes
			.filter((file) => !file.isDir)
			.map(async (file) => {
				console.log(file);
				const data = await ffmpeg.readFile(`frames/${file.name}`);
				return { name: file.name, data: data as Uint8Array };
			});

		return Promise.all(frameDataP);
	}

	async function downloadFrames(frames: FrameFile[]) {
		const zip = new JSZip();

		frames.forEach(({ name, data }) => {
			zip.file(name, data);
		});

		const zipped = await zip.generateAsync({ type: 'blob' });

		const a = document.createElement('a');
		a.href = URL.createObjectURL(new Blob([zipped], { type: 'application/zip' }));
		a.download = 'frames.zip';

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
			const data = await getVideoFrames(file);
			state = 'convert.done';
			await downloadFrames(data);
			state = 'loaded';
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
</script>

<div class="body">
	<div class="info-container">
		<h1 class="title">Frame Extractor</h1>
		<p>
			This Svelte app uses <a href="https://ffmpegwasm.netlify.app">ffmpeg.wasm</a> to extract frames
			(one per second, for now!) from a video file and saves them as a zip file, all without any information
			leaving your browser! Drag a video file into the box below to get started.
		</p>
	</div>

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
			<p in:fade>Drag and drop your video here</p>
		{/if}
		{#if state === 'convert.start'}
			<p in:fade>Converting...</p>
			<div class="progress-bar">
				<div class="progress" style:--progress="{$progress}%">
					{$progress.toFixed(0)}%
				</div>
			</div>
		{/if}
		{#if state === 'convert.done'}
			<div use:confetti />
			<p in:fade>Done! 🎉</p>
		{/if}

		{#if error}
			<p in:fade class="error">{error}</p>
		{/if}
	</div>

	<TermOutput {output} />
</div>

<style>
	.title {
		text-align: center;
	}

	.body {
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
</style>
