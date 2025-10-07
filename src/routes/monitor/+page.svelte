<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

	let canvasContainer: HTMLDivElement;
	let renderer: THREE.WebGLRenderer;
	let scene: THREE.Scene;
	let camera: THREE.PerspectiveCamera;
	let mixer: THREE.AnimationMixer | null = null;
	let clock: THREE.Clock;
	let animationId: number;
	let animationClips: THREE.AnimationClip[] = [];
	let currentFrame: number = 1;
	let isAnimating: boolean = false;

	onMount(() => {
		if (!browser) return;

		// Create scene
		scene = new THREE.Scene();
		scene.background = new THREE.Color(0x87ceeb);

		// Create camera
		camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
		camera.position.set(2, 2, 2);

		// Create renderer
		renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(window.innerWidth, window.innerHeight);
		canvasContainer.appendChild(renderer.domElement);

		// Add lights
		const ambientLight = new THREE.AmbientLight(0x404040, 0.6);
		scene.add(ambientLight);

		const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
		directionalLight.position.set(10, 10, 5);
		scene.add(directionalLight);

		// Add orbit controls
		const controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;

		// Load and play the GLB model
		const loader = new GLTFLoader();
		loader.load('/curved1.glb', (gltf) => {
			const model = gltf.scene;
			scene.add(model);

			// Set up animations
			if (gltf.animations.length > 0) {
				animationClips = gltf.animations;
				mixer = new THREE.AnimationMixer(model);
				// Don't play automatically - start at first frame
				gltf.animations.forEach((clip) => {
					const action = mixer!.clipAction(clip);
					action.time = 0;
					action.paused = true;
					action.play();
				});
			}

			// Center and scale the model
			const box = new THREE.Box3().setFromObject(model);
			const center = box.getCenter(new THREE.Vector3());
			const size = box.getSize(new THREE.Vector3());
			const maxDim = Math.max(size.x, size.y, size.z);
			const scale = 2 / maxDim;

			model.scale.setScalar(scale);
			model.position.sub(center.multiplyScalar(scale));
		});

		// Animation loop
		clock = new THREE.Clock();
		function animate() {
			animationId = requestAnimationFrame(animate);

			if (mixer) {
				mixer.update(clock.getDelta());
			}

			controls.update();
			renderer.render(scene, camera);
		}
		animate();

		// Handle resize
		window.addEventListener('resize', () => {
			camera.aspect = window.innerWidth / window.innerHeight;
			camera.updateProjectionMatrix();
			renderer.setSize(window.innerWidth, window.innerHeight);
		});
	});

	// Function to animate to a specific frame
	function animateToFrame(targetFrame: number) {
		if (!mixer || animationClips.length === 0 || isAnimating) return;

		isAnimating = true;

		animationClips.forEach((clip) => {
			const action = mixer!.clipAction(clip);
			const startTime = action.time;
			const endTime = (targetFrame - 1) * (clip.duration / 20); // Assuming 20 total frames

			// Create a smooth transition
			const duration = 1.0; // 1 second transition
			const startTimeStamp = performance.now();

			function updateFrame() {
				const elapsed = (performance.now() - startTimeStamp) / 1000;
				const progress = Math.min(elapsed / duration, 1);

				// Ease in-out function for smooth animation
				const easeProgress =
					progress < 0.5 ? 2 * progress * progress : 1 - Math.pow(-2 * progress + 2, 3) / 2;

				action.time = startTime + (endTime - startTime) * easeProgress;

				if (progress < 1) {
					requestAnimationFrame(updateFrame);
				} else {
					currentFrame = targetFrame;
					isAnimating = false;
				}
			}

			updateFrame();
		});
	}

	onDestroy(() => {
		if (!browser) return;
		if (animationId) cancelAnimationFrame(animationId);
		if (renderer) renderer.dispose();
	});
</script>

<div class="relative m-0 h-screen w-screen overflow-hidden p-0">
	<div bind:this={canvasContainer} class="h-full w-full"></div>
	<div class="absolute top-5 left-5 z-[100] flex gap-2.5">
		<button
			on:click={() => animateToFrame(1)}
			class="cursor-pointer rounded-lg border-2 px-5 py-2.5 text-sm font-semibold text-white backdrop-blur-sm transition-all duration-300
				{currentFrame === 1
				? 'border-green-500 bg-green-500/90 hover:bg-green-500'
				: 'border-blue-500 bg-blue-500/90 hover:-translate-y-0.5 hover:bg-blue-500 hover:shadow-lg'}
				{isAnimating ? 'cursor-not-allowed opacity-50' : ''}"
			disabled={isAnimating}
		>
			Frame 1
		</button>
		<button
			on:click={() => animateToFrame(10)}
			class="cursor-pointer rounded-lg border-2 px-5 py-2.5 text-sm font-semibold text-white backdrop-blur-sm transition-all duration-300
				{currentFrame === 10
				? 'border-green-500 bg-green-500/90 hover:bg-green-500'
				: 'border-blue-500 bg-blue-500/90 hover:-translate-y-0.5 hover:bg-blue-500 hover:shadow-lg'}
				{isAnimating ? 'cursor-not-allowed opacity-50' : ''}"
			disabled={isAnimating}
		>
			Frame 10
		</button>
		<button
			on:click={() => animateToFrame(20)}
			class="cursor-pointer rounded-lg border-2 px-5 py-2.5 text-sm font-semibold text-white backdrop-blur-sm transition-all duration-300
				{currentFrame === 20
				? 'border-green-500 bg-green-500/90 hover:bg-green-500'
				: 'border-blue-500 bg-blue-500/90 hover:-translate-y-0.5 hover:bg-blue-500 hover:shadow-lg'}
				{isAnimating ? 'cursor-not-allowed opacity-50' : ''}"
			disabled={isAnimating}
		>
			Frame 20
		</button>
	</div>
</div>
