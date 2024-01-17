<script>
	import { onMount } from 'svelte';

	export let gridSize;
	export let chips;
	export let termites;

	let canvas;
	let canvasSize = 500;
	let ctx;

	$: cellSize = canvasSize / gridSize;

	$: if (canvas && canvasSize && gridSize) {
		// updates the canvas size and redraws
		canvas.width = canvasSize;
		canvas.height = canvasSize;
		draw();
	}

	onMount(() => {
		ctx = canvas.getContext('2d');
	});

	function getColor(percent) {
		return `hsl(${percent * 260}, 60%, 50%)`; // 260 instead of 360 to avoid repeating red
	}

	export function draw() {
		ctx.clearRect(0, 0, canvas.width, canvas.height);

		const clusters = new Set(chips.map((chip) => chip.cluster));
		for (const chip of chips) {
			ctx.fillStyle = chip.cluster === null ? 'grey' : getColor(chip.cluster / clusters.size);
			ctx.fillRect(chip.x * cellSize, chip.y * cellSize, cellSize, cellSize);
		}

		for (const termite of termites) {
			ctx.save();

			ctx.translate(termite.x * cellSize, termite.y * cellSize);
			ctx.rotate((termite.angle * Math.PI) / 180);

			// body
			ctx.beginPath();
			ctx.ellipse(0, 0, cellSize * 0.6, cellSize * 0.4, 0, 0, 2 * Math.PI);
			ctx.fillStyle = termite.carrying ? 'brown' : 'black';
			ctx.fill();

			// head
			ctx.beginPath();
			ctx.ellipse(cellSize * 0.5, 0, cellSize * 0.25, cellSize * 0.25, 0, 0, 2 * Math.PI);
			ctx.fillStyle = 'grey';
			ctx.fill();

			ctx.restore();
		}
	}
</script>

<div class="wrapper">
	<input type="range" min="200" max="800" bind:value={canvasSize} />
	<canvas bind:this={canvas}></canvas>
</div>

<style>
	.wrapper {
		display: flex;
		flex-direction: column;
		align-items: center;
	}
	input {
		width: 300px;
	}
	canvas {
		border: 1px solid rgb(211, 211, 211);
	}
</style>
