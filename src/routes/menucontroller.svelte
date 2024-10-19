<script lang="ts">
  import * as THREE from 'three'
  import { T, useTask, useThrelte } from '@threlte/core'
  import { Text, HTML } from '@threlte/extras'
  import { Controller, type XRControllerEvent, pointerControls, useController } from '@threlte/xr'
  import { text } from './stores'

  const { scene } = useThrelte();
  const controller = useController($$restProps.left ? 'left' : 'right')

  pointerControls('right')

  const menuButtons = [
    'pen', 'select'
  ]

</script>

<Controller
  left={$$props.left}
  right={$$props.right}
>
  <T.Group slot="target-ray" position={[0.1, 0, 0]}>

    {#each menuButtons as button, idx}
      <T.Mesh position={[0.06 * idx, 0, 0]} on:click={() => {
        text.set(button)
      }}>
        <T.BoxGeometry args={[0.05, 0.05, 0.05]} />
        <T.MeshStandardMaterial color={0xbbbbbb} />
      </T.Mesh>
    {/each}

  </T.Group>
</Controller>