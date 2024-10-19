<script lang="ts">
  import * as THREE from 'three'
  import { T, useTask } from '@threlte/core'
  import { Text, HTML } from '@threlte/extras'
  import { Controller, type XRControllerEvent, useController } from '@threlte/xr'

  let text = ''
  let isDrawing = false
  let points: THREE.Vector3[] = []
  let meshes: THREE.Mesh[] = []
  let line: THREE.Line | null = null

  const lineMaterial = new THREE.LineBasicMaterial({ color: 0xff0000 })
  let lineGeometry = new THREE.BufferGeometry()

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
    const tubeGeometry = new THREE.TubeGeometry(curve, 32, 0.005, 32, false)
    const tubeMaterial = new THREE.MeshPhongMaterial({ color: 0x00ff00 })
    const tube = new THREE.Mesh(tubeGeometry, tubeMaterial)

    meshes = [...meshes, tube]
    line = null
    points = []
  }

  const handleSqueezeStart = (event: XRControllerEvent) => {
    meshes = []
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
    <Text text={"hi"} />
  </T.Group>

</Controller>

{#if line}
  <T is={line} />
{/if}

{#each meshes as mesh, index (index)}
  <T is={mesh} />
{/each}