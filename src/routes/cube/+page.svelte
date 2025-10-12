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
	let glbCamera: THREE.Camera | null = null;
	let mixer: THREE.AnimationMixer | null = null;
	let clock: THREE.Clock;
	let animationId: number;
	let animationClips: THREE.AnimationClip[] = [];
	let currentFrame: number = 1;
	let isAnimating: boolean = false;
	let animationSpeed: number = 1.0; // 1.0 = normal speed, 2.0 = double speed, 0.5 = half speed
	let isPlaying: boolean = false;
	let showCameraView: boolean = false;
	let useGlbCamera: boolean = false;
	let cameraViewContainer: HTMLDivElement;
	let cameraRenderer: THREE.WebGLRenderer;
	let controls: OrbitControls | null = null;
	let glbCameraZoom: number = 1.0;

	onMount(() => {
		if (!browser) return;

		// Create scene
		scene = new THREE.Scene();
		scene.background = new THREE.Color(0x87ceeb);

		// Create camera
		camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
		camera.position.set(2, 2, 2);

		// Create GLB camera view renderer (container sized)
		cameraRenderer = new THREE.WebGLRenderer({ antialias: true });
		cameraRenderer.setSize(800, 600); // Initial size, will be updated by container
		cameraRenderer.domElement.style.width = '100%';
		cameraRenderer.domElement.style.height = '100%';
		cameraRenderer.domElement.style.display = 'block';
		cameraRenderer.domElement.style.borderRadius = '8px';
		canvasContainer.appendChild(cameraRenderer.domElement);

		// Add lights
		const ambientLight = new THREE.AmbientLight(0x404040, 0.6);
		scene.add(ambientLight);

		const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
		directionalLight.position.set(10, 10, 5);
		scene.add(directionalLight);

		// No orbit controls needed - we'll only use GLB camera

		// Load and play the GLB model
		const loader = new GLTFLoader();
		loader.load('/cube-animation.glb', (gltf) => {
			const model = gltf.scene;
			scene.add(model);

			// Extract camera from GLB file
			gltf.scene.traverse((child) => {
				if (child instanceof THREE.Camera) {
					glbCamera = child;
					console.log('Found camera in GLB:', child);
					// Store original zoom/fov for reference
					if (child instanceof THREE.PerspectiveCamera) {
						console.log('GLB Camera FOV:', child.fov);
					}
					// Automatically use GLB camera
					useGlbCamera = true;
				}
			});

			// Set up animations
			if (gltf.animations.length > 0) {
				animationClips = gltf.animations;
				mixer = new THREE.AnimationMixer(model);
				// Set up animations - start paused at first frame
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
				mixer.update(clock.getDelta() * animationSpeed);
			}

			// Only render GLB camera view
			if (glbCamera && cameraRenderer) {
				cameraRenderer.render(scene, glbCamera);
			}
		}
		animate();

		// Handle resize
		const handleResize = () => {
			if (canvasContainer && cameraRenderer) {
				const rect = canvasContainer.getBoundingClientRect();
				cameraRenderer.setSize(rect.width, rect.height);
				if (glbCamera && glbCamera instanceof THREE.PerspectiveCamera) {
					glbCamera.aspect = rect.width / rect.height;
					glbCamera.updateProjectionMatrix();
				}
			}
		};

		window.addEventListener('resize', handleResize);

		// Initial resize to set proper size
		setTimeout(handleResize, 100);

		// Handle keyboard shortcuts for GLB camera zoom
		window.addEventListener('keydown', (event) => {
			if (glbCamera) {
				if (event.key === '+' || event.key === '=') {
					event.preventDefault();
					adjustGlbCameraZoom(0.2);
				} else if (event.key === '-') {
					event.preventDefault();
					adjustGlbCameraZoom(-0.2);
				}
			}
		});
	});

	// Function to toggle play/pause
	function togglePlayPause() {
		if (!mixer || animationClips.length === 0) return;

		isPlaying = !isPlaying;
		animationClips.forEach((clip) => {
			const action = mixer!.clipAction(clip);
			action.paused = !isPlaying;
		});
	}

	// Function to toggle camera view
	function toggleCameraView() {
		showCameraView = !showCameraView;
		if (cameraRenderer) {
			cameraRenderer.domElement.style.display = showCameraView ? 'block' : 'none';
		}
	}

	// Function to toggle between GLB camera and default camera
	function toggleGlbCamera() {
		useGlbCamera = !useGlbCamera;
		if (controls) {
			if (useGlbCamera && glbCamera) {
				console.log('Switched to GLB camera');
				// Update orbit controls to use GLB camera
				controls.object = glbCamera;
			} else {
				console.log('Switched to default camera');
				// Update orbit controls to use default camera
				controls.object = camera;
			}
		}
	}

	// Function to adjust GLB camera zoom
	function adjustGlbCameraZoom(zoomDelta: number) {
		if (glbCamera && glbCamera instanceof THREE.PerspectiveCamera) {
			glbCameraZoom = Math.max(0.1, Math.min(5.0, glbCameraZoom + zoomDelta));
			glbCamera.fov = 75 / glbCameraZoom; // Lower FOV = more zoomed in
			glbCamera.updateProjectionMatrix();
			console.log('GLB Camera zoom:', glbCameraZoom.toFixed(2), 'FOV:', glbCamera.fov.toFixed(2));
		}
	}

	// Function to animate to a specific frame
	function animateToFrame(targetFrame: number) {
		if (!mixer || animationClips.length === 0 || isAnimating) return;

		// Pause automatic playback when jumping to frames
		if (isPlaying) {
			togglePlayPause();
		}

		isAnimating = true;

		animationClips.forEach((clip) => {
			const action = mixer!.clipAction(clip);
			const startTime = action.time;
			const endTime = (targetFrame - 1) * (clip.duration / 135); // Assuming 20 total frames

			// Create a smooth transition
			const duration = 0.5; // 0.5 second transition (faster)
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
		if (cameraRenderer) {
			cameraRenderer.dispose();
			if (cameraRenderer.domElement.parentNode) {
				cameraRenderer.domElement.parentNode.removeChild(cameraRenderer.domElement);
			}
		}
	});
</script>

<!-- GLB Camera View with Play Button -->
<div class="relative m-0 flex h-screen w-screen items-center justify-center overflow-hidden p-0">
	<div
		bind:this={canvasContainer}
		class="relative h-3/4 w-3/4 rounded-lg"
		style="mask-image: radial-gradient(circle, rgba(0,0,0,1) 75%, rgba(0,0,0,0) 90%);"
	>
		<div class="pointer-events-none absolute inset-0 rounded-lg"></div>
	</div>

	<!-- Play/Pause Button -->
	<div class="absolute top-5 left-5 z-[100]">
		<button
			on:click={togglePlayPause}
			class="cursor-pointer rounded-lg border-2 px-6 py-3 text-lg font-semibold text-white shadow-lg backdrop-blur-sm transition-all duration-300
				{isPlaying
				? 'border-red-500 bg-red-500/90 hover:scale-105 hover:bg-red-500'
				: 'border-green-500 bg-green-500/90 hover:scale-105 hover:bg-green-500'}"
		>
			{isPlaying ? '⏸️ Pause' : '▶️ Play'}
		</button>
	</div>
</div>
