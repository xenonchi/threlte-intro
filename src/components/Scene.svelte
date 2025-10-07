<script lang="ts">
	import { T } from '@threlte/core';
	import { interactivity } from '@threlte/extras';
	import { Spring } from 'svelte/motion';
	import { onMount, onDestroy } from 'svelte';
	interactivity();
	const scale = new Spring(1);

	// Accept props
	let {
		boxColor = 'hotpink',
		rotationX = $bindable(0),
		rotationY = $bindable(0),
		rotationZ = $bindable(0)
	}: {
		boxColor?: string;
		rotationX?: number;
		rotationY?: number;
		rotationZ?: number;
	} = $props();

	// Drag state
	let isDragging = false;
	let lastMouseX = 0;
	let lastMouseY = 0;

	// Drag sensitivity
	const dragSensitivity = 0.01;

	function handleMouseDown(event: any) {
		event.stopPropagation();
		isDragging = true;
		lastMouseX = event.clientX || event.offsetX;
		lastMouseY = event.clientY || event.offsetY;
		document.body.style.cursor = 'grabbing';
		console.log('Mouse down, dragging started');
	}

	function handleMouseMove(event: MouseEvent) {
		if (!isDragging) return;

		const deltaX = event.clientX - lastMouseX;
		const deltaY = event.clientY - lastMouseY;

		// Update rotations based on mouse movement
		rotationY += deltaX * dragSensitivity;
		rotationX += deltaY * dragSensitivity;

		lastMouseX = event.clientX;
		lastMouseY = event.clientY;

		console.log('Dragging:', { deltaX, deltaY, rotationX, rotationY });
	}

	function handleMouseUp() {
		isDragging = false;
		document.body.style.cursor = 'default';
		console.log('Mouse up, dragging stopped');
	}

	// Touch events for mobile
	function handleTouchStart(event: TouchEvent) {
		if (event.touches.length === 1) {
			isDragging = true;
			lastMouseX = event.touches[0].clientX;
			lastMouseY = event.touches[0].clientY;
		}
	}

	function handleTouchMove(event: TouchEvent) {
		if (!isDragging || event.touches.length !== 1) return;

		event.preventDefault();
		const deltaX = event.touches[0].clientX - lastMouseX;
		const deltaY = event.touches[0].clientY - lastMouseY;

		rotationY += deltaX * dragSensitivity;
		rotationX += deltaY * dragSensitivity;

		lastMouseX = event.touches[0].clientX;
		lastMouseY = event.touches[0].clientY;
	}

	function handleTouchEnd() {
		isDragging = false;
	}

	// Add event listeners when component mounts
	onMount(() => {
		document.addEventListener('mousemove', handleMouseMove);
		document.addEventListener('mouseup', handleMouseUp);
		document.addEventListener('touchmove', handleTouchMove, { passive: false });
		document.addEventListener('touchend', handleTouchEnd);
	});

	onDestroy(() => {
		document.removeEventListener('mousemove', handleMouseMove);
		document.removeEventListener('mouseup', handleMouseUp);
		document.removeEventListener('touchmove', handleTouchMove);
		document.removeEventListener('touchend', handleTouchEnd);
	});
</script>

<T.PerspectiveCamera
	makeDefault
	position={[10, 10, 10]}
	oncreate={(ref) => {
		ref.lookAt(0, 1, 0);
	}}
/>
<T.Mesh
	position.y={1}
	scale={scale.current}
	rotation.x={rotationX}
	rotation.y={rotationY}
	rotation.z={rotationZ}
	onpointerenter={() => {
		scale.target = 1.5;
		if (!isDragging) {
			document.body.style.cursor = 'grab';
		}
	}}
	onpointerleave={() => {
		scale.target = 1;
		if (!isDragging) {
			document.body.style.cursor = 'default';
		}
	}}
	onpointerdown={handleMouseDown}
	onpointerup={handleMouseUp}
>
	<T.BoxGeometry args={[1, 2, 1]} />
	<T.MeshBasicMaterial color={boxColor} />
</T.Mesh>
