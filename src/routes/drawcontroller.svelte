<script lang="ts">
  import * as THREE from 'three'
  import { T, useTask, useThrelte } from '@threlte/core'
  import { Text } from '@threlte/extras'
  import { Controller, type XRControllerEvent, useController, pointerControls, useXR } from '@threlte/xr'
  import { text } from './stores'
  import { MeshLineGeometry, MeshLineMaterial, raycast } from 'meshline'
	import { onMount } from 'svelte';

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
    // const offscreenCanvas = document.createElement('canvas');
    // offscreenCanvas.width = 1920;
    // offscreenCanvas.height = 1080;
    // const offscreenContext = offscreenCanvas.getContext('2d');
    // renderer.render(scene, camera.current);
    // offscreenContext?.drawImage(renderer.domElement, 0, 0, 1920, 1080);
    // offscreenCanvas.toBlob((blob) => {
    //   const url = URL.createObjectURL(blob);
    //   const a = document.createElement('a');
    //   a.href = url;
    //   a.download = 'dawdasd.png';
    //   a.click();
    //   URL.revokeObjectURL(url);
    // });

    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');
    canvas.width = 800;
    canvas.height = 600;

    const scale = 400;
    const howtall = 1.2;

    // const maxYPoint = Math.max(...strokes.map((stroke) => Math.max(...stroke.map((point) => point.y))))
    // const minYPoint = Math.min(...strokes.map((stroke) => Math.min(...stroke.map((point) => point.y))))
    // const maxXPoint = Math.max(...strokes.map((stroke) => Math.max(...stroke.map((point) => point.x))))
    // const minXPoint = Math.min(...strokes.map((stroke) => Math.min(...stroke.map((point) => point.x))))

    // const normalize = (point: THREE.Vector3) => {
    //   const x = (point.x - minXPoint) / (maxXPoint - minXPoint);
    //   const y = (point.y - minYPoint) / (maxYPoint - minYPoint);
    //   return new THREE.Vector3(x, y, point.z);
    // };

    // const normalizedStrokes = strokes.map((stroke) => stroke.map(normalize));

    if (!context) return;

    strokes.forEach((stroke) => {
      if (stroke.length < 2) return;

      context.beginPath();

      let firstPoint = stroke[0];
      console.log(firstPoint);
      let mappedX = firstPoint.x * scale + canvas.width / 2;
      let mappedY = ((1 - firstPoint.y) * scale) + canvas.height / 2;

      context.moveTo(mappedX, mappedY);

      stroke.forEach((vec3) => {
        let x = vec3.x * scale + canvas.width / 2;
        let y = ((1 - vec3.y) * scale) + canvas.height / 2;
        context.lineTo(x, y);
      });

      context.strokeStyle = 'white'; // Set stroke color
      context.lineWidth = 2; // Set line width
      context.stroke(); // Draw the path
    });

    const url = canvas.toDataURL('image/png');
    const a = document.createElement('a');
    a.href = url;
    a.download = 'drawing.png';
    a.click();

  }

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