<script lang="ts">
  import * as THREE from 'three'
  import { T, useTask, useThrelte } from '@threlte/core'
  import { Text, HTML } from '@threlte/extras'
  import { Controller, type XRControllerEvent, useController } from '@threlte/xr'

  const { scene } = useThrelte();
  const controller = useController($$restProps.left ? 'left' : 'right')

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
      <T.Mesh position={[0.1 * idx, 0, 0]}>
        <T.BoxGeometry args={[0.05, 0.05, 0.05]} />
        <T.MeshStandardMaterial color={0xffffff} />
      </T.Mesh>
    {/each}

  </T.Group>
</Controller>