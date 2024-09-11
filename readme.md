# GLTFJSX

[![Version](https://img.shields.io/npm/v/gltfjsx?style=flat&colorA=000000&colorB=000000)](https://www.npmjs.com/package/gltfjsx) [![Discord Shield](https://img.shields.io/discord/740090768164651008?style=flat&colorA=000000&colorB=000000&label=discord&logo=discord&logoColor=ffffff)](https://discord.gg/ZZjjNvJ)

<https://user-images.githubusercontent.com/2223602/126318148-99da7ed6-a578-48dd-bdd2-21056dbad003.mp4>

A small command-line tool that turns GLTF assets into declarative and re-usable [react-three-fiber](https://github.com/pmndrs/react-three-fiber) JSX components.

## The GLTF workflow on the web is not ideal

- GLTF is thrown whole into the scene which prevents re-use, in threejs objects can only be mounted once
- Contents can only be found by traversal which is cumbersome and slow
- Changes to queried nodes are made by mutation, which alters the source data and prevents re-use
- Re-structuring content, making nodes conditional or adding/removing is cumbersome
- Model compression is complex and not easily achieved
- Models often have unnecessary nodes that cause extra work and matrix updates

### GLTFJSX fixes that

- 🧑‍💻 It creates a virtual graph of all objects and materials. Now you can easily alter contents and re-use.
- 🏎️ The graph gets pruned (empty groups, unnecessary transforms, ...) and will perform better.
- ⚡️ It will optionally compress your model with up to 70%-90% size reduction.

## Usage

```text
Usage
  $ npx gltfjsx [Model.glb] [options]

Options
  --output, -o        Output file name/path
  --types, -t         Add Typescript definitions
  --keepnames, -k     Keep original names
  --keepgroups, -K    Keep (empty) groups, disable pruning
  --bones, -b         Lay out bones declaratively (default: false)
  --meta, -m          Include metadata (as userData)
  --shadows, s        Let meshes cast and receive shadows
  --printwidth, w     Prettier printWidth (default: 120)
  --precision, -p     Number of fractional digits (default: 3)
  --draco, -d         Draco binary path
  --root, -r          Sets directory from which .gltf file is served
  --instance, -i      Instance re-occuring geometry
  --instanceall, -I   Instance every geometry (for cheaper re-use)
  --exportdefault, -E Use default export
  --transform, -T     Transform the asset for the web (draco, prune, resize)
    --resolution, -R  Resolution for texture resizing (default: 1024)
    --keepmeshes, -j  Do not join compatible meshes
    --keepmaterials, -M Do not palette join materials
    --format, -f      Texture format (default: "webp")
    --simplify, -S    Mesh simplification (default: false)
      --ratio         Simplifier ratio (default: 0)
      --error         Simplifier error threshold (default: 0.0001)
  --console, -c       Log JSX to console, won't produce a file
  --debug, -D         Debug output
```

### A typical use-case

First you run your model through gltfjsx. `npx` allows you to use npm packages without installing them.

```bash
npx gltfjsx model.gltf --transform
```

This will create a `Model.jsx` file that plots out all of the assets contents.

```jsx
/*
auto-generated by: https://github.com/pmdrs/gltfjsx
author: abcdef (https://sketchfab.com/abcdef)
license: CC-BY-4.0 (http://creativecommons.org/licenses/by/4.0/)
source: https://sketchfab.com/models/...
title: Model
*/

import { useGLTF, PerspectiveCamera } from '@react-three/drei'

export function Model(props) {
  const { nodes, materials } = useGLTF('/model-transformed.glb')
  return (
    <group {...props} dispose={null}>
      <PerspectiveCamera name="camera" fov={40} near={10} far={1000} position={[10, 0, 50]} />
      <pointLight intensity={10} position={[100, 50, 100]} rotation={[-Math.PI / 2, 0, 0]} />
      <group position={[10, -5, 0]}>
        <mesh geometry={nodes.robot.geometry} material={materials.metal} />
        <mesh geometry={nodes.rocket.geometry} material={materials.wood} />
      </group>
    </group>
  )
}

useGLTF.preload('/model.gltf')
```

Add your model to your `/public` folder as you would normally do. With the `--transform` flag it has created a compressed copy of it (in the above case `model-transformed.glb`). Without the flag just copy the original model.

```text
/public
  model-transformed.glb
```

The component can now be dropped into your scene.

```jsx
import { Canvas } from '@react-three/fiber'
import { Model } from './Model'

function App() {
  return (
    <Canvas>
      <Model />
```

You can re-use it, it will re-use geometries and materials out of the box:

```jsx
<Model position={[0, 0, 0]} />
<Model position={[10, 0, -10]} />
```

Or make the model dynamic. Change its colors for example:

```jsx
<mesh geometry={nodes.robot.geometry} material={materials.metal} material-color="green" />
```

Or exchange materials:

```jsx
<mesh geometry={nodes.robot.geometry}>
  <meshPhysicalMaterial color="hotpink" />
</mesh>
```

Make contents conditional:

```jsx
{
  condition && <mesh geometry={nodes.robot.geometry} material={materials.metal} />
}
```

Add events:

```jsx
<mesh geometry={nodes.robot.geometry} material={materials.metal} onClick={handleClick} />
```

## Features

### ⚡️ Draco and meshopt compression ootb

You don't need to do anything if your models are draco compressed, since `useGLTF` defaults to a [draco CDN](https://www.gstatic.com/draco/v1/decoders/). By adding the `--draco` flag you can refer to [local binaries](https://github.com/mrdoob/three.js/tree/dev/examples/js/libs/draco/gltf) which must reside in your /public folder.

### ⚡️ Preload your assets for faster response

The asset will be preloaded by default, this makes it quicker to load and reduces time-to-paint. Remove the preloader if you don't need it.

```jsx
useGLTF.preload('/model.gltf')
```

### ⚡️ Auto-transform (compression, resize)

With the `--transform` flag it creates a binary-packed, draco-compressed, texture-resized (1024x1024), webp compressed, deduped, instanced and pruned `*.glb` ready to be consumed on a web site. It uses [glTF-Transform](https://github.com/donmccurdy/glTF-Transform). This can reduce the size of an asset by 70%-90%.

It will not alter the original but create a copy and append `[modelname]-transformed.glb`.

### ⚡️ Type-safety

Add the `--types` flag and your GLTF will be typesafe.

```tsx
type GLTFResult = GLTF & {
  nodes: { robot: THREE.Mesh; rocket: THREE.Mesh }
  materials: { metal: THREE.MeshStandardMaterial; wood: THREE.MeshStandardMaterial }
}

export default function Model(props: JSX.IntrinsicElements['group']) {
  const { nodes, materials } = useGLTF<GLTFResult>('/model.gltf')
```

### ⚡️ Easier access to animations

If your GLTF contains animations it will add [drei's](https://github.com/pmndrs/drei) `useAnimations` hook, which extracts all clips and prepares them as actions:

```jsx
const { nodes, materials, animations } = useGLTF('/model.gltf')
const { actions } = useAnimations(animations, group)
```

If you want to play an animation you can do so at any time:

```jsx
<mesh onClick={(e) => actions.jump.play()} />
```

If you want to blend animations:

```jsx
const [name, setName] = useState("jump")
...
useEffect(() => {
  actions[name].reset().fadeIn(0.5).play()
  return () => actions[name].fadeOut(0.5)
}, [name])
```

### ⚡️ Auto-instancing

Use the `--instance` flag and it will look for similar geometry and create instances of them. Look into [drei/Merged](https://github.com/pmndrs/drei#instances) to understand how it works. It does not matter if you instanced the model previously in Blender, it creates instances for each mesh that has a specific geometry and/or material.

`--instanceall` will create instances of all the geometry. This allows you to re-use the model with the smallest amount of drawcalls.

Your export will look like something like this:

```jsx
const context = createContext()
export function Instances({ children, ...props }) {
  const { nodes } = useGLTF('/model-transformed.glb')
  const instances = useMemo(() => ({ Screw1: nodes['Screw1'], Screw2: nodes['Screw2'] }), [nodes])
  return (
    <Merged meshes={instances} {...props}>
      {(instances) => <context.Provider value={instances} children={children} />}
    </Merged>
  )
}

export function Model(props) {
  const instances = useContext(context)
  return (
    <group {...props} dispose={null}>
      <instances.Screw1 position={[-0.42, 0.04, -0.08]} rotation={[-Math.PI / 2, 0, 0]} />
      <instances.Screw2 position={[-0.42, 0.04, -0.08]} rotation={[-Math.PI / 2, 0, 0]} />
    </group>
  )
}
```

Note that similar to `--transform` it also has to transform the model. In order to use and re-use the model import both `Instances` and `Model`. Put all your models into the `Instances` component (you can nest).

The following will show the model three times, but you will only have 2 drawcalls tops.

```jsx
import { Instances, Model } from './Model'

<Instances>
  <Model position={[10,0,0]}>
  <Model position={[-10,0,0]}>
  <Model position={[-10,10,0]}>
</Instance>
```

## Using the parser stand-alone

```jsx
import { parse } from 'gltfjsx'
import { GLTFLoader, DRACOLoader } from 'three-stdlib'

const gltfLoader = new GLTFLoader()
const dracoloader = new DRACOLoader()
dracoloader.setDecoderPath('https://www.gstatic.com/draco/v1/decoders/')
gltfLoader.setDRACOLoader(dracoloader)

gltfLoader.load(url, (gltf) => {
  const jsx = parse(gltf, optionalConfig)
})
```

## Using the parser stand-alone for scenes (object3d's)

```jsx
const jsx = parse(scene, optionalConfig)
```

## Using GLTFStructureLoader stand-alone

The GLTFStructureLoader can come in handy while testing gltf assets. It allows you to extract the structure without the actual binaries and textures making it possible to run in a testing environment.

```jsx
import { GLTFStructureLoader } from 'gltfjsx'
import fs from 'fs/promises'

it('should have a scene with a blue mesh', async () => {
  const loader = new GLTFStructureLoader()
  const data = await fs.readFile('./model.glb')
  const { scene } = await new Promise((res) => loader.parse(data, '', res))
  expect(() => scene.children.length).toEqual(1)
  expect(() => scene.children[0].type).toEqual('mesh')
  expect(() => scene.children[0].material.color).toEqual('blue')
})
```

## Requirements

- Nodejs must be installed
- The GLTF file has to be present in your projects `/public` folder
- [three](https://github.com/mrdoob/three.js/) (>= 122.x)
- [@react-three/fiber](https://github.com/pmndrs/react-three-fiber) (>= 5.x)
- [@react-three/drei](https://github.com/pmndrs/drei) (>= 2.x)
