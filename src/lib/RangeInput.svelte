<script>
	export let label = '';
	export let min = 0;
	export let max = 100;
	export let step = 1;
	export let value = 50;
	export let valueDesc = '';
</script>

<label>
	<span class="label">{label}</span>
	<input type="range" {min} {max} {step} bind:value />
	<div class="value">
		<input type="number" {min} {max} {step} bind:value />
		<span>{value + ' ' + valueDesc}</span>
	</div>
</label>

<style>
	label {
		display: grid;
		grid-template-columns: 1fr 1fr 1fr;
		column-gap: 1rem;
	}
	.label {
		justify-self: end;
	}
	.value {
		display: flex;
		align-items: center;
		gap: 1rem;
	}
	.value > input {
		width: 4rem;
	}
</style>
