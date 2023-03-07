# Mesa RADV Under-The-Hood

This is based on Mesa 23.0.0.

## Important Information

- [RADV ACO Documentation](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/compiler/README.md)

## Vulkan's Entry Point

I think the most logical place to start is `vkCreateGraphicsPipelines`, because (at least I presume) the creation of the
graphics pipeline is where the shaders are actually compiled and linked. If not, the second candidate would be
`vkCreateShaderModule`.

The entrypoints of all Vulkan functions are located in `radv_entrypoints.c`, which is actually generated from the build
process (`build/src/amd/vulkan/radv_entrypoints.c`). A C-`struct` (`vk_device_entrypoint_table`) houses all entrypoints
to Vulkan function, presumably for all drivers.

`vkCreateGraphicsPipelines` directly maps to `radv_CreateGraphicsPipelines` (in `src/amd/vulkan/radv_pipeline.c`). The
implementation appears to be trivial, only looping over each pipeline and invoking `radv_graphics_pipeline_create` on
each `VkGraphicsPipelineCreateInfo`. So far, so good.

## `radv_graphics_pipeline_create`

This function is the first RADV-specific function. It appears that this is not the only caller of the function, rather
other meta-operations would also need to build a pipeline.

First, it recovers an instance of `radv_device*` and `radv_pipeline_cache*` from `VkDevice` and `VkPipelineCache`
respectively. I am guessing that `VkDevice` and `VkPipelineCache` are just opaque pointers to the actual implementation
structures.

## ACO's Entry Point

Coming back to this, I think a better way to do this is is to find where shader programs get passed into ACO, and 
follow its callers to find which Vulkan functions (or its RADV-specific implementations) invoke the compilation 
process.

With a bit of digging, it appears that the "entry-point" to the ACO compiler is `aco_compile_shader` in 
`src/amd/compiler/aco_interface.cpp`. Following the callers, we have:

```
aco_compile_shader [src/amd/compiler/aco_interface.cpp]
    shader_compile [src/amd/vulkan/radv_shader.c]
```

### `shader_compile`

After this, we have 3 callers that will invoke the `shader_compile` function:

- `radv_shader_nir_to_asm`
- `radv_create_gs_copy_shader`
- `radv_create_trap_handler_shader`

`radv_create_gs_copy_shader` references a type of shader called "GS Copy", which when referring to the RADV ACO
documentation, is a shader which executes in the *hardware Vertex Shader* stage in GFX6-9 GPUs, in order to copy the
result of the *software Geometry Shader* from the VRAM to the *hardware Pixel Shader* input.

I am not sure what the Trap Handler Shader is or what it does, but following its callers we have:

```
radv_trap_handler_init [src/amd/vulkan/radv_debug.c]
radv_CreateDevice [src/amd/vulkan/radv_device.c]
```

The code path which invokes `radv_trap_handler_init` is guarded by a `getenv("RADV_TRAP_HANDLER")` followed by an
assertion that the GPU is of the GFX8 family. This seems to be just for debugging purposes and currently only
implemented for the GFX8 family, so we are going to skip this.

We will continue to follow the callers of `radv_shader_nir_to_asm` first, then depending on the complexity, we will do
a deep dive on one of the two interesting code paths first.

Continuing from `radv_shader_nir_to_asm`, we have:

```
radv_pipeline_nir_to_asm [src/amd/vulkan/radv_pipeline.c]
radv_create_shaders [src/amd/vulkan/radv_pipeline.c]
```

And then we have another 5 callers to this function.

### `radv_create_shaders`

The 5 callers to this function are:

- `radv_compute_pipeline_create`

  This function appears to create a Vulkan compute pipeline, and is called by `radv_CreateComputePipeline` (which is a
  function delegate for `vkCreateComputePipeline`).

- `radv_graphics_lib_pipeline_init`

  This function appears to initialize a Vulkan graphics library pipeline. The Vulkan graphics library pipeline is
  introduced in the `VK_EXT_graphics_pipeline_library` extension, which aims to speed up shader linking. The function is
  called by `radv_graphics_lib_pipeline_create`, and is in turn called by `radv_CreateGraphicsPipeline` (which is a
  function delegate for `vkCreateGraphicsPipeline`). This code path is only hit if the
  `VK_PIPELINE_CREATE_LIBRARY_BIT_KHR` flag is set in the `VkGraphicsPipelineCreateInfo` of the to-be-created pipeline.

- `radv_graphics_pipeline_init`

  This function appears to create a Vulkan graphics pipeline, and is called by `radv_graphics_pipeline_create`. This
  internal function is called by many locations within the code, one of which is `radv_CreateGraphicsPipeline`as
  expected.

- 2 calls from `radv_rt_pipeline_create`

  This function appears to create a Vulkan ray-tracing pipeline. The Vulkan ray-tracing pipelien is introduced in the
  `VK_KHR_ray_tracing_pipeline` extension, which provides ray-tracing pipeline support. The function is called by
  `radv_CreateRayTracingPipelineKHR` (which is a function delegate for `vkCreateRayTracingPipelineKHR`).

  The reason why there are two calls to `radv_create_shaders` is mentioned in the comment:

  ```c
     /* First check if we can get things from the cache before we take the expensive step of
      * generating the nir. */
  ```

  The first call to `radv_create_shaders` passes an additional flag of
  `VK_PIPELINE_CREATE_FAIL_ON_PIPELINE_COMPILE_REQUIRED_BIT` to get and return a cached version of the pipeline if it
  exists. If the cached version does not exist, the second call is used to perform the actual compilation of the shader.

## Appendix

### GFX Family Table

| GFX Family | Generation       | Architecture  |
| ---------- | ---------------- | ------------- |
| GFX6       | Southern Islands | GCN 1.0       |
| GFX7       | Sea Islands      | GCN 2.0       |
| GFX8       | Volcanic Islands | GCN 3.0       |
| GFX8       | Arctic Islands   | GCN 4.0       |
| GFX8       | Polaris          | GCN 4.0       |
| GFX9       | Vega             | GCN 5.0       |
| GFX9       | Vega II          | GCN 5.1       |
| GFX10      | Navi             | RDNA 1.0      |
| GFX10.3    | Navi II          | RDNA 2.0      |
| GFX11      | Navi III         | RDNA 3.0      |
