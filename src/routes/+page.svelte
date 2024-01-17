<script>
	import Plot from 'svelte-plotly.js';
	import Canvas from '$lib/Canvas.svelte';
	import RangeInput from '$lib/RangeInput.svelte';

	const viableClusterSize = 3;
	const params = {
		gridSize: { min: 1, max: 100, step: 1, default: 50 },
		chipsAmount: { min: 1, step: 1, default: 800 },
		termitesAmount: { min: 1, max: 200, step: 1, default: 200 },
		termitesMoveDistance: { min: 0, max: 1, step: 0.01, default: 1 },
		termitesMoveAngleRange: { min: 1, max: 360, step: 1, default: 50 },
		termitesOnlyMoveTicks: { min: 0, max: 200, step: 1, default: 20 },
		simIntervalTimeout: { min: 1, max: 200, step: 1, default: 30 }
	};

	let canvas;

	let gridWrap = true;
	let gridSize = params.gridSize.default;
	$: cellsAmount = gridSize * gridSize;

	let chips = [];
	let chipsAmount = params.chipsAmount.default;
	$: if (chipsAmount > cellsAmount) chipsAmount = cellsAmount;

	let termites = [];
	let termitesAmount = params.termitesAmount.default;
	let termitesMoveDistance = params.termitesMoveDistance.default;
	let termitesMoveAngleRange = params.termitesMoveAngleRange.default;
	let termitesOnlyMoveTicks = params.termitesOnlyMoveTicks.default;

	let simRunning = false;
	let simFinished = false;
	let simOnce = false;
	let simInterval;
	let simIntervalTimeout = params.simIntervalTimeout.default;
	let finisher = true;

	let plotlyData = [{ x: [], y: [], type: 'scatter' }];

	function generateCoords(gridSize) {
		const x = Math.floor(Math.random() * gridSize);
		const y = Math.floor(Math.random() * gridSize);
		return { x, y };
	}

	function generateChips(amount, gridSize) {
		const chips = [];
		for (let i = 0; i < amount; i++) {
			let { x, y } = generateCoords(gridSize);
			while (chips.some((chip) => chip.x === x && chip.y === y)) {
				({ x, y } = generateCoords(gridSize));
			}
			chips.push({ x, y, cluster: null });
		}
		return chips;
	}

	function generateTermites(amount, gridSize) {
		return new Array(amount).fill().map(() => ({
			x: Math.floor(Math.random() * gridSize),
			y: Math.floor(Math.random() * gridSize),
			angle: Math.floor(Math.random() * 360),
			state: 'search-for-chip', // search-for-chip, find-new-pile, put-down-chip, get-away
			onlyMoveTicks: 0,
			carrying: null
		}));
	}

	function rotate(termite, angle) {
		if (angle === 360) {
			termite.angle += Math.random() * 360;
		} else {
			termite.angle += Math.random() * angle;
			termite.angle -= Math.random() * angle;
		}
	}

	function move(termite) {
		// Change x any y based on angle
		termite.x += Math.cos(termite.angle * (Math.PI / 180)) * termitesMoveDistance;
		termite.y += Math.sin(termite.angle * (Math.PI / 180)) * termitesMoveDistance;
		if (gridWrap) {
			// Wrap around the grid
			if (termite.x < 0) termite.x = gridSize;
			if (termite.x > gridSize) termite.x = 0;
			if (termite.y < 0) termite.y = gridSize;
			if (termite.y > gridSize) termite.y = 0;
		} else {
			// Block termites from going out of the grid
			if (termite.x < 0) termite.x = 0;
			if (termite.x > gridSize) termite.x = gridSize;
			if (termite.y < 0) termite.y = 0;
			if (termite.y > gridSize) termite.y = gridSize;
		}
	}

	function wiggle(termite) {
		move(termite);
		rotate(termite, termitesMoveAngleRange);
	}

	function checkChip(termite, func) {
		if (termite.x < 0 || termite.x > gridSize || termite.y < 0 || termite.y > gridSize) {
			return func(null, null);
		}
		const tx = Math.floor(termite.x);
		const ty = Math.floor(termite.y);
		const i = chips.findIndex((chip) => chip.x === tx && chip.y === ty);
		return func(chips[i], i);
	}

	function getClusterNeighborsPositon(point, distance) {
		const positions = [];
		for (let d = 1; d <= distance; d++) {
			const newPositions = [
				{ x: point.x - d, y: point.y - d },
				{ x: point.x, y: point.y - d },
				{ x: point.x + d, y: point.y - d },
				{ x: point.x - d, y: point.y },
				{ x: point.x + d, y: point.y },
				{ x: point.x - d, y: point.y + d },
				{ x: point.x, y: point.y + d },
				{ x: point.x + d, y: point.y + d }
			];
			// remove positions that are out of bounds
			for (const pos of newPositions) {
				if (gridWrap) {
					if (pos.x < 0) pos.x = d + gridSize - 1;
					if (pos.x > gridSize) pos.x = d - 1;
					if (pos.y < 0) pos.y = d + gridSize - 1;
					if (pos.y > gridSize) pos.y = d - 1;
				} else {
					if (pos.x < 0 || pos.x > gridSize || pos.y < 0 || pos.y > gridSize) {
						newPositions.splice(newPositions.indexOf(pos), 1);
					}
				}
			}
			positions.push(...newPositions);
		}
		return positions;
	}

	function getClusters(points) {
		let clusterCounter = 0;
		for (let i = 0; i < points.length; i++) {
			const point = points[i];
			const neighborPositions = getClusterNeighborsPositon(point, 1);
			const neighbors = points.filter((p) =>
				neighborPositions.some((n) => n.x === p.x && n.y === p.y)
			);
			for (const neighbor of neighbors) {
				if (
					neighbor.cluster !== null &&
					point.cluster !== null &&
					neighbor.cluster !== point.cluster
				) {
					// Merge clusters
					const oldCluster = neighbor.cluster;
					const newCluster = point.cluster;
					for (const p of points) {
						if (p.cluster === oldCluster) p.cluster = newCluster;
					}
				} else if (neighbor.cluster !== null && point.cluster === null) {
					// Add to clusterw
					point.cluster = neighbor.cluster;
				}
			}
			if (point.cluster === null) {
				// Create new cluster
				point.cluster = clusterCounter++;
			}
		}

		const clusters = {};
		for (const point of points) {
			if (clusters[point.cluster]) {
				clusters[point.cluster].push(point);
			} else {
				clusters[point.cluster] = [point];
			}
		}
		return Object.values(clusters);
	}

	function tick() {
		termites.forEach((termite) => {
			if (termite.state === 'search-for-chip') {
				if (termite.onlyMoveTicks > 0) {
					termite.onlyMoveTicks--;
					move(termite);
					if (termite.onlyMoveTicks === 0) {
						termite.state = 'find-new-pile';
					}
				} else {
					checkChip(termite, (chip, i) => {
						if (chip) {
							chips.splice(i, 1);
							termite.carrying = true;
							termite.onlyMoveTicks = termitesOnlyMoveTicks;
						} else {
							wiggle(termite);
						}
					});
				}
			} else if (termite.state === 'find-new-pile') {
				checkChip(termite, (chip) => {
					if (!chip) {
						wiggle(termite);
					} else {
						termite.state = 'put-down-chip';
					}
				});
			} else if (termite.state === 'put-down-chip') {
				checkChip(termite, (chip) => {
					if (!chip) {
						const tx = Math.floor(termite.x);
						const ty = Math.floor(termite.y);
						chips.push({ x: tx, y: ty, cluster: null });
						termite.carrying = false;
						termite.state = 'get-away';
					} else {
						rotate(termite, 360);
						move(termite);
					}
				});
			} else if (termite.state === 'get-away') {
				if (termite.onlyMoveTicks > 0) {
					termite.onlyMoveTicks--;
					move(termite);
					if (termite.onlyMoveTicks === 0) {
						checkChip(termite, (chip) => {
							if (!chip) {
								termite.state = 'search-for-chip';
							}
						});
					}
				} else {
					rotate(termite, 360);
					termite.onlyMoveTicks = termitesOnlyMoveTicks;
				}
			}
		});

		const points = chips.map((chip) => ({ x: chip.x, y: chip.y, cluster: null }));
		const clusters = getClusters(points);
		clusters.forEach((cluster, i) => {
			cluster.forEach((point) => {
				const chip = chips.find((chip) => chip.x === point.x && chip.y === point.y);
				chip.cluster = i;
			});
		});
		plotlyData[0].x.push(plotlyData[0].x.length);
		plotlyData[0].y.push(clusters.length);
		plotlyData = [...plotlyData];

		const finished = clusters.filter((cluster) => cluster.length >= viableClusterSize).length === 1;
		return { finished, clusters: clusters.length };
	}

	function setup() {
		chips = generateChips(chipsAmount, gridSize);
		termites = generateTermites(termitesAmount, gridSize);
	}

	function simRun() {
		if (simRunning) return;
		simRunning = true;
		simFinished = false;
	}
	function simStop() {
		simRunning = false;
		clearInterval(simInterval);
	}
	function simTick() {
		const { finished } = tick();
		if (finisher && finished) {
			simFinished = true;
			simStop();
		}
		canvas.draw();
	}
	function simSetup() {
		simFinished = false;
		setup();
		canvas.draw();
	}

	$: if (canvas && chipsAmount && termitesAmount) {
		simSetup();
	}
	$: if (canvas && simRunning && simIntervalTimeout) {
		clearInterval(simInterval);
		simInterval = setInterval(simTick, simIntervalTimeout);
	}

	function simulateOnce() {
		if (simOnce) return;
		simOnce = true;
		const result = { ticks: [], clusters: [] };
		setup();
		let ticks = 0;
		let finished = false;
		while (!finished) {
			const data = tick();
			finished = data.finished;
			result.ticks.push(ticks);
			result.clusters.push(data.clusters);
			ticks++;
		}
		plotlyData.push({ x: result.ticks, y: result.clusters, type: 'scatter' });
		simOnce = false;
		return {
			gridSize,
			cellsAmount,
			termitesAmount,
			termitesMoveDistance,
			termitesMoveAngleRange,
			termitesOnlyMoveTicks,
			data: result
		};
	}
	function simulate() {
		let progress = 0;
		const results = [];
		for (let loop = 0; loop < 100; loop++) {
			for (let ca = 100; ca <= 2500; ca += 100) {
				for (let ta = 20; ta <= 200; ta += 20) {
					for (let tmar = 10; tmar <= 360; tmar += 10) {
						for (let tomt = 5; tomt <= 40; tomt += 5) {
							progress = loop / 100;
							gridSize = gs;
							chipsAmount = ca;
							termitesAmount = ta;
							termitesMoveDistance = tmd;
							termitesMoveAngleRange = tmar;
							termitesOnlyMoveTicks = tomt;
							const result = { loop, ...simulateOnce(loop, 50, ca, ta, 1, tmar, tomt) };
							results.push(result);
						}
					}
				}
			}
		}
		return results;
	}
</script>

<svelte:head>
	<title>Termites</title>
</svelte:head>

<main>
	<div class="visual">
		<Canvas bind:this={canvas} {gridSize} {chips} {termites} />
		<div class="plot">
			<Plot data={plotlyData} layout={{ width: 500, height: 500 }} />
			{#if simFinished}
				<p><b>Simulation finished</b></p>
			{/if}
		</div>
	</div>

	<div class="ranges">
		<RangeInput
			min={params.gridSize.min}
			max={params.gridSize.max}
			bind:value={gridSize}
			label="Map size"
		/>
		<RangeInput
			min={params.chipsAmount.min}
			max={cellsAmount}
			bind:value={chipsAmount}
			label="Chips"
		/>
		<RangeInput
			min={params.termitesAmount.min}
			max={params.termitesAmount.max}
			bind:value={termitesAmount}
			label="Termites"
		/>
		<RangeInput
			min={params.termitesMoveDistance.min}
			max={params.termitesMoveDistance.max}
			step={params.termitesMoveDistance.step}
			bind:value={termitesMoveDistance}
			label="Termites move distance"
			valueDesc="cells/tick"
		/>
		<RangeInput
			min={params.termitesMoveAngleRange.min}
			max={params.termitesMoveAngleRange.max}
			bind:value={termitesMoveAngleRange}
			label="Termites move angle range"
			valueDesc="deg/tick"
		/>
		<RangeInput
			min={params.termitesOnlyMoveTicks.min}
			max={params.termitesOnlyMoveTicks.max}
			bind:value={termitesOnlyMoveTicks}
			label="Termites only move ticks"
			valueDesc="ticks"
		/>
		<RangeInput
			min={params.simIntervalTimeout.min}
			max={params.simIntervalTimeout.max}
			bind:value={simIntervalTimeout}
			label="Simulation interval"
			valueDesc="ms"
		/>
	</div>

	<div class="buttons">
		{#if simRunning}
			<button on:click={simStop} class="red">Stop</button>
		{:else}
			<button on:click={simRun} class="green">Run</button>
		{/if}
		<button on:click={simTick}>Tick</button>
		<button on:click={simSetup}>Setup</button>
		<button on:click={() => (gridWrap = !gridWrap)}>{gridWrap ? 'Unwrap' : 'Wrap'}</button>
		<button on:click={() => (finisher = !finisher)}>{finisher ? 'Manual' : 'Auto'}</button>
		<button on:click={simulateOnce} class="blue">{simOnce ? '...' : 'Simulate'}</button>
	</div>
</main>

<style>
	* {
		font-family: 'Poppins', sans-serif;
	}

	main {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 2rem;
		width: 100%;
	}

	.visual {
		display: flex;
		justify-content: center;
		align-items: center;
		gap: 2rem;
		width: 100%;
	}
	.plot p {
		text-align: center;
	}

	.ranges {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		width: 100%;
	}

	.buttons {
		display: flex;
		gap: 2rem;
	}
	button {
		padding: 0.6rem 0;
		border-radius: 0.5rem;
		border: none;
		background: #eee;
		font-weight: bold;
		cursor: pointer;
		width: 5rem;
	}
	button:hover {
		background: #ddd;
	}
	button.green {
		background: #74d877;
	}
	button.green:hover {
		background: #6bc76e;
	}
	button.red {
		background: #f44336;
	}
	button.red:hover {
		background: #e53935;
	}
	button.blue {
		color: #fff;
		background: #3682f4;
	}
	button.blue:hover {
		background: #355de0;
	}
</style>
