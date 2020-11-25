# superpixel-mesh-wasm

CMake based scripts to build [superpixel-mesh](https://github.com/manlito/superpixel-mesh) library into a WebAssembly module. A lot of inspiration taken from emscripten examples and [ammo.js](https://github.com/kripken/ammo.js/)

## Prerequisites

- [emscripten](https://emscripten.org/) (make sure to activate the environment)
- Git

## How to use

```
git clone https://github.com/manlito/superpixel-mesh-wasm
mkdir superpixel-mesh-wasm
cd superpixel-mesh-wasm
cmake ../superpixel-mesh-wasm
make -j5
```
After that, you should have `SuperpixelMeshModule.js` and `SuperpixelMeshModule.wasm`.

## Motivation

Generate a WASM version of [superpixel-mesh](https://github.com/manlito/superpixel-mesh) that is completely independent of the library. [superpixel-mesh](https://github.com/manlito/superpixel-mesh) does not know anything about this repo. However, to be compatible with current features of WebIDL (as supported today by emscripten), SuperpixelMesh had some tweaks done that may result in not so modern patterns in a few places.

## How does this script work

Everything happends inside [CMakeLists.txt](CMakeLists.txt). In order:

1. Download of [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page), [superpixel-mesh](https://github.com/ceres-solver/ceres-solver) and [superpixel-mesh](https://github.com/manlito/superpixel-mesh).
2. Create targets for `ceres-solver` and `superpixel-mesh`.
3. Create and build glue code using [superpixel_mesh.idl](superpixel_mesh.idl) and emscripten [WebIDL binder](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/WebIDL-Binder.html) python script.
4. Build `ceres-solver` and `superpixel-mesh`.
5. Link glue code and targets just built. This will finally generate files `SuperpixelMeshModule.js` and `SuperpixelMeshModule.wasm`.

Please note the main prerequesite is you having [emscripten](https://emscripten.org/) compiler already setup.