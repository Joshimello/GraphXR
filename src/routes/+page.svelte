<script lang="ts">
  import { Canvas } from '@threlte/core'
  import { Debug, World } from '@threlte/rapier'
  import { ARButton, VRButton } from '@threlte/xr'
  import Scene from './scene.svelte'
</script>

<div class="w-full h-screen">
  <Canvas rendererParameters={{
    preserveDrawingBuffer: true
  }}>
    <World>
      <Debug />
      <Scene />
    </World>
  </Canvas>
  <VRButton />
</div>