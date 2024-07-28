## General

**ATTENTION!** you will not need to use OpenMVS if you have a properly configured CUDA environment.

I used OpenMVS for multi-view stereo because COLMAP can only perform dense reconstruction on a CUDA device. Refer to the [Dense reconstruction without CUDA colmap/colmap#192](https://github.com/colmap/colmap/issues/192) for more details.

<br />

## Visualization

You can visualize the generated `.mvs` or `.ply` files using the OpenMVS Viewer module:
```
bin/Viewer file.mvs|file.ply
```

<br />

## Dependencies

1. [COLMAP](https://github.com/colmap/colmap)

On MacOS, you can install COLMAP via [homebrew](https://brew.sh/):
 ```
 brew install colmap
 ```
 For other platforms, please refer to the official [COLMAP installation guide](https://colmap.github.io/install.html).

2. [OpenMVS](https://github.com/cdcseacave/openMVS)

Refer the the [OpenMVS Build Guide](https://github.com/electech6/openMVS_comments/blob/master/BUILD.md) in order to build the software from source.
After building the tool, you should see the binaries in the `openMVS_build/bin` directory.


## Running the pipeline

Follow these steps to create a 3D model from human captured images

1. **Setup**
```
export RUN_NUM=<number>

mkdir "run${RUN_NUM}"

cp -r path/to/openMVS_build/bin "run${RUN_NUM}"

cd "run${RUN_NUM}"

export DATASET_PATH=`pwd`
```

 * Place your images folder inside `run${RUN_NUM}`. The `images` directory should only contain image files, (perhaps) with lexicographical naming.
```
mv path/to/images ./
```

 * (Optional) Down-scale the images. I use `sips` on macos:
```
cd images

sips -Z <image_size> ./*

cd ..
```

The `image_size` parameter ensures that no image has width or height greater than the specified integer, preserving the aspect ratio.

<br />

2. **Feature extraction**
```
colmap feature_extractor \
   --database_path $DATASET_PATH/database.db \
   --image_path $DATASET_PATH/images
```

<br />

3. **Feature matching**
```
colmap exhaustive_matcher \
   --database_path $DATASET_PATH/database.db
```

<br />

4. **Create a sparse 3D reconstruction**
```
mkdir $DATASET_PATH/sparse

colmap mapper \
    --database_path $DATASET_PATH/database.db \
    --image_path $DATASET_PATH/images \
    --output_path $DATASET_PATH/sparse
```

<br />

5. **Undistort images**
```
mkdir $DATASET_PATH/dense

colmap image_undistorter \
    --image_path $DATASET_PATH/images \
    --input_path $DATASET_PATH/sparse/0 \
    --output_path $DATASET_PATH/dense \
    --output_type COLMAP \
    --max_image_size <image_size>
```

The `max_image_size` argument ensures that no image has width or height greater than the specified `image_size` integer.
I keep it the same as `image_size` from step 1.

<br />

6. **Create a `scene.mvs` file using the undistorted camera poses and images, and the sparse point-cloud from the previous step:**
```
bin/InterfaceCOLMAP -i dense -o scene.mvs --image-folder dense/images
```

<br />

7. **(Optional) Densify the point cloud**
```
bin/DensifyPointCloud scene.mvs
```

> **Note:** The vertex colors are roughly estimated for visualization and do not contribute farther down the pipeline.

<br />

8. **Reconstruct a rough mesh**
```
bin/ReconstructMesh scene_dense.mvs -p scene_dense.ply
```

> **Note:** I have used the dense point-cloud for mesh reconstruction and refinement, the same can be acheived using the sparse point-cloud.

<br />

9. **(Optional) Refine the obtained mesh**
```
bin/RefineMesh scene_dense.mvs -m scene_dense_mesh.ply -o scene_dense_mesh_refine.mvs --max-face-area 16
```

> **Note:** Mesh refinement might produce better results when the `--scale 1` argument is provided. On macos, it seems to get stuck.

<br />

10. **Texture the mesh**
```
bin/TextureMesh scene_dense.mvs -m scene_dense_mesh_refine.ply -o scene_dense_mesh_refine_texture.mvs
```

<br />

## References

1. [COLMAP - Command-line Interface](https://colmap.github.io/cli.html#example)
2. [OpenMVS Usage](https://github.com/cdcseacave/openMVS/wiki/Usage#dense-point-cloud-reconstruction-using-available-depth-maps-optional)
