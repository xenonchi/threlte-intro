<script lang="ts">
	import { Canvas } from '@threlte/core';
	import Scene from '../components/Scene.svelte';

	// Color state
	let boxColor = 'hotpink';

	// Rotation state
	let rotationX = 0;
	let rotationY = 0;
	let rotationZ = 0;

	// Color options
	const colors = [
		{ name: 'Hot Pink', value: 'hotpink' },
		{ name: 'Blue', value: '#3498db' },
		{ name: 'Green', value: '#2ecc71' },
		{ name: 'Orange', value: '#f39c12' },
		{ name: 'Purple', value: '#9b59b6' },
		{ name: 'Red', value: '#e74c3c' }
	];

	function changeColor(color: string): void {
		boxColor = color;
	}
</script>

<div class="page-container">
	<Canvas>
		<Scene {boxColor} bind:rotationX bind:rotationY bind:rotationZ />
	</Canvas>

	<!-- Color Selection Buttons -->
	<div class="color-controls">
		<h3>Change Box Color</h3>
		<div class="color-buttons">
			{#each colors as color}
				<button
					class="color-btn"
					style="background-color: {color.value};"
					on:click={() => changeColor(color.value)}
					title={color.name}
				>
					{color.name}
				</button>
			{/each}
		</div>
	</div>
</div>

<style>
	.page-container {
		position: relative;
		width: 100vw;
		height: 100vh;
		background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
	}

	.color-controls {
		position: absolute;
		bottom: 20px;
		left: 20px;
		background: rgba(0, 0, 0, 0.8);
		padding: 20px;
		border-radius: 10px;
		backdrop-filter: blur(10px);
		z-index: 10;
	}

	.color-controls h3 {
		color: white;
		margin: 0 0 15px 0;
		font-size: 16px;
		font-weight: 600;
	}

	.color-buttons {
		display: flex;
		gap: 10px;
		flex-wrap: wrap;
	}

	.color-btn {
		padding: 8px 12px;
		border: 2px solid rgba(255, 255, 255, 0.3);
		border-radius: 6px;
		color: white;
		font-size: 12px;
		font-weight: 500;
		cursor: pointer;
		transition: all 0.2s ease;
		text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
	}

	.color-btn:hover {
		border-color: rgba(255, 255, 255, 0.8);
		transform: translateY(-2px);
		box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
	}

	.color-btn:active {
		transform: translateY(0);
	}
</style>
