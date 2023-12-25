<script lang="ts">
	import { afterUpdate } from 'svelte';

	export let output: string[];

	let termOutputEl: HTMLElement;
	let isUserScrolling = false;
	let isExpanded = false;

	function onScroll() {
		const isAtBottom =
			termOutputEl.scrollHeight - termOutputEl.scrollTop === termOutputEl.clientHeight;
		isUserScrolling = !isAtBottom;
	}

	afterUpdate(() => {
		if (!isUserScrolling && isExpanded) {
			termOutputEl.scrollTop = termOutputEl.scrollHeight;
		}
	});
</script>

<div bind:this={termOutputEl} on:scroll={onScroll} class="term">
	{#if isExpanded}
		<div class="term-output">
			{#each output as line}
				<div class="term-line"><span class="term-text">{line}</span></div>
			{/each}
		</div>
		<hr />
	{/if}
	<div class="expand">
		<button on:click={() => (isExpanded = !isExpanded)} class="expand-button"
			>{isExpanded ? 'Hide output' : 'Show output'}</button
		>
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
		max-height: 300px;
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

	.expand-button {
		all: unset;
		outline: inherit;
		color: hsl(300, 50%, 60%);
		cursor: pointer;
		text-decoration: underline;
	}
</style>
