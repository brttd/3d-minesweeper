<script>
	import '../app.css';

	import * as THREE from 'three';

	import { onMount } from 'svelte';

	let main;
	let renderer;

	const scene = new THREE.Scene();

	const cubeGroup = new THREE.Group();
	const cubeOffset = new THREE.Group();

	const cubes = [];

	const cameraFOV = 45;
	const camera = new THREE.PerspectiveCamera(cameraFOV, 1);

	const config = {
		size: 5
	};

	const cubeMesh = new THREE.BoxGeometry(0.8, 0.8, 0.8);

	const cubeMaterial = new THREE.MeshStandardMaterial({
		color: 0xff0000
	});

	let needsResize = true;
	let width = 0;
	let height = 0;

	const rotateAxisX = new THREE.Vector3(0, 1, 0);
	const rotateAxisY = new THREE.Vector3(1, 0, 0);

	const mouse = new THREE.Vector2();
	const mousePrev = new THREE.Vector2();
	const touch = [new THREE.Vector2(), new THREE.Vector2()];
	const touchPrev = [new THREE.Vector2(), new THREE.Vector2()];

	const maxTouches = 2;

	let touchCount = 0;
	let mouseDown = false;

	let mouseChanged = false;
	let touchChanged = false;

	const mouseAnglePerPixel = 0.18 * THREE.MathUtils.DEG2RAD;

	function render() {
		requestAnimationFrame(render);

		if (needsResize) {
			needsResize = false;

			resize();
		}

		if (mouseChanged) {
			mouseChanged = false;

			if (mouseDown) {
				cubeGroup.rotateOnWorldAxis(rotateAxisX, (mouse.x - mousePrev.x) * mouseAnglePerPixel);
				cubeGroup.rotateOnWorldAxis(rotateAxisY, (mouse.y - mousePrev.y) * mouseAnglePerPixel);
			}

			mousePrev.copy(mouse);
		}

		if (touchChanged) {
			touchChanged = false;

			if (touchCount === 1) {
				//Single touch - rotate
				cubeGroup.rotateOnWorldAxis(
					rotateAxisX,
					(touch[0].x - touchPrev[0].x) * mouseAnglePerPixel
				);
				cubeGroup.rotateOnWorldAxis(
					rotateAxisY,
					(touch[0].y - touchPrev[0].y) * mouseAnglePerPixel
				);
			} else if (touchCount === 2) {
				//Double touch - pan/zoom
			}

			touchPrev[0].copy(touch[0]);
			touchPrev[1].copy(touch[1]);
		}

		renderer.render(scene, camera);
	}

	function resize() {
		width = window.innerWidth;
		height = window.innerHeight;

		renderer.setSize(width, height);

		camera.aspect = width / height;

		if (width < height) {
			camera.fov =
				(Math.atan(Math.tan((cameraFOV * Math.PI) / 360) / camera.aspect) * 360) / Math.PI;
		} else {
			camera.fov = cameraFOV;
		}

		camera.updateProjectionMatrix();
	}

	function requestResize() {
		needsResize = true;
	}

	function onMouseMove(event) {
		if (mouse.x === 0 && mouse.y === 0 && mousePrev.x === 0 && mousePrev.y === 0) {
			mousePrev.set(event.clientX, event.clientY);
		} else {
			mouseChanged = true;
		}

		mouse.set(event.clientX, event.clientY);
	}

	function onMouseDown(event) {
		//TODO...
		mouseDown = true;
	}
	function onMouseUp(event) {
		//TODO...
		mouseDown = false;
	}

	function onTouchMove(event) {
		if (touch[0].x === 0 && touch[0].y === 0 && touchPrev[0].x === 0 && touchPrev[0].y === 0) {
			touchPrev[0].set(event.touches[0].clientX, event.touches[0].clientY);
		} else {
			if (touchCount !== Math.min(maxTouches, event.touches.length)) {
				touchChanged = false;
			} else {
				touchChanged = true;
			}
		}

		for (let i = 0; i < event.touches.length && i < maxTouches; i++) {
			touch[i].set(event.touches[i].clientX, event.touches[i].clientY);
		}

		touchCount = Math.min(maxTouches, event.touches.length);
	}
	function onTouchStart(event) {
		//When a new touch starts
		for (let i = 0; i < event.touches.length && i < maxTouches; i++) {
			touchPrev[i].set(event.touches[i].clientX, event.touches[i].clientY);
		}

		touchChanged = false;
	}
	function onTouchEnd(event) {
		touchChanged = false;

		if (event.touches.length === 0) {
			touch[0].set(0, 0);
			touch[1].set(0, 0);

			touchPrev[0].set(0, 0);
			touchPrev[1].set(0, 0);
		}
	}

	function onBlur() {
		//TODO...
		mouseDown = false;

		touchCount = 0;

		mouse.set(0, 0);
		mousePrev.set(0, 0);

		for (let i = 0; i < touch.length; i++) {
			touch[i].set(0, 0);
			touchPrev[i].set(0, 0);
		}
	}

	function createCube() {
		cubeOffset.position.set(
			-((config.size - 1) / 2),
			-((config.size - 1) / 2),
			-((config.size - 1) / 2)
		);

		for (let x = 0; x < config.size; x++) {
			for (let y = 0; y < config.size; y++) {
				for (let z = 0; z < config.size; z++) {
					const cube = new THREE.Mesh(cubeMesh, cubeMaterial);

					cube.position.set(x, y, z);

					cubeOffset.add(cube);
				}
			}
		}

		camera.position.set(0, 0, config.size * 2);
		camera.lookAt(0, 0, 0);
	}

	function init() {
		//Setup lights
		const mainLight = new THREE.DirectionalLight(0xffffff, 0.8);
		mainLight.position.set(1, 1, 1);
		mainLight.lookAt(0, 0, 0);
		scene.add(mainLight);

		const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
		scene.add(ambientLight);

		//Setup cube
		cubeGroup.add(cubeOffset);
		scene.add(cubeGroup);

		createCube();

		//Setup renderer
		renderer = new THREE.WebGLRenderer();
		renderer.setPixelRatio(window.devicePixelRatio ? window.devicePixelRatio : 1);

		needsResize = true;
		window.addEventListener('resize', requestResize);

		window.addEventListener('mousemove', onMouseMove);
		window.addEventListener('mousedown', onMouseDown);
		window.addEventListener('mouseup', onMouseUp);

		window.addEventListener('touchstart', onTouchStart);
		window.addEventListener('touchmove', onTouchMove);
		window.addEventListener('touchend', onTouchEnd);
		window.addEventListener('touchcancel', onTouchEnd);

		window.addEventListener('blur', onBlur);

		render();
		main.appendChild(renderer.domElement);
	}

	onMount(init);
</script>

<div bind:this={main}></div>
