# Mesa RADV Under-The-Hood

## Important Information

- [RADV ACO Documentation](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/compiler/README.md)

## Creating a Graphics Pipeline using ACO

### Vulkan's Entry Point

I think the most logical place to start is `vkCreateGraphicsPipelines`, because (at least I presume) the creation of the
graphics pipeline is where the shaders are actually compiled and linked. If not, the second candidate would be
`vkCreateShaderModule`.

The entrypoints of all Vulkan functions are located in `radv_entrypoints.c`, which is actually generated from the build
process (`build/src/amd/vulkan/radv_entrypoints.c`). A C-`struct` (`vk_device_entrypoint_table`) houses all entrypoints
to Vulkan function, presumably for all drivers.

`vkCreateGraphicsPipelines` directly maps to `radv_CreateGraphicsPipelines` (in `src/amd/vulkan/radv_pipeline.c`). The
implementation appears to be trivial, only looping over each pipeline and invoking `radv_graphics_pipeline_create` on
each `VkGraphicsPipelineCreateInfo`. So far, so good.

### `radv_graphics_pipeline_create`

This function is the first RADV-specific function.

First, it recovers an instance of `radv_device*` and `radv_pipeline_cache*` from `VkDevice` and `VkPipelineCache`
respectively. I am guessing that `VkDevice` and `VkPipelineCache` are just opaque pointers to the actual implementation
structures.

```c
RADV_FROM_HANDLE(radv_device, device, _device);  
VK_FROM_HANDLE(vk_pipeline_cache, cache, _cache);
```

Then, a pipeline is allocated using the allocator provided from `vkCreateGraphicsPipeline` (`pAllocator`). `vk_zalloc2` is a Vulkan-common function that zero-allocates an object, similar to `malloc` + `memset(0)`.

```c
pipeline = vk_zalloc2(&device->vk.alloc, pAllocator, sizeof(*pipeline), 8, VK_SYSTEM_ALLOCATION_SCOPE_OBJECT);
```

Some fields of the pipeline object is then populated, notably the `vk_object_base` (which is present for all Vulkan objects in Mesa), and `type` (which indicates the type of the pipeline object).
```c
struct radv_pipeline {  
   struct vk_object_base base;
   enum radv_pipeline_type type;

   // ...
};

radv_pipeline_init(device, &pipeline->base, RADV_PIPELINE_GRAPHICS);
```

The other fields of the pipeline object is populated from the `VkGraphicsPipelineCreateInfo`. The `vk_graphics_pipeline_create_flags` provides compatibility between core Vulkan and `VK_KHR_maintenance5`. `is_internal`  determines whether the passed-in shader is internal to RADV by checking whether the internal `meta_state` cache is passed in instead of a user-provided cache.

```c
pipeline->base.create_flags = vk_graphics_pipeline_create_flags(pCreateInfo);  
pipeline->base.is_internal = _cache == device->meta_state.cache;
```

Finally, control is passed to `radv_graphics_pipeline_init`, and if a failure occurred while initializing the pipeline, the pipeline is destroyed and its status is returned.

```c
result = radv_graphics_pipeline_init(pipeline, device, cache, pCreateInfo, extra);  
if (result != VK_SUCCESS) {  
   radv_pipeline_destroy(device, &pipeline->base, pAllocator);  
   return result;  
}
```

#### Sidenote: Meta Shaders

Meta-Shaders appear to be shaders that implement commonly used functionality that are not provided by the hardware. The current list of meta-shaders (as of Mesa 24.0.6) include:

- `astc_decode`
- `blit`
- `blit2d`
- `buffer`
- `bufimage`
- `clear`
- `copy`
- `copy_vrs_htile`
- `dcc_retile`
- `decompress`
- `etc_decode`
- `fast_clear`
- `fmask_copy`
- `fmask_expand`
- `resolve`
- `resolve_cs`
- `resolve_fs` 

These shaders are generally implemented in NIR and is compiled by LLVM or ACO when these shaders are used, which is why they also contain calls to `radv_graphics_pipeline_create`.

### `radv_graphics_pipeline_init`

This is pretty much the bulk of logic when a graphics pipeline is created. Let's assume that `VK_EXT_graphics_pipeline_library` is not used.

First, we need to create and initialize a pipeline layout object to determine how the different stages would need to be pieced together. This layout object appears to only be used within this scope to merge layout information from both the graphics pipeline library (GPL) and the current invocation of `vkCreateGraphicsPipeline`.

```c
struct radv_pipeline_layout pipeline_layout;
radv_pipeline_layout_init(device, &pipeline_layout, false);
```

The pipeline layout information would be imported first from the GPL, then from our current `pCreateInfo` struct.

```c
/* If we have libraries, import them first. */  
if (libs_info) {    
      radv_graphics_pipeline_import_lib(...);
   }  
}  
  
/* Import graphics pipeline info that was not included in the libraries. */  
result = radv_pipeline_import_graphics_info(device, pipeline, &state, &pipeline_layout, pCreateInfo, needed_lib_flags);
if (result != VK_SUCCESS) {
   radv_pipeline_layout_finish(device, &pipeline_layout);
   return result;
}
```

Diving into `radv_pipeline_import_graphics_info`... First step seem pretty self-explanatory, setting internal dynamic states (from `VK_KHR_dynamic_rendering`) as they are.

```c
/* Mark all states declared dynamic at pipeline creation. */  
if (pCreateInfo->pDynamicState) {  
   uint32_t count = pCreateInfo->pDynamicState->dynamicStateCount;  
   for (uint32_t s = 0; s < count; s++) {  
      pipeline->dynamic_states |= radv_dynamic_state_mask(pCreateInfo->pDynamicState->pDynamicStates[s]);  
   }  
}
```

Next, set the active stages of the pipeline based on `pCreateInfo`, ignoring any that have already been set by the GPL.

```c
/* Mark all active stages at pipeline creation. */  
for (uint32_t i = 0; i < pCreateInfo->stageCount; i++) {  
   const VkPipelineShaderStageCreateInfo *sinfo = &pCreateInfo->pStages[i];  
  
   /* Ignore shader stages that don't need to be imported. */  
   if (!(shader_stage_to_pipeline_library_flags(sinfo->stage) & lib_flags))  
      continue;  
  
   pipeline->active_stages |= sinfo->stage;  
}
```



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
