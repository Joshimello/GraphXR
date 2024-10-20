<script lang="ts">
  import * as THREE from 'three'
  import { T, useTask, useThrelte } from '@threlte/core'
  import { Text } from '@threlte/extras'
  import { Controller, type XRControllerEvent, useController, pointerControls, useXR } from '@threlte/xr'
  import { text } from './stores'
  import { MeshLineGeometry, MeshLineMaterial, raycast } from 'meshline'
	import { onMount } from 'svelte';
  import { OpenAI } from 'openai';
	import { PUBLIC_OPENAI } from '$env/static/public';

  let isDrawing = false
  let points: THREE.Vector3[] = []
  let meshes: THREE.Mesh[] = []
  let strokes: THREE.Vector3[][] = []
  let line: THREE.Line | null = null

  const lineMaterial = new THREE.LineBasicMaterial({ color: 0xffffff })
  let lineGeometry = new THREE.BufferGeometry()

  const { scene, renderer, camera } = useThrelte()

  pointerControls('right')

  const handleStart = (event: XRControllerEvent) => {
    isDrawing = true

    points = []
    lineGeometry = new THREE.BufferGeometry()
    const positions = new Float32Array(0)
    lineGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3))
    line = new THREE.Line(lineGeometry, lineMaterial)
  }

  const handleEnd = (event: XRControllerEvent) => {
    isDrawing = false

    const curve = new THREE.CatmullRomCurve3(points)
    const geometry = new MeshLineGeometry()
    geometry.setPoints(curve.getPoints(50))
    const material = new MeshLineMaterial({ color: 0xffffff, lineWidth: 0.001, resolution: new THREE.Vector2(100, 100) })
    const meshLine = new THREE.Mesh(geometry, material)
    meshes = [...meshes, meshLine]
    strokes = [...strokes, points]

    line = null
    points = []
  }

  let renderTarget: THREE.WebGLRenderTarget

  onMount(() => {
    const size = renderer.getSize(new THREE.Vector2())
    renderTarget = new THREE.WebGLRenderTarget(size.width, size.height)
  })

  const handleSqueezeStart = (event: XRControllerEvent) => {
  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d');
  canvas.width = 800;
  canvas.height = 600;

  const scale = 400;

  if (!context) return;

  // Initialize bounding box values
  let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity;

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
    let mappedY = (1 - (firstPoint.y)) * scale + canvasCenterY;

    context.moveTo(mappedX, mappedY);

    stroke.forEach((vec3) => {
      let x = (vec3.x - bboxCenterX) * scale + canvasCenterX;
      let y = (1 - (vec3.y)) * scale + canvasCenterY;
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
            { type: 'text', text: 'parse the formula to a latex formula, for example, if i wrote "sin(x+y)", please output "\\sin(x+y)", without any other additional text, ONLY the formula. ' },
            { type: 'image_url', image_url: { url: `data:image/png;base64,${base64Image}` } }
          ]
        }
      ]
    })

    text.set(response.choices[0].message.content ?? 'No response')
  })()

};


  const handleSqueezeEnd = (event: XRControllerEvent) => {

  }

  const controller = useController($$restProps.left ? 'left' : 'right')

  useTask(() => {
    if (isDrawing) {
      const ray = controller.current?.targetRay
      const { x = 0, y = 0, z = 0 } = ray?.position ?? {}
      const point = new THREE.Vector3(x, y, z)
      points.push(point)

      const positions = new Float32Array(points.length * 3)
      points.forEach((p, i) => {
        positions[i * 3] = p.x
        positions[i * 3 + 1] = p.y
        positions[i * 3 + 2] = p.z
      })
      lineGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3))
      lineGeometry.attributes.position.needsUpdate = true
    }
  })

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
    <Text text={$text} anchorX="right" fontSize={0.02} rotation={[-Math.PI/2, 0, -Math.PI/2]} position={[-0.005, 0, 0.16]} />
  </T.Group>

  <T.Mesh slot="pointer-ray" position={[0, -0.001, 0]}>
    <T.SphereGeometry args={[0.002, 32, 32]} />
    <T.MeshStandardMaterial color={0xffffff} />
  </T.Mesh>

  <T.Mesh slot="pointer-cursor">

  </T.Mesh>

</Controller>

{#if line}
  <T is={line} />
{/if}

{#each meshes as mesh, index (index)}
  <T is={mesh} />
{/each}