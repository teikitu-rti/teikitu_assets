# Vulkan Dynamic Rendering - Quick Reference

## Quick Start Guide

This is a **quick reference** for the Vulkan dynamic rendering migration. For complete details, see [DYNAMIC_RENDERING_MIGRATION_PLAN.md](./DYNAMIC_RENDERING_MIGRATION_PLAN.md).

---

## At a Glance

### What's Changing

| Old (Render Pass) | New (Dynamic Rendering) |
|-------------------|-------------------------|
| `vkCreateRenderPass` | *(eliminated)* |
| `vkCreateFramebuffer` | *(eliminated)* |
| `vkCmdBeginRenderPass` | `vkCmdBeginRendering` |
| `vkCmdEndRenderPass` | `vkCmdEndRendering` |
| `VkGraphicsPipelineCreateInfo::renderPass` | `VkPipelineRenderingCreateInfo` (chained) |

### Key Benefits

✅ **~30-40% less boilerplate code**  
✅ **No render pass/framebuffer objects**  
✅ **Flexible attachment changes**  
✅ **Vulkan 1.3 best practices**  
✅ **Better driver optimization**  

---

## Minimal Code Example

### Before (Old Way)
```c
// 1. Create render pass
VkRenderPassCreateInfo rp_ci = { /* ... */ };
vkCreateRenderPass(device, &rp_ci, NULL, &render_pass);

// 2. Create framebuffer
VkFramebufferCreateInfo fb_ci = { /* ... */ };
vkCreateFramebuffer(device, &fb_ci, NULL, &framebuffer);

// 3. Create pipeline
VkGraphicsPipelineCreateInfo pipeline_ci = {
    .renderPass = render_pass,
    .subpass = 0,
    /* ... */
};
vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipeline_ci, NULL, &pipeline);

// 4. Record commands
VkRenderPassBeginInfo begin_info = { /* ... */ };
vkCmdBeginRenderPass(cmd, &begin_info, VK_SUBPASS_CONTENTS_INLINE);
vkCmdDraw(cmd, 3, 1, 0, 0);
vkCmdEndRenderPass(cmd);
```

### After (New Way)
```c
// 1. Create pipeline (no render pass!)
VkPipelineRenderingCreateInfo rendering_ci = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_RENDERING_CREATE_INFO,
    .colorAttachmentCount = 1,
    .pColorAttachmentFormats = &(VkFormat){VK_FORMAT_B8G8R8A8_SRGB}
};

VkGraphicsPipelineCreateInfo pipeline_ci = {
    .pNext = &rendering_ci,  // Chain rendering info
    .renderPass = VK_NULL_HANDLE,
    /* ... */
};
vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipeline_ci, NULL, &pipeline);

// 2. Record commands
VkRenderingAttachmentInfo color_attach = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
    .imageView = image_view,
    .imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
    .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
    .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
    .clearValue = {.color = {{0.0f, 0.0f, 0.0f, 1.0f}}}
};

VkRenderingInfo rendering_info = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_INFO,
    .renderArea = {{0, 0}, {width, height}},
    .layerCount = 1,
    .colorAttachmentCount = 1,
    .pColorAttachments = &color_attach
};

vkCmdBeginRendering(cmd, &rendering_info);
vkCmdDraw(cmd, 3, 1, 0, 0);
vkCmdEndRendering(cmd);
```

---

## Teikitu Structure Changes

### New Structures to Add

```c
typedef struct STg2_KN_GPU_Dynamic_Rendering_State {
    TgUINT_E32                nuiColorAttachments;
    VkFormat                  asColorFormats[KTgKN_GPU_MAX_COLOR_ATTACHMENTS];
    VkFormat                  enDepthFormat;
    VkFormat                  enStencilFormat;
    VkSampleCountFlagBits     enSamples;
} STg2_KN_GPU_Dynamic_Rendering_State;

typedef struct STg2_KN_GPU_Rendering_Attachment_Info {
    VkImageView               vk_image_view;
    VkImageLayout             enImageLayout;
    VkResolveModeFlagBits     enResolveMode;
    VkImageView               vk_resolve_image_view;
    VkImageLayout             enResolveImageLayout;
    VkAttachmentLoadOp        enLoadOp;
    VkAttachmentStoreOp       enStoreOp;
    VkClearValue              sClearValue;
} STg2_KN_GPU_Rendering_Attachment_Info;
```

### Old Structures to Remove

```c
// REMOVE:
// typedef struct STg2_KN_GPU_Render_Pass { ... };
// typedef struct STg2_KN_GPU_Framebuffer { ... };
```

### Modified Structures

```c
typedef struct STg2_KN_GPU_Pipeline {
    VkPipeline                          vk_pipeline;
    VkPipelineLayout                    vk_pipeline_layout;
    STg2_KN_GPU_Dynamic_Rendering_State sDynamic_Rendering_State; // ADDED
    // TgKN_GPU_RENDER_PASS_ID          tiRender_Pass; // REMOVED
} STg2_KN_GPU_Pipeline;
```

---

## Function Signature Changes

### Old Functions (Remove)

```c
TgRESULT tgKN_GPU_EXT__Render_Pass__Create(...);
TgRESULT tgKN_GPU_EXT__Render_Pass__Destroy(...);
TgRESULT tgKN_GPU_EXT__Framebuffer__Create(...);
TgRESULT tgKN_GPU_EXT__Framebuffer__Destroy(...);
TgRESULT tgKN_GPU_EXT__Execute__Begin_Render_Pass(...);
TgRESULT tgKN_GPU_EXT__Execute__End_Render_Pass(...);
```

### New Functions (Add)

```c
TgRESULT tgKN_GPU_EXT__Execute__Begin_Rendering(
    VkCommandBuffer vk_cmd_buffer,
    TgUINT_E32 uiNum_Color_Attachments,
    STg2_KN_GPU_Rendering_Attachment_Info *IN asColor_Attachments,
    STg2_KN_GPU_Rendering_Attachment_Info *IN psDepth_Attachment,
    STg2_KN_GPU_Rendering_Attachment_Info *IN psStencil_Attachment,
    VkRect2D sRender_Area,
    TgUINT_E32 uiLayer_Count
);

TgRESULT tgKN_GPU_EXT__Execute__End_Rendering(
    VkCommandBuffer vk_cmd_buffer
);
```

### Modified Functions

```c
// OLD:
TgRESULT tgKN_GPU_EXT__Pipeline__Create(
    STg2_KN_GPU_Pipeline *OUT psPipeline,
    TgKN_GPU_RENDER_PASS_ID tiRender_Pass,  // REMOVE THIS
    /* ... */
);

// NEW:
TgRESULT tgKN_GPU_EXT__Pipeline__Create(
    STg2_KN_GPU_Pipeline *OUT psPipeline,
    STg2_KN_GPU_Dynamic_Rendering_State *IN psDynamic_Rendering_State,  // ADD THIS
    /* ... */
);
```

---

## Migration Checklist

Use this as a quick progress tracker:

### Phase 1: Preparation
- [ ] Update CMake to require Vulkan 1.3
- [ ] Add dynamic rendering feature detection
- [ ] Enable feature in device creation

### Phase 2: Structure Updates
- [ ] Add new dynamic rendering structures to Type.h
- [ ] Remove old render pass/framebuffer structures
- [ ] Update pipeline structure

### Phase 3: Pipeline Creation
- [ ] Refactor pipeline creation function
- [ ] Add VkPipelineRenderingCreateInfo support
- [ ] Update all pipeline creation call sites

### Phase 4: Render Pass Removal
- [ ] Remove render pass creation code
- [ ] Remove framebuffer creation code
- [ ] Clean up unused functions

### Phase 5: Command Recording
- [ ] Implement Begin_Rendering function
- [ ] Implement End_Rendering function
- [ ] Update all command recording call sites

### Phase 6: Debug Rendering
- [ ] Update debug overlay rendering
- [ ] Update UI rendering

### Phase 7: Validation
- [ ] Run functional tests
- [ ] Run visual regression tests
- [ ] Run performance benchmarks
- [ ] Validate on all platforms

---

## Common Patterns

### Single Color Attachment
```c
VkRenderingAttachmentInfo color = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
    .imageView = view,
    .imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
    .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
    .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
    .clearValue = {.color = {{0, 0, 0, 1}}}
};

VkRenderingInfo info = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_INFO,
    .renderArea = {{0, 0}, {w, h}},
    .layerCount = 1,
    .colorAttachmentCount = 1,
    .pColorAttachments = &color
};
```

### Color + Depth
```c
VkRenderingAttachmentInfo depth = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
    .imageView = depth_view,
    .imageLayout = VK_IMAGE_LAYOUT_DEPTH_ATTACHMENT_OPTIMAL,
    .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
    .storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE,
    .clearValue = {.depthStencil = {1.0f, 0}}
};

VkRenderingInfo info = {
    /* ... color setup ... */
    .pDepthAttachment = &depth
};
```

### Multiple Render Targets (MRT)
```c
VkRenderingAttachmentInfo colors[2] = {
    { /* albedo setup */ },
    { /* normal setup */ }
};

VkRenderingInfo info = {
    .renderArea = {{0, 0}, {w, h}},
    .layerCount = 1,
    .colorAttachmentCount = 2,
    .pColorAttachments = colors
};
```

---

## Required Headers

```c
#include <vulkan/vulkan.h>

// Ensure Vulkan 1.3 or extension is available
#ifndef VK_KHR_dynamic_rendering
#error "VK_KHR_dynamic_rendering required"
#endif
```

---

## Validation

Enable validation layers and check for errors:

```bash
# Set environment variables
export VK_LAYER_PATH=/path/to/vulkan/layers
export VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation

# Run with validation
./your_app --validation
```

Expected: **Zero validation errors or warnings**

---

## Performance Notes

### Expected Impact
- **CPU overhead**: Similar or slightly better
- **Memory usage**: Reduced (no render pass/framebuffer objects)
- **Pipeline creation**: Slightly faster

### Optimization Tips
1. **Cache attachment info**: Reuse `VkRenderingAttachmentInfo` structures
2. **Batch format changes**: Group draws with same formats
3. **Use DONT_CARE**: For transient attachments (depth in forward rendering)

---

## Troubleshooting

### Validation Error: "dynamicRendering feature not enabled"
**Fix**: Enable feature during device creation
```c
VkPhysicalDeviceDynamicRenderingFeatures dr = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DYNAMIC_RENDERING_FEATURES,
    .dynamicRendering = VK_TRUE
};
// Chain to VkDeviceCreateInfo::pNext
```

### Validation Error: "Format mismatch between pipeline and attachment"
**Fix**: Ensure pipeline formats match attachment formats exactly
```c
// Pipeline creation:
.pColorAttachmentFormats = &(VkFormat){VK_FORMAT_B8G8R8A8_SRGB}

// Rendering:
.imageView = view_with_same_format  // Must be VK_FORMAT_B8G8R8A8_SRGB
```

### Black Screen / No Rendering
**Check**:
1. Image layout is `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`
2. Load op is `CLEAR` or `LOAD` (not `DONT_CARE`)
3. Store op is `STORE` (not `DONT_CARE`)
4. Pipeline format matches attachment format

---

## Resources

- 📄 **Full Migration Plan**: [DYNAMIC_RENDERING_MIGRATION_PLAN.md](./DYNAMIC_RENDERING_MIGRATION_PLAN.md)
- 📘 **Vulkan Spec**: [VK_KHR_dynamic_rendering](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_dynamic_rendering.html)
- 🎓 **Khronos Tutorial**: [Dynamic Rendering Sample](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/extensions/dynamic_rendering)

---

**Quick Reference Version**: 1.0  
**Last Updated**: 2025-10-24
