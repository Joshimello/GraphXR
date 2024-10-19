<script lang="ts">
	import * as THREE from 'three';
	import { T, useTask, useThrelte } from '@threlte/core';
	import { XR } from '@threlte/xr';
  import { Text } from '@threlte/extras';
  import Controller from './controller.svelte';
  import DrawController from './drawcontroller.svelte';
	import Menucontroller from './menucontroller.svelte';
</script>

<XR>
  <Menucontroller left />
  <DrawController right />
</XR>

<T.HemisphereLight args={[0xffffff, 0xbbbbff, 1]} position={[0.5, 1, 0.25]} />
<T.AmbientLight intensity={0.5} />