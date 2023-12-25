<script lang="ts">
	import { afterUpdate } from 'svelte';

	export let output: string[] = [];

	let termOutputEl: HTMLElement;
	let isUserScrolling = false;

	function onScroll() {
		const isAtBottom =
			termOutputEl.scrollHeight - termOutputEl.scrollTop === termOutputEl.clientHeight;
		isUserScrolling = !isAtBottom;
	}

	afterUpdate(() => {
		if (!isUserScrolling) {
			termOutputEl.scrollTop = termOutputEl.scrollHeight;
		}
	});
</script>

<div bind:this={termOutputEl} on:scroll={onScroll} class="term">
	<div class="term-output">
		{#each output as line}
			<div class="term-line"><span class="term-text">{line}</span></div>
		{/each}
	</div>
</div>

<style>
	.term {
		background-color: hsl(0, 0%, 10%);
		color: hsl(300, 50%, 40%);
		font-family: 'Iosevka', monospace;
		padding: 10px;
		margin-bottom: 30px;
		overflow-y: auto;
		height: 300px;
		width: auto;
		border: 1px solid hsl(0 0% 30%);
		border-radius: 5px;
	}
	.term-output {
		white-space: pre;
	}
	.term-line {
		margin: 0;
	}
	.term-text {
		display: block;
	}
</style>
