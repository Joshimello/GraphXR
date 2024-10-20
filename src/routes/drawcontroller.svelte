<script lang="ts">
	import * as THREE from 'three';
	import { T, useTask, useThrelte } from '@threlte/core';
	import { Text } from '@threlte/extras';
	import {
		Controller,
		type XRControllerEvent,
		useController,
		pointerControls,
		useXR
	} from '@threlte/xr';
	import { text } from './stores';
	import { MeshLineGeometry, MeshLineMaterial, raycast } from 'meshline';
	import { onMount } from 'svelte';
	import { OpenAI } from 'openai';
	import { PUBLIC_OPENAI } from '$env/static/public';
	import { ComputeEngine } from '@cortex-js/compute-engine';

	let isDrawing = false;
	let points: THREE.Vector3[] = [];
	let meshes: THREE.Mesh[] = [];
	let strokes: THREE.Vector3[][] = [];
	let line: THREE.Line | null = null;

  const reset = () => {
    meshes = [];
    strokes = [];
  };

	const lineMaterial = new THREE.LineBasicMaterial({ color: 0xffffff });
	let lineGeometry = new THREE.BufferGeometry();

	const { scene, renderer, camera } = useThrelte();

	pointerControls('right');

	const handleStart = (event: XRControllerEvent) => {
		isDrawing = true;

		points = [];
		lineGeometry = new THREE.BufferGeometry();
		const positions = new Float32Array(0);
		lineGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
		line = new THREE.Line(lineGeometry, lineMaterial);
	};

	const handleEnd = (event: XRControllerEvent) => {
		isDrawing = false;

		const curve = new THREE.CatmullRomCurve3(points);
		const geometry = new MeshLineGeometry();
		geometry.setPoints(curve.getPoints(50));
		const material = new MeshLineMaterial({
			color: 0xffffff,
			lineWidth: 0.001,
			resolution: new THREE.Vector2(100, 100)
		});
		const meshLine = new THREE.Mesh(geometry, material);
		meshes = [...meshes, meshLine];
		strokes = [...strokes, points];

		line = null;
		points = [];
	};

	let renderTarget: THREE.WebGLRenderTarget;

	onMount(() => {
		const size = renderer.getSize(new THREE.Vector2());
		renderTarget = new THREE.WebGLRenderTarget(size.width, size.height);
	});

  let squeezeStart: number | null = null;

	const handleSqueezeStart = (event: XRControllerEvent) => {
    squeezeStart = Date.now();
  };


	const handleSqueezeEnd = (event: XRControllerEvent) => {
		if (squeezeStart && Date.now() - squeezeStart > 1000) {
			const canvas = document.createElement('canvas');
			const context = canvas.getContext('2d');
			canvas.width = 800;
			canvas.height = 600;

			const scale = 400;

			if (!context) return;

			// Initialize bounding box values
			let minX = Infinity,
				maxX = -Infinity,
				minY = Infinity,
				maxY = -Infinity;

			// First, find the bounding box of all the strokes
			strokes.forEach((stroke) => {
				stroke.forEach((vec3) => {
					minX = Math.min(minX, vec3.x);
					maxX = Math.max(maxX, vec3.x);
					minY = Math.min(minY, vec3.y);
					maxY = Math.max(maxY, vec3.y);
				});
			});

			// Calculate the center of the bounding box
			const bboxWidth = maxX - minX;
			const bboxHeight = maxY - minY;
			const bboxCenterX = minX + bboxWidth / 2;
			const bboxCenterY = minY + bboxHeight / 2;

			// Translate points so that the center of the bounding box is in the middle of the canvas
			const canvasCenterX = canvas.width / 2;
			const canvasCenterY = canvas.height / 2;

			strokes.forEach((stroke) => {
				if (stroke.length < 2) return;

				context.beginPath();

				// Get the first point and map its position
				let firstPoint = stroke[0];
				let mappedX = (firstPoint.x - bboxCenterX) * scale + canvasCenterX;
				let mappedY = (1 - firstPoint.y) * scale + canvasCenterY;

				context.moveTo(mappedX, mappedY);

				stroke.forEach((vec3) => {
					let x = (vec3.x - bboxCenterX) * scale + canvasCenterX;
					let y = (1 - vec3.y) * scale + canvasCenterY;
					context.lineTo(x, y);
				});

				context.strokeStyle = 'white'; // Set stroke color
				context.lineWidth = 2; // Set line width
				context.stroke(); // Draw the path
			});

			// Generate a downloadable image
			// const url = canvas.toDataURL('image/png');
			// const a = document.createElement('a');
			// a.href = url;
			// a.download = 'test.png';
			// a.click();

			const openai = new OpenAI({
				apiKey: PUBLIC_OPENAI,
				dangerouslyAllowBrowser: true
			});

			// convert image to base64
			const data = canvas.toDataURL('image/png');
			const base64Image = data.split(';base64,').pop();

			(async () => {
				const response = await openai.chat.completions.create({
					model: 'gpt-4o',
					messages: [
						{
							role: 'user',
							content: [
								{
									type: 'text',
									text: 'parse the formula to a latex formula, for example, if i wrote "sin(x+y)", please output "\\sin(x+y)", without any other additional text, ONLY the formula. '
								},
								{ type: 'image_url', image_url: { url: `data:image/png;base64,${base64Image}` } }
							]
						}
					]
				});

				text.set(response.choices[0].message.content ?? 'No response');
				changeMesh(response.choices[0].message.content ?? 'x+y');
			})();
		}
    else {
      reset();
    }
	};

	const controller = useController($$restProps.left ? 'left' : 'right');

	useTask(() => {
		if (isDrawing) {
			const ray = controller.current?.targetRay;
			const { x = 0, y = 0, z = 0 } = ray?.position ?? {};
			const point = new THREE.Vector3(x, y, z);
			points.push(point);

			const positions = new Float32Array(points.length * 3);
			points.forEach((p, i) => {
				positions[i * 3] = p.x;
				positions[i * 3 + 1] = p.y;
				positions[i * 3 + 2] = p.z;
			});
			lineGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
			lineGeometry.attributes.position.needsUpdate = true;
		}
	});

	const vertexShader = `
    varying float vX;  
    varying float vY;  
    varying float vZ;  
    void main() {
        vec4 modelPosition = vec4(position, 1.0);  // Use model local coordinates
        vZ = modelPosition.z;
        vX = modelPosition.x;
        vY = modelPosition.y;
        gl_Position = projectionMatrix * modelViewMatrix * modelPosition;
    }
    `;
	const fragmentShader = `
    varying float vX;  
    varying float vY;  
    varying float vZ;  
    uniform bool useRed;
    uniform bool useGreen;
    uniform bool useBlue;
    uniform float initRed;
    uniform float initGreen;
    uniform float initBlue;
    uniform float maxZ;
    uniform float minZ;
    void main() {
        float red = initRed;
        float green = initGreen;
        float blue = initBlue;
        float interval = (maxZ - vY) / (maxZ - minZ);
        if (interval < 0.5) {
            red = 1.0 - 2.0 * interval; // Red fades out
            green = 2.0 * interval;     // Green fades in
            blue = 0.0;                 // Blue stays at 0
        } else {
            red = 0.0;                  // Red stays at 0
            green = 2.0 * (1.0 - interval); // Green fades out
            blue = 2.0 * (interval - 0.5);  // Blue fades in
        }
        gl_FragColor = vec4(red, green, blue, 1.0);
        // if(vZ > 10.0 || vZ < -10.0){
        //     gl_FragColor = vec4(1.0, 1.0, 1.0, 0.0);
        // }
        if(maxZ == minZ){
            gl_FragColor = vec4(1.0, 0.0, 1.0, 1.0);
        }
    }
    `;
	const ce = new ComputeEngine();
	const size = 10;
	const splitter = 64;
	const offsetY = 0.5;
	let formula = '';

	function createGraph() {
		console.log('Creating Mesh ...');
		const geometry = new THREE.PlaneGeometry(size, size, splitter, splitter);
		const materialGrid = new THREE.MeshBasicMaterial({
			color: 0x000000, // Hex code for black
			wireframe: true,
			opacity: 0.05,
			transparent: true
		});
		var useRed: boolean = Math.random() < 0.5;
		var useGreen: boolean = Math.random() < 0.5;
		var useBlue: boolean = useRed || useGreen ? Math.random() < 0.5 : true;
		var maxZ = -Infinity;
		var minZ = Infinity;
		const material = new THREE.ShaderMaterial({
			vertexShader,
			fragmentShader,
			side: THREE.DoubleSide,
			transparent: true,
			uniforms: {
				maxZ: { value: 0 },
				minZ: { value: 0 },
				useRed: { value: useRed },
				initRed: { value: useRed ? 0.2 : 0.8 },
				useGreen: { value: useGreen },
				initGreen: { value: useRed ? 0.2 : 0.8 },
				useBlue: { value: useBlue },
				initBlue: { value: useRed ? 0.2 : 0.8 }
			}
		});
		const mesh = new THREE.Mesh(geometry, material);
		const mesh2 = new THREE.Mesh(geometry, materialGrid);
		mesh.castShadow = true; // This mesh casts shadows
		mesh.receiveShadow = true;
		mesh.name = formula;
		scene.add(mesh);
		// scene.add(mesh2);
		// Modify vertex positions
		const positions = geometry.attributes.position;
		const vertexCount = positions.count;
		for (let i = 0; i < vertexCount; i++) {
			const x = positions.getX(i);
			const y = positions.getY(i);
			ce.assign('x', x);
			ce.assign('y', y);
			const z = Number(ce.parse(formula).N().value);
			positions.setXYZ(i, x, z, y);
			// positions.setZ(i, z);
			if (z > maxZ) {
				maxZ = z;
			}
			if (z < minZ) {
				minZ = z;
			}
		}
		// geometry.scale(1.0, 1.0, size/(maxZ * 2))
		const scale = 0.3;
		material.uniforms.maxZ.value = maxZ * scale + offsetY;
		material.uniforms.minZ.value = minZ * scale + offsetY;
		geometry.scale(scale, scale, scale);
		geometry.translate(2, offsetY, 0);
		positions.needsUpdate = true;
		console.log('Finish Mesh ...');
		return mesh;
	}
	function addGridHelper() {
		const divisions = size; // One line per unit
		const gridHelper = new THREE.GridHelper(size * 0.3, divisions, 0x888888, 0x888888);
		gridHelper.position.y = 0; // Align it with the base of your mesh if needed
		// gridHelper.rotateX(Math.PI / 2);
		gridHelper.translateX(2);
		gridHelper.translateY(offsetY);
		scene.add(gridHelper);
	}
	addGridHelper();
	// To set a new formula, do:
	formula = '\\sin(x) * \\cos(y)';
	console.log(formula);
	var curMesh = createGraph();
	function changeMesh(newFormula: string) {
		scene.remove(curMesh);
		curMesh.geometry.dispose();
		curMesh.material.dispose();
		formula = newFormula;
		curMesh = createGraph();
	}
</script>

<Controller
	left={$$props.left}
	right={$$props.right}
	on:selectstart={handleStart}
	on:selectend={handleEnd}
	on:squeezestart={handleSqueezeStart}
	on:squeezeend={handleSqueezeEnd}
>
	<T.Group slot="target-ray">
		<Text
			text={$text}
			anchorX="right"
			fontSize={0.02}
			rotation={[-Math.PI / 2, 0, -Math.PI / 2]}
			position={[-0.005, 0, 0.16]}
		/>
	</T.Group>

	<T.Mesh slot="pointer-ray" position={[0, -0.001, 0]}>
		<T.SphereGeometry args={[0.002, 32, 32]} />
		<T.MeshStandardMaterial color={0xffffff} />
	</T.Mesh>

	<T.Mesh slot="pointer-cursor"></T.Mesh>
</Controller>

{#if line}
	<T is={line} />
{/if}

{#each meshes as mesh, index (index)}
	<T is={mesh} />
{/each}
