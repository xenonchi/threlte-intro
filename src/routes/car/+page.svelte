<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

	let canvasContainer: HTMLDivElement;
	let renderer: THREE.WebGLRenderer;
	let scene: THREE.Scene;
	let camera: THREE.Camera;
	let mixer: THREE.AnimationMixer | null = null;
	let clock: THREE.Clock;
	let animationId: number;
	let animationClips: THREE.AnimationClip[] = [];
	let currentFrame: number = 1;
	let isAnimating: boolean = false;
	let gltfCamera: THREE.Camera | null = null;
	let doorState: 'open' | 'closed' = 'closed';

	onMount(() => {
		if (!browser) return;

		// Create scene
		scene = new THREE.Scene();
		scene.background = new THREE.Color(0x87ceeb);

		// Create a temporary camera for initial setup (will be replaced by GLB camera)
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
		loader.load('/car.glb', (gltf) => {
			const model = gltf.scene;
			scene.add(model);

			// Look for camera in the GLB file
			model.traverse((child) => {
				if (child instanceof THREE.Camera) {
					gltfCamera = child;
					camera = child; // Use the GLB camera
					console.log('Found camera in GLB file:', child);
				}
			});

			// If no camera found in GLB, keep the default one
			if (!gltfCamera) {
				console.log('No camera found in GLB file, using default camera');
			}

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
			// Only update aspect ratio for PerspectiveCamera
			if (camera instanceof THREE.PerspectiveCamera) {
				camera.aspect = window.innerWidth / window.innerHeight;
				camera.updateProjectionMatrix();
			}
			renderer.setSize(window.innerWidth, window.innerHeight);
		});
	});

	// Function to animate through a range of frames
	function animateFrameRange(startFrame: number, endFrame: number) {
		if (!mixer || animationClips.length === 0 || isAnimating) return;

		isAnimating = true;

		animationClips.forEach((clip) => {
			const action = mixer!.clipAction(clip);
			const startTime = (startFrame - 1) * (clip.duration / 40); // Assuming 40 total frames
			const endTime = (endFrame - 1) * (clip.duration / 40);

			// Set initial time to start frame
			action.time = startTime;

			// Create a smooth transition through the frame range
			const duration = 2.0; // 2 second transition
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
					currentFrame = endFrame;
					isAnimating = false;
					// Update door state based on final frame
					if (endFrame <= 21) {
						doorState = 'open';
					} else if (endFrame >= 30) {
						doorState = 'closed';
					}
				}
			}

			updateFrame();
		});
	}

	// Function to open door (frames 1-21)
	function openDoor() {
		if (doorState === 'closed' && !isAnimating) {
			animateFrameRange(1, 21);
			// If using GLB camera, it should already be positioned correctly
			// If using default camera, position it for door viewing
			if (!gltfCamera) {
				camera.position.set(3, 1.5, 2);
				if (camera instanceof THREE.PerspectiveCamera) {
					camera.lookAt(0, 0, 0);
				}
			}
		}
	}

	// Function to close door (frames 30-41)
	function closeDoor() {
		if (doorState === 'open' && !isAnimating) {
			animateFrameRange(30, 41);
			// If using GLB camera, it should already be positioned correctly
			// If using default camera, position it for door viewing
			if (!gltfCamera) {
				camera.position.set(3, 1.5, 2);
				if (camera instanceof THREE.PerspectiveCamera) {
					camera.lookAt(0, 0, 0);
				}
			}
		}
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
			on:click={openDoor}
			class="rounded-lg border-2 px-5 py-2.5 text-sm font-semibold text-white backdrop-blur-sm transition-all duration-300
				{doorState === 'closed' && !isAnimating
					? 'cursor-pointer border-green-500 bg-green-500/90 hover:-translate-y-0.5 hover:bg-green-500 hover:shadow-lg'
					: 'cursor-not-allowed border-gray-500 bg-gray-500/50 opacity-50'}"
			disabled={doorState !== 'closed' || isAnimating}
		>
			Open Door
		</button>
		<button
			on:click={closeDoor}
			class="rounded-lg border-2 px-5 py-2.5 text-sm font-semibold text-white backdrop-blur-sm transition-all duration-300
				{doorState === 'open' && !isAnimating
					? 'cursor-pointer border-red-500 bg-red-500/90 hover:-translate-y-0.5 hover:bg-red-500 hover:shadow-lg'
					: 'cursor-not-allowed border-gray-500 bg-gray-500/50 opacity-50'}"
			disabled={doorState !== 'open' || isAnimating}
		>
			Close Door
		</button>
	</div>
</div>
