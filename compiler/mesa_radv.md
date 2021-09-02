# Mesa RADV Under-The-Hood

This is based on Mesa 21.2.1.

## Vulkan's Entry Point

I think the most logical place to start is `vkCreateGraphicsPipelines`, because (at least I presume) the creation of the
graphics pipeline is where the shaders are actually compiled and linked. If not, the second candidate would be
`vkCreateShaderModule`.

The entrypoints of all Vulkan functions are located in `radv_entrypoints.c`, which is actually generated from the build
process (`build/src/amd/vulkan/radv_entrypoints.c`). A C-`struct` (`vk_device_entrypoint_table`) houses all entrypoints
to Vulkan function, presumably for all drivers.

`vkCreateGraphicsPipelines` directly maps to `radv_CreateGraphicsPipelines`
([here](https://gitlab.freedesktop.org/mesa/mesa/-/blob/mesa-21.2.1/src/amd/vulkan/radv_pipeline.c#L5492)). The
implementation appears to be trivial, only looping over each pipeline and invoking `radv_graphics_pipeline_create` on
each `VkGraphicsPipelineCreateInfo`. So far, so good.

## `radv_graphics_pipeline_create`

This function is the first RADV-specific function. It appears that this is not the only caller of the function, rather
other meta-operations would also need to build a pipeline.

First, it recovers an instance of `radv_device*` and `radv_pipeline_cache*` from `VkDevice` and `VkPipelineCache`
respectively. I am guessing that `VkDevice` and `VkPipelineCache` are just opaque pointers to the actual implementation
structures.
