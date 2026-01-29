<script lang="ts">
  import {T, extend, useThrelte, useTask} from '@threlte/core';
  import {Environment, OrbitControls, useGltf} from '@threlte/extras';
  import * as THREE from 'three';

  // Integration between Threlte and Three.js.
  extend(THREE);

  const {invalidate} = useThrelte();

  /** 
   * facial morph targets.
   * it shape names to their values (0.0 to 1.0).
   */
  export let faceSettings: Record<string, number> = {};

  /** The playback speed for animation */
  export let playbackSpeed: number = 0;

  /** load the GLTF file */
  const gltf = useGltf('/ria.glb');

  /** Internal animation */
  let animationMixer: THREE.AnimationMixer | undefined;

  /** array of meshe contain facial expressions. */
  let faceMeshes: THREE.Mesh[] = [];

  /**
   * if GLTF file is loaded.
   * it can handles centering, scaling, and caching.
   */
  $: if ($gltf) {
    const robotRoot = $gltf.scene;

    // Calculate the box for centering and scaling.
    const boundingBox = new THREE.Box3().setFromObject(robotRoot);
    const modelSize = boundingBox.getSize(new THREE.Vector3()).length();
    const modelCenter = boundingBox.getCenter(new THREE.Vector3());

    robotRoot.position.set(-modelCenter.x, -modelCenter.y, -modelCenter.z);
    
    if (modelSize > 0) {
      robotRoot.scale.setScalar(2 / modelSize);
    }

    // animations.
    animationMixer = new THREE.AnimationMixer(robotRoot);
    $gltf.animations.forEach((clip: THREE.AnimationClip) => {
      animationMixer?.clipAction(clip).play();
    });

    // slider performance.
    faceMeshes = [];
    robotRoot.traverse((node: THREE.Object3D) => {
      const mesh = node as THREE.Mesh;
      if (mesh.isMesh && mesh.morphTargetInfluences) {
        faceMeshes.push(mesh);
      }
    });
  }

  /**
   * Animation.
   * Executes approximately 60 times per second.
   */
  useTask((delta: number) => {
    if (animationMixer && playbackSpeed > 0) {
      animationMixer.update(delta * playbackSpeed);
      invalidate();
    }
  });

  /**
   * Reactive update for facial expressions.
   * morph target works with the current faceSettings.
   */
  $: if (faceMeshes.length > 0 && faceSettings) {
    for (const mesh of faceMeshes) {
      for (const [name, value] of Object.entries(faceSettings)) {
        const dictionary = mesh.morphTargetDictionary;
        if (dictionary) {
          const muscleIndex = dictionary[name];
          if (muscleIndex !== undefined) {
            mesh.morphTargetInfluences![muscleIndex] = value;
          }
        }
      }
    }
    invalidate();
  }
</script>

<T.PerspectiveCamera
  makeDefault
  position={[0, 1, 3]}
  fov={35}>
  <OrbitControls
    enableDamping
    target={[0, 1, 0]}/>
</T.PerspectiveCamera>

<Environment
  path="https://raw.githubusercontent.com/pmndrs/drei-assets/master/"
  files="reid_free_area_1k.hdr"
/>

<T.AmbientLight intensity={1} />
<T.DirectionalLight
  position={[5, 10, 5]}
  intensity={2}/>

{#if $gltf}
  <T is={$gltf.scene} />
{/if}