# Vulkan Dynamic Rendering Migration Plan

## Executive Summary

This document provides a comprehensive migration strategy for refactoring the Teikitu Vulkan GPU backend from traditional render pass objects (`VkRenderPass`) to Vulkan's dynamic rendering feature (VK_KHR_dynamic_rendering / Vulkan 1.3 core).

**Migration Type**: Architectural Refactor  
**Target Vulkan Version**: 1.3+  
**Extension Fallback**: VK_KHR_dynamic_rendering (Vulkan 1.1+, 1.2+)  
**Estimated Impact**: Medium - Core rendering pipeline changes  
**Backwards Compatibility**: Requires runtime feature detection

---

## Table of Contents

1. [Overview](#overview)
2. [Current Architecture Analysis](#current-architecture-analysis)
3. [Target Architecture](#target-architecture)
4. [Migration Phases](#migration-phases)
5. [Technical Implementation Details](#technical-implementation-details)
6. [File-by-File Changes](#file-by-file-changes)
7. [Validation and Testing](#validation-and-testing)
8. [Rollback Strategy](#rollback-strategy)
9. [Performance Considerations](#performance-considerations)
10. [References](#references)

---

## Overview

### Motivation

Traditional Vulkan render passes require pre-declaring:
- Attachment formats and layouts
- Load/store operations
- Subpass dependencies
- Render pass compatibility rules

**Dynamic rendering eliminates**:
- Render pass object creation and management
- Framebuffer objects (when using imageless framebuffers)
- Complex render pass compatibility tracking
- Inflexible attachment specification

**Dynamic rendering provides**:
- Inline attachment specification during command recording
- Simplified pipeline creation
- Better flexibility for runtime attachment changes
- Alignment with modern Vulkan best practices
- Reduced API overhead

### Benefits

1. **Code Simplification**: ~30-40% reduction in render pass management code
2. **Flexibility**: Change attachments per draw without recreating render passes
3. **Performance**: Reduced validation overhead, better driver optimization opportunities
4. **Maintainability**: Clearer code flow, fewer state dependencies
5. **Future-Proofing**: Alignment with Vulkan 1.3+ best practices

### Compatibility Requirements

**Minimum Requirements**:
- Vulkan 1.3 (dynamic rendering is core)
- OR Vulkan 1.1/1.2 + VK_KHR_dynamic_rendering extension

**Feature Detection Required**:
```c
VkPhysicalDeviceDynamicRenderingFeatures dynamicRenderingFeatures = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DYNAMIC_RENDERING_FEATURES,
    .dynamicRendering = VK_TRUE
};

VkPhysicalDeviceFeatures2 features2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2,
    .pNext = &dynamicRenderingFeatures
};
```

---

## Current Architecture Analysis

### Render Pass Object Model

**Current Flow**:
```
1. Create VkRenderPass with attachment descriptions
   ├─ Define color/depth attachment formats
   ├─ Specify load/store operations
   ├─ Declare subpass dependencies
   └─ Define attachment layouts

2. Create VkFramebuffer
   ├─ Bind image views to render pass
   ├─ Specify dimensions
   └─ Layer count

3. Begin render pass (vkCmdBeginRenderPass)
   ├─ Provide framebuffer
   ├─ Specify clear values
   └─ Define render area

4. Record rendering commands
   └─ Bind pipelines compatible with render pass

5. End render pass (vkCmdEndRenderPass)
```

### Current File Structure

**Core Files Involved**:
```
teikitu_private/src/TgS KERNEL/
├── TgS (VULKAN) Kernel [GPU] - Internal - Type.h
│   └── Structure definitions (STg2_KN_GPU_HLSL_*, render pass handles)
│
├── TgS (VULKAN) Kernel [GPU] - System [Context] [EXEC].c
│   └── Command recording, begin/end render pass logic
│
├── TgS (VULKAN) Kernel [GPU] - Resource [Render Target].c
│   └── Render pass creation, framebuffer management
│
├── TgS (VULKAN) Kernel [GPU] - Resource [Pipeline].c
│   └── Pipeline creation with render pass compatibility
│
└── TgS (VULKAN) Kernel [GPU] - Debug [Render Context].c
    └── Debug rendering, overlay passes
```

### Data Structures (Current)

```c
// Typical current structures (hypothetical based on naming conventions)
typedef struct STg2_KN_GPU_Render_Pass {
    VkRenderPass                        vk_render_pass;
    TgUINT_E32                          nuiAttachments;
    VkAttachmentDescription             asAttachments[KTgKN_GPU_MAX_ATTACHMENTS];
    VkSubpassDescription                sSubpass;
    VkSubpassDependency                 asDependencies[2];
} STg2_KN_GPU_Render_Pass;

typedef struct STg2_KN_GPU_Framebuffer {
    VkFramebuffer                       vk_framebuffer;
    TgKN_GPU_RENDER_PASS_ID             tiRender_Pass;
    TgUINT_E32                          nuiAttachments;
    VkImageView                         asAttachments[KTgKN_GPU_MAX_ATTACHMENTS];
    TgUINT_E32                          uiWidth;
    TgUINT_E32                          uiHeight;
} STg2_KN_GPU_Framebuffer;

typedef struct STg2_KN_GPU_Pipeline {
    VkPipeline                          vk_pipeline;
    VkPipelineLayout                    vk_pipeline_layout;
    TgKN_GPU_RENDER_PASS_ID             tiRender_Pass; // <-- Dependency to remove
} STg2_KN_GPU_Pipeline;
```

---

## Target Architecture

### Dynamic Rendering Model

**New Flow**:
```
1. Create graphics pipeline with dynamic rendering state
   ├─ Specify color attachment formats
   ├─ Specify depth/stencil format
   └─ No render pass required

2. Begin rendering (vkCmdBeginRendering)
   ├─ Provide VkRenderingInfo structure
   │   ├─ Color attachment info (image views, layouts, load/store ops)
   │   ├─ Depth attachment info (if used)
   │   └─ Stencil attachment info (if used)
   └─ Specify render area

3. Record rendering commands
   └─ Bind pipelines (format-compatible)

4. End rendering (vkCmdEndRendering)
```

### New Data Structures

```c
// Proposed new structures
typedef struct STg2_KN_GPU_Dynamic_Rendering_State {
    TgUINT_E32                          nuiColorAttachments;
    VkFormat                            asColorFormats[KTgKN_GPU_MAX_COLOR_ATTACHMENTS];
    VkFormat                            enDepthFormat;
    VkFormat                            enStencilFormat;
    VkSampleCountFlagBits               enSamples;
} STg2_KN_GPU_Dynamic_Rendering_State;

typedef struct STg2_KN_GPU_Render_Target_Info {
    VkImageView                         vk_image_view;
    VkImageLayout                       enImageLayout;
    VkResolveModeFlagBits               enResolveMode;
    VkImageView                         vk_resolve_image_view;
    VkImageLayout                       enResolveImageLayout;
    VkAttachmentLoadOp                  enLoadOp;
    VkAttachmentStoreOp                 enStoreOp;
    VkClearValue                        sClearValue;
} STg2_KN_GPU_Render_Target_Info;

typedef struct STg2_KN_GPU_Pipeline {
    VkPipeline                          vk_pipeline;
    VkPipelineLayout                    vk_pipeline_layout;
    STg2_KN_GPU_Dynamic_Rendering_State sDynamic_Rendering_State; // <-- New
} STg2_KN_GPU_Pipeline;
```

### API Changes Summary

| Current API | New API | Purpose |
|-------------|---------|---------|
| `vkCreateRenderPass` | *(removed)* | Eliminated |
| `vkCreateFramebuffer` | *(removed)* | Eliminated |
| `vkCmdBeginRenderPass` | `vkCmdBeginRendering` | Start rendering |
| `vkCmdEndRenderPass` | `vkCmdEndRendering` | End rendering |
| `VkGraphicsPipelineCreateInfo::renderPass` | `VkPipelineRenderingCreateInfo` | Pipeline creation |

---

## Migration Phases

### Phase 1: Preparation and Feature Detection (Week 1)

**Objectives**:
- Add Vulkan 1.3 / VK_KHR_dynamic_rendering feature detection
- Update CMake configuration for Vulkan 1.3 SDK requirement
- Add runtime validation for dynamic rendering support

**Tasks**:
1. Update CMake to require Vulkan 1.3 SDK
   ```cmake
   find_package(Vulkan 1.3 REQUIRED)
   ```

2. Add feature detection in device initialization
   ```c
   TgRESULT tgKN_GPU_EXT__Init_Device_Features(
       VkPhysicalDevice vk_physical_device,
       VkPhysicalDeviceFeatures2 *OUT psFeatures2
   ) {
       VkPhysicalDeviceDynamicRenderingFeatures *psDynamicRendering = 
           TgMALLOC_POOL(sizeof(VkPhysicalDeviceDynamicRenderingFeatures));
       
       psDynamicRendering->sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DYNAMIC_RENDERING_FEATURES;
       psDynamicRendering->pNext = psFeatures2->pNext;
       psDynamicRendering->dynamicRendering = VK_TRUE;
       
       psFeatures2->pNext = psDynamicRendering;
       
       vkGetPhysicalDeviceFeatures2(vk_physical_device, psFeatures2);
       
       if (!psDynamicRendering->dynamicRendering) {
           return KTgE_FAIL;
       }
       
       return KTgS_OK;
   }
   ```

3. Update validation layers configuration

**Deliverables**:
- [ ] CMake changes committed
- [ ] Feature detection code added
- [ ] Validation tests passing

---

### Phase 2: Structure and Type Updates (Week 1-2)

**Objectives**:
- Update internal type definitions
- Remove render pass and framebuffer structures
- Add dynamic rendering state structures

**File**: `TgS (VULKAN) Kernel [GPU] - Internal - Type.h`

**Changes**:

1. **Add new dynamic rendering structures**:
   ```c
   // Dynamic rendering state for pipeline creation
   typedef struct STg2_KN_GPU_Dynamic_Rendering_State {
       TgUINT_E32                          nuiColorAttachments;
       VkFormat                            asColorFormats[KTgKN_GPU_MAX_COLOR_ATTACHMENTS];
       VkFormat                            enDepthFormat;
       VkFormat                            enStencilFormat;
       VkSampleCountFlagBits               enSamples;
   } STg2_KN_GPU_Dynamic_Rendering_State;
   
   // Rendering attachment info (used at begin rendering time)
   typedef struct STg2_KN_GPU_Rendering_Attachment_Info {
       VkImageView                         vk_image_view;
       VkImageLayout                       enImageLayout;
       VkResolveModeFlagBits               enResolveMode;
       VkImageView                         vk_resolve_image_view;
       VkImageLayout                       enResolveImageLayout;
       VkAttachmentLoadOp                  enLoadOp;
       VkAttachmentStoreOp                 enStoreOp;
       VkClearValue                        sClearValue;
   } STg2_KN_GPU_Rendering_Attachment_Info;
   ```

2. **Remove deprecated structures**:
   ```c
   // REMOVE:
   // typedef struct STg2_KN_GPU_Render_Pass { ... };
   // typedef struct STg2_KN_GPU_Framebuffer { ... };
   ```

3. **Update pipeline structure**:
   ```c
   typedef struct STg2_KN_GPU_Pipeline {
       VkPipeline                          vk_pipeline;
       VkPipelineLayout                    vk_pipeline_layout;
       STg2_KN_GPU_Dynamic_Rendering_State sDynamic_Rendering_State; // NEW
       // REMOVE: TgKN_GPU_RENDER_PASS_ID  tiRender_Pass;
   } STg2_KN_GPU_Pipeline;
   ```

**Deliverables**:
- [ ] Structure definitions updated
- [ ] Compilation successful (may have link errors)
- [ ] Documentation updated

---

### Phase 3: Pipeline Creation Refactor (Week 2)

**Objectives**:
- Update pipeline creation to use dynamic rendering
- Remove render pass parameter from pipeline creation
- Add format specification

**File**: `TgS (VULKAN) Kernel [GPU] - Resource [Pipeline].c`

**Current Pattern** (hypothetical):
```c
TgRESULT tgKN_GPU_EXT__Pipeline__Create(
    STg2_KN_GPU_Pipeline *OUT psPipeline,
    TgKN_GPU_RENDER_PASS_ID tiRender_Pass,
    /* ... other parameters ... */
) {
    VkGraphicsPipelineCreateInfo sPipeline_CI = {
        .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
        .renderPass = g_asRender_Pass[tiRender_Pass.m_uiI].vk_render_pass,
        .subpass = 0,
        /* ... */
    };
    
    vkCreateGraphicsPipelines(/* ... */);
}
```

**New Pattern**:
```c
TgRESULT tgKN_GPU_EXT__Pipeline__Create(
    STg2_KN_GPU_Pipeline *OUT psPipeline,
    STg2_KN_GPU_Dynamic_Rendering_State *IN psDynamic_Rendering_State,
    /* ... other parameters ... */
) {
    // Create pipeline rendering info
    VkPipelineRenderingCreateInfo sPipeline_Rendering_CI = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_RENDERING_CREATE_INFO,
        .pNext = nullptr,
        .viewMask = 0,
        .colorAttachmentCount = psDynamic_Rendering_State->nuiColorAttachments,
        .pColorAttachmentFormats = psDynamic_Rendering_State->asColorFormats,
        .depthAttachmentFormat = psDynamic_Rendering_State->enDepthFormat,
        .stencilAttachmentFormat = psDynamic_Rendering_State->enStencilFormat
    };
    
    VkGraphicsPipelineCreateInfo sPipeline_CI = {
        .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
        .pNext = &sPipeline_Rendering_CI, // Chain rendering info
        .renderPass = VK_NULL_HANDLE,     // No render pass needed
        .subpass = 0,
        /* ... */
    };
    
    vkCreateGraphicsPipelines(/* ... */);
    
    // Store dynamic rendering state for validation
    psPipeline->sDynamic_Rendering_State = *psDynamic_Rendering_State;
}
```

**Key Changes**:
1. Replace `tiRender_Pass` parameter with `psDynamic_Rendering_State`
2. Add `VkPipelineRenderingCreateInfo` to pipeline creation
3. Set `renderPass` to `VK_NULL_HANDLE`
4. Store format information in pipeline structure

**Deliverables**:
- [ ] Pipeline creation refactored
- [ ] All pipeline creation call sites updated
- [ ] Unit tests updated

---

### Phase 4: Render Pass Removal (Week 2-3)

**Objectives**:
- Remove render pass and framebuffer creation code
- Clean up unused functions and structures

**File**: `TgS (VULKAN) Kernel [GPU] - Resource [Render Target].c`

**Functions to Remove**:
```c
// REMOVE these functions:
TgRESULT tgKN_GPU_EXT__Render_Pass__Create(/* ... */);
TgRESULT tgKN_GPU_EXT__Render_Pass__Destroy(/* ... */);
TgRESULT tgKN_GPU_EXT__Framebuffer__Create(/* ... */);
TgRESULT tgKN_GPU_EXT__Framebuffer__Destroy(/* ... */);
```

**Functions to Modify**:
```c
// Simplify render target creation - no render pass needed
TgRESULT tgKN_GPU_EXT__Render_Target__Create(
    STg2_KN_GPU_Render_Target *OUT psRender_Target,
    VkFormat enColor_Format,
    VkFormat enDepth_Format,
    TgUINT_E32 uiWidth,
    TgUINT_E32 uiHeight
) {
    // Create image views
    // No render pass or framebuffer creation needed
    // Store format information for later use
    psRender_Target->enColor_Format = enColor_Format;
    psRender_Target->enDepth_Format = enDepth_Format;
    
    return KTgS_OK;
}
```

**Deliverables**:
- [ ] Render pass creation code removed
- [ ] Framebuffer creation code removed
- [ ] Cleanup verified
- [ ] No dangling references

---

### Phase 5: Command Recording Update (Week 3)

**Objectives**:
- Replace `vkCmdBeginRenderPass`/`vkCmdEndRenderPass` with dynamic rendering
- Update command buffer recording logic

**File**: `TgS (VULKAN) Kernel [GPU] - System [Context] [EXEC].c`

**Current Pattern**:
```c
TgRESULT tgKN_GPU_EXT__Execute__Begin_Render_Pass(
    VkCommandBuffer vk_cmd_buffer,
    TgKN_GPU_RENDER_PASS_ID tiRender_Pass,
    TgKN_GPU_FRAMEBUFFER_ID tiFramebuffer
) {
    VkRenderPassBeginInfo sBegin_Info = {
        .sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
        .renderPass = g_asRender_Pass[tiRender_Pass.m_uiI].vk_render_pass,
        .framebuffer = g_asFramebuffer[tiFramebuffer.m_uiI].vk_framebuffer,
        .renderArea = { /* ... */ },
        .clearValueCount = /* ... */,
        .pClearValues = /* ... */
    };
    
    vkCmdBeginRenderPass(vk_cmd_buffer, &sBegin_Info, VK_SUBPASS_CONTENTS_INLINE);
    return KTgS_OK;
}

TgRESULT tgKN_GPU_EXT__Execute__End_Render_Pass(
    VkCommandBuffer vk_cmd_buffer
) {
    vkCmdEndRenderPass(vk_cmd_buffer);
    return KTgS_OK;
}
```

**New Pattern**:
```c
TgRESULT tgKN_GPU_EXT__Execute__Begin_Rendering(
    VkCommandBuffer vk_cmd_buffer,
    TgUINT_E32 uiNum_Color_Attachments,
    STg2_KN_GPU_Rendering_Attachment_Info *IN asColor_Attachments,
    STg2_KN_GPU_Rendering_Attachment_Info *IN psDepth_Attachment,
    VkRect2D sRender_Area
) {
    // Prepare color attachment info structures
    VkRenderingAttachmentInfo asColor_Attachment_Infos[KTgKN_GPU_MAX_COLOR_ATTACHMENTS];
    
    for (TgUINT_E32 uiIndex = 0; uiIndex < uiNum_Color_Attachments; ++uiIndex) {
        asColor_Attachment_Infos[uiIndex] = (VkRenderingAttachmentInfo){
            .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
            .pNext = nullptr,
            .imageView = asColor_Attachments[uiIndex].vk_image_view,
            .imageLayout = asColor_Attachments[uiIndex].enImageLayout,
            .resolveMode = asColor_Attachments[uiIndex].enResolveMode,
            .resolveImageView = asColor_Attachments[uiIndex].vk_resolve_image_view,
            .resolveImageLayout = asColor_Attachments[uiIndex].enResolveImageLayout,
            .loadOp = asColor_Attachments[uiIndex].enLoadOp,
            .storeOp = asColor_Attachments[uiIndex].enStoreOp,
            .clearValue = asColor_Attachments[uiIndex].sClearValue
        };
    }
    
    // Prepare depth attachment info (if present)
    VkRenderingAttachmentInfo sDepth_Attachment_Info = {0};
    VkRenderingAttachmentInfo *psDepth_Info = nullptr;
    
    if (psDepth_Attachment && psDepth_Attachment->vk_image_view != VK_NULL_HANDLE) {
        sDepth_Attachment_Info = (VkRenderingAttachmentInfo){
            .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
            .pNext = nullptr,
            .imageView = psDepth_Attachment->vk_image_view,
            .imageLayout = psDepth_Attachment->enImageLayout,
            .resolveMode = VK_RESOLVE_MODE_NONE,
            .loadOp = psDepth_Attachment->enLoadOp,
            .storeOp = psDepth_Attachment->enStoreOp,
            .clearValue = psDepth_Attachment->sClearValue
        };
        psDepth_Info = &sDepth_Attachment_Info;
    }
    
    // Create rendering info
    VkRenderingInfo sRendering_Info = {
        .sType = VK_STRUCTURE_TYPE_RENDERING_INFO,
        .pNext = nullptr,
        .flags = 0,
        .renderArea = sRender_Area,
        .layerCount = 1,
        .viewMask = 0,
        .colorAttachmentCount = uiNum_Color_Attachments,
        .pColorAttachments = asColor_Attachment_Infos,
        .pDepthAttachment = psDepth_Info,
        .pStencilAttachment = nullptr // Can be same as depth or separate
    };
    
    vkCmdBeginRendering(vk_cmd_buffer, &sRendering_Info);
    return KTgS_OK;
}

TgRESULT tgKN_GPU_EXT__Execute__End_Rendering(
    VkCommandBuffer vk_cmd_buffer
) {
    vkCmdEndRendering(vk_cmd_buffer);
    return KTgS_OK;
}
```

**Key Changes**:
1. Replace render pass/framebuffer IDs with attachment info structures
2. Build `VkRenderingInfo` dynamically
3. Use `vkCmdBeginRendering`/`vkCmdEndRendering`

**Deliverables**:
- [ ] Begin/End rendering functions updated
- [ ] All call sites updated
- [ ] Command recording tests passing

---

### Phase 6: Debug Rendering Update (Week 3)

**Objectives**:
- Update debug overlay rendering
- Ensure ImGui/debug UI compatibility

**File**: `TgS (VULKAN) Kernel [GPU] - Debug [Render Context].c`

**Example Update**:
```c
TgRESULT tgKN_GPU_EXT__Debug__Render_UI(
    VkCommandBuffer vk_cmd_buffer,
    VkImageView vk_target_view,
    VkFormat enTarget_Format
) {
    // Prepare attachment for debug rendering
    STg2_KN_GPU_Rendering_Attachment_Info sColor_Attachment = {
        .vk_image_view = vk_target_view,
        .enImageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
        .enResolveMode = VK_RESOLVE_MODE_NONE,
        .vk_resolve_image_view = VK_NULL_HANDLE,
        .enLoadOp = VK_ATTACHMENT_LOAD_OP_LOAD, // Preserve existing content
        .enStoreOp = VK_ATTACHMENT_STORE_OP_STORE,
        .sClearValue = {0}
    };
    
    VkRect2D sRender_Area = {
        .offset = {0, 0},
        .extent = {/* swapchain extent */}
    };
    
    // Begin rendering for debug overlay
    tgKN_GPU_EXT__Execute__Begin_Rendering(
        vk_cmd_buffer,
        1, // Single color attachment
        &sColor_Attachment,
        nullptr, // No depth for UI
        sRender_Area
    );
    
    // Render debug UI elements
    // ...
    
    tgKN_GPU_EXT__Execute__End_Rendering(vk_cmd_buffer);
    
    return KTgS_OK;
}
```

**Deliverables**:
- [ ] Debug rendering updated
- [ ] UI overlay tests passing
- [ ] Visual validation complete

---

### Phase 7: Testing and Validation (Week 4)

**Objectives**:
- Comprehensive testing of all rendering paths
- Performance benchmarking
- Visual regression testing

**Test Categories**:

1. **Functional Tests**:
   - Single color attachment rendering
   - Multiple color attachments (MRT)
   - Depth-only rendering
   - Color + depth rendering
   - MSAA rendering with resolve
   - Different clear operations (load, clear, don't care)
   - Different store operations (store, don't care)

2. **Integration Tests**:
   - Full frame rendering
   - Multiple render passes per frame
   - Render to texture
   - Post-processing chains
   - Debug overlay rendering

3. **Performance Tests**:
   - Frame time comparison (before/after)
   - CPU time for command recording
   - Memory usage
   - Pipeline creation time

4. **Visual Tests**:
   - Screenshot comparison with reference images
   - Render different scenes
   - Validate color correctness
   - Validate depth buffer correctness

**Validation Checklist**:
- [ ] All existing shaders work without modification
- [ ] Rendered output matches pre-migration reference
- [ ] No validation errors from Vulkan layers
- [ ] No memory leaks detected
- [ ] Performance is equivalent or better
- [ ] All platforms tested (Windows, Linux, macOS if applicable)

---

## Technical Implementation Details

### Feature Detection and Initialization

```c
// 1. Check for dynamic rendering support
VkPhysicalDeviceVulkan13Features vk13_features = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_13_FEATURES,
    .dynamicRendering = VK_TRUE
};

VkPhysicalDeviceFeatures2 features2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2,
    .pNext = &vk13_features
};

vkGetPhysicalDeviceFeatures2(vk_physical_device, &features2);

if (!vk13_features.dynamicRendering) {
    // Fallback: Check for VK_KHR_dynamic_rendering extension
    // For Vulkan 1.1/1.2 compatibility
}

// 2. Enable during device creation
VkDeviceCreateInfo device_ci = {
    .sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO,
    .pNext = &vk13_features, // Chain features
    // ...
};

vkCreateDevice(vk_physical_device, &device_ci, nullptr, &vk_device);
```

### Pipeline Creation Details

**Before** (with render pass):
```c
VkGraphicsPipelineCreateInfo pipeline_ci = {
    .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
    .stageCount = 2,
    .pStages = shader_stages,
    .pVertexInputState = &vertex_input,
    .pInputAssemblyState = &input_assembly,
    .pViewportState = &viewport_state,
    .pRasterizationState = &rasterizer,
    .pMultisampleState = &multisampling,
    .pDepthStencilState = &depth_stencil,
    .pColorBlendState = &color_blending,
    .layout = pipeline_layout,
    .renderPass = render_pass,      // <-- Required
    .subpass = 0,
    .basePipelineHandle = VK_NULL_HANDLE
};
```

**After** (dynamic rendering):
```c
VkPipelineRenderingCreateInfo pipeline_rendering_ci = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_RENDERING_CREATE_INFO,
    .colorAttachmentCount = 1,
    .pColorAttachmentFormats = (VkFormat[]){ VK_FORMAT_B8G8R8A8_SRGB },
    .depthAttachmentFormat = VK_FORMAT_D32_SFLOAT,
    .stencilAttachmentFormat = VK_FORMAT_UNDEFINED
};

VkGraphicsPipelineCreateInfo pipeline_ci = {
    .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
    .pNext = &pipeline_rendering_ci, // <-- Chain rendering info
    .stageCount = 2,
    .pStages = shader_stages,
    .pVertexInputState = &vertex_input,
    .pInputAssemblyState = &input_assembly,
    .pViewportState = &viewport_state,
    .pRasterizationState = &rasterizer,
    .pMultisampleState = &multisampling,
    .pDepthStencilState = &depth_stencil,
    .pColorBlendState = &color_blending,
    .layout = pipeline_layout,
    .renderPass = VK_NULL_HANDLE,    // <-- NULL for dynamic rendering
    .subpass = 0,
    .basePipelineHandle = VK_NULL_HANDLE
};
```

### Command Recording Details

**Multiple Render Targets Example**:
```c
// Setup multiple color attachments
VkRenderingAttachmentInfo color_attachments[2] = {
    {
        .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
        .imageView = albedo_view,
        .imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
        .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
        .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
        .clearValue = {.color = {0.0f, 0.0f, 0.0f, 1.0f}}
    },
    {
        .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
        .imageView = normal_view,
        .imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
        .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
        .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
        .clearValue = {.color = {0.5f, 0.5f, 1.0f, 1.0f}}
    }
};

VkRenderingAttachmentInfo depth_attachment = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
    .imageView = depth_view,
    .imageLayout = VK_IMAGE_LAYOUT_DEPTH_ATTACHMENT_OPTIMAL,
    .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
    .storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE,
    .clearValue = {.depthStencil = {1.0f, 0}}
};

VkRenderingInfo rendering_info = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_INFO,
    .renderArea = {{0, 0}, {width, height}},
    .layerCount = 1,
    .colorAttachmentCount = 2,
    .pColorAttachments = color_attachments,
    .pDepthAttachment = &depth_attachment
};

vkCmdBeginRendering(cmd_buffer, &rendering_info);
// Draw commands...
vkCmdEndRendering(cmd_buffer);
```

### MSAA with Resolve Example

```c
VkRenderingAttachmentInfo color_attachment = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
    .imageView = msaa_view,               // MSAA source
    .imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
    .resolveMode = VK_RESOLVE_MODE_AVERAGE_BIT,
    .resolveImageView = resolved_view,     // Resolve target
    .resolveImageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
    .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
    .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
    .clearValue = {.color = {0.0f, 0.0f, 0.0f, 1.0f}}
};
// Resolve happens automatically at vkCmdEndRendering
```

---

## File-by-File Changes

### 1. TgS (VULKAN) Kernel [GPU] - Internal - Type.h

**Change Type**: Structure modifications

**Additions**:
```c
// New structures for dynamic rendering
typedef struct STg2_KN_GPU_Dynamic_Rendering_State {
    TgUINT_E32                          nuiColorAttachments;
    VkFormat                            asColorFormats[KTgKN_GPU_MAX_COLOR_ATTACHMENTS];
    VkFormat                            enDepthFormat;
    VkFormat                            enStencilFormat;
    VkSampleCountFlagBits               enSamples;
} STg2_KN_GPU_Dynamic_Rendering_State;

typedef struct STg2_KN_GPU_Rendering_Attachment_Info {
    VkImageView                         vk_image_view;
    VkImageLayout                       enImageLayout;
    VkResolveModeFlagBits               enResolveMode;
    VkImageView                         vk_resolve_image_view;
    VkImageLayout                       enResolveImageLayout;
    VkAttachmentLoadOp                  enLoadOp;
    VkAttachmentStoreOp                 enStoreOp;
    VkClearValue                        sClearValue;
} STg2_KN_GPU_Rendering_Attachment_Info;
```

**Removals**:
```c
// Remove render pass structures
// typedef struct STg2_KN_GPU_Render_Pass { ... };
// typedef struct STg2_KN_GPU_Framebuffer { ... };
```

**Modifications**:
```c
// Update pipeline structure
typedef struct STg2_KN_GPU_Pipeline {
    VkPipeline                          vk_pipeline;
    VkPipelineLayout                    vk_pipeline_layout;
    STg2_KN_GPU_Dynamic_Rendering_State sDynamic_Rendering_State; // ADDED
    // TgKN_GPU_RENDER_PASS_ID          tiRender_Pass; // REMOVED
} STg2_KN_GPU_Pipeline;
```

---

### 2. TgS (VULKAN) Kernel [GPU] - System [Context] [EXEC].c

**Change Type**: Command recording refactor

**Function Changes**:
- Remove: `tgKN_GPU_EXT__Execute__Begin_Render_Pass`
- Remove: `tgKN_GPU_EXT__Execute__End_Render_Pass`
- Add: `tgKN_GPU_EXT__Execute__Begin_Rendering`
- Add: `tgKN_GPU_EXT__Execute__End_Rendering`

**New Function Signatures**:
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

---

### 3. TgS (VULKAN) Kernel [GPU] - Resource [Render Target].c

**Change Type**: Render pass/framebuffer removal

**Function Removals**:
```c
// Remove all render pass creation/destruction
// TgRESULT tgKN_GPU_EXT__Render_Pass__Create(...);
// TgRESULT tgKN_GPU_EXT__Render_Pass__Destroy(...);
// TgRESULT tgKN_GPU_EXT__Framebuffer__Create(...);
// TgRESULT tgKN_GPU_EXT__Framebuffer__Destroy(...);
```

**Function Simplifications**:
```c
// Simplified render target creation
TgRESULT tgKN_GPU_EXT__Render_Target__Create(
    STg2_KN_GPU_Render_Target *OUT psRender_Target,
    // ... parameters ...
) {
    // Create images and image views
    // Store format information
    // NO render pass or framebuffer creation
    
    psRender_Target->enColor_Format = enColor_Format;
    psRender_Target->enDepth_Format = enDepth_Format;
    
    return KTgS_OK;
}
```

---

### 4. TgS (VULKAN) Kernel [GPU] - Resource [Pipeline].c

**Change Type**: Pipeline creation refactor

**Function Modification**:
```c
TgRESULT tgKN_GPU_EXT__Pipeline__Create(
    STg2_KN_GPU_Pipeline *OUT psPipeline,
    STg2_KN_GPU_Dynamic_Rendering_State *IN psDynamic_Rendering_State, // NEW PARAM
    // ... other parameters ...
) {
    // Create pipeline rendering info
    VkPipelineRenderingCreateInfo sPipeline_Rendering_CI = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_RENDERING_CREATE_INFO,
        .colorAttachmentCount = psDynamic_Rendering_State->nuiColorAttachments,
        .pColorAttachmentFormats = psDynamic_Rendering_State->asColorFormats,
        .depthAttachmentFormat = psDynamic_Rendering_State->enDepthFormat,
        .stencilAttachmentFormat = psDynamic_Rendering_State->enStencilFormat
    };
    
    VkGraphicsPipelineCreateInfo sPipeline_CI = {
        .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
        .pNext = &sPipeline_Rendering_CI, // Chain
        .renderPass = VK_NULL_HANDLE,      // No render pass
        // ... other fields ...
    };
    
    vkCreateGraphicsPipelines(/* ... */);
    
    // Store state
    psPipeline->sDynamic_Rendering_State = *psDynamic_Rendering_State;
    
    return KTgS_OK;
}
```

---

### 5. TgS (VULKAN) Kernel [GPU] - Debug [Render Context].c

**Change Type**: Debug rendering update

**Pattern Change**:
```c
TgRESULT tgKN_GPU_EXT__Debug__Render(
    VkCommandBuffer vk_cmd_buffer,
    VkImageView vk_target_view,
    VkFormat enTarget_Format
) {
    // Setup attachment info
    STg2_KN_GPU_Rendering_Attachment_Info sColor_Attachment = {
        .vk_image_view = vk_target_view,
        .enImageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
        .enLoadOp = VK_ATTACHMENT_LOAD_OP_LOAD,  // Preserve existing
        .enStoreOp = VK_ATTACHMENT_STORE_OP_STORE,
        // ...
    };
    
    // Begin rendering
    tgKN_GPU_EXT__Execute__Begin_Rendering(
        vk_cmd_buffer,
        1,
        &sColor_Attachment,
        nullptr,
        sRender_Area
    );
    
    // Debug draw commands...
    
    tgKN_GPU_EXT__Execute__End_Rendering(vk_cmd_buffer);
    
    return KTgS_OK;
}
```

---

### 6. CMakeLists.txt (if exists in main repo)

**Change Type**: Vulkan version requirement

```cmake
# Update Vulkan SDK version requirement
find_package(Vulkan 1.3 REQUIRED)

# Add compile definition for dynamic rendering
target_compile_definitions(teikitu_kernel PRIVATE
    VK_API_VERSION_1_3
    TgKN_GPU_USE_DYNAMIC_RENDERING=1
)
```

---

## Validation and Testing

### Validation Layers

Enable all validation during development:
```c
const char *validation_layers[] = {
    "VK_LAYER_KHRONOS_validation"
};

// Enable best practices warnings
VkValidationFeatureEnableEXT enables[] = {
    VK_VALIDATION_FEATURE_ENABLE_BEST_PRACTICES_EXT,
    VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT
};

VkValidationFeaturesEXT features = {
    .sType = VK_STRUCTURE_TYPE_VALIDATION_FEATURES_EXT,
    .enabledValidationFeatureCount = 2,
    .pEnabledValidationFeatures = enables
};
```

### Test Scenarios

1. **Basic Rendering**:
   ```c
   // Test: Single color attachment, no depth
   // Expected: Triangle renders correctly
   ```

2. **Depth Testing**:
   ```c
   // Test: Color + depth attachment
   // Expected: Depth test works correctly
   ```

3. **Multiple Render Targets**:
   ```c
   // Test: 2+ color attachments
   // Expected: MRT output correct
   ```

4. **MSAA**:
   ```c
   // Test: MSAA with automatic resolve
   // Expected: Anti-aliased output
   ```

5. **Load Operations**:
   ```c
   // Test: LOAD_OP_LOAD preserves content
   // Expected: Previous content visible
   ```

6. **Store Operations**:
   ```c
   // Test: STORE_OP_DONT_CARE for temp attachments
   // Expected: No validation errors
   ```

### Performance Benchmarks

Measure before and after migration:

| Metric | Target |
|--------|--------|
| Frame time | ≤ 100% of baseline |
| CPU time | ≤ 100% of baseline |
| Memory usage | ≤ 100% of baseline |
| Pipeline creation | ≤ 100% of baseline |
| Validation overhead | Reduced |

### Visual Regression Testing

1. Capture reference screenshots before migration
2. Capture test screenshots after migration
3. Compare pixel-by-pixel (allow for minor GPU differences)
4. Manual visual inspection for quality

---

## Rollback Strategy

### Compatibility Layer (Optional)

Create a compile-time switch to support both paths during transition:

```c
#if defined(TgKN_GPU_USE_DYNAMIC_RENDERING)
    // New dynamic rendering path
    vkCmdBeginRendering(cmd_buffer, &rendering_info);
#else
    // Legacy render pass path
    vkCmdBeginRenderPass(cmd_buffer, &begin_info, VK_SUBPASS_CONTENTS_INLINE);
#endif
```

### Rollback Checklist

If issues arise:
- [ ] Switch compile flag to disable dynamic rendering
- [ ] Revert to previous commit
- [ ] Identify specific failure case
- [ ] Create isolated test
- [ ] Fix and re-test
- [ ] Re-enable dynamic rendering

---

## Performance Considerations

### Expected Improvements

1. **Reduced Validation Overhead**: Dynamic rendering simplifies state tracking
2. **Better Driver Optimization**: Modern drivers optimize for dynamic rendering
3. **Reduced CPU Time**: Less state management and validation
4. **Memory Savings**: No render pass/framebuffer objects

### Potential Concerns

1. **Attachment Info Construction**: May add per-frame CPU overhead
   - **Mitigation**: Cache and reuse attachment info structures

2. **Pipeline Compatibility**: Must ensure format compatibility at runtime
   - **Mitigation**: Validation in debug builds

### Optimization Tips

```c
// Cache attachment info structures
typedef struct STg2_KN_GPU_Cached_Rendering_Info {
    VkRenderingInfo                     sRendering_Info;
    VkRenderingAttachmentInfo           asColor_Attachments[KTgKN_GPU_MAX_COLOR_ATTACHMENTS];
    VkRenderingAttachmentInfo           sDepth_Attachment;
    TgBOOL                              bDirty;
} STg2_KN_GPU_Cached_Rendering_Info;

// Update only when attachments change
if (psCached->bDirty) {
    // Rebuild rendering info
    psCached->bDirty = KTgFALSE;
}

vkCmdBeginRendering(cmd_buffer, &psCached->sRendering_Info);
```

---

## References

### Vulkan Specifications

- [VK_KHR_dynamic_rendering](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_dynamic_rendering.html)
- [Vulkan 1.3 Specification](https://registry.khronos.org/vulkan/specs/1.3/html/)
- [VkRenderingInfo](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkRenderingInfo.html)
- [VkPipelineRenderingCreateInfo](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPipelineRenderingCreateInfo.html)

### Migration Guides

- [Khronos Dynamic Rendering Tutorial](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/extensions/dynamic_rendering)
- [LunarG Dynamic Rendering Guide](https://www.lunarg.com/wp-content/uploads/2022/03/Dynamic-Rendering-Guide.pdf)

### Best Practices

- [Vulkan Guide - Dynamic Rendering](https://vkguide.dev/)
- [ARM Vulkan Best Practices - Dynamic Rendering](https://arm-software.github.io/vulkan_best_practice_for_mobile_developers/)

---

## Appendix A: Complete Example

### Before: Traditional Render Pass

```c
// Create render pass
VkAttachmentDescription color_attachment = {
    .format = VK_FORMAT_B8G8R8A8_SRGB,
    .samples = VK_SAMPLE_COUNT_1_BIT,
    .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
    .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
    .initialLayout = VK_IMAGE_LAYOUT_UNDEFINED,
    .finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR
};

VkAttachmentReference color_ref = {
    .attachment = 0,
    .layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL
};

VkSubpassDescription subpass = {
    .pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS,
    .colorAttachmentCount = 1,
    .pColorAttachments = &color_ref
};

VkRenderPassCreateInfo render_pass_ci = {
    .sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO,
    .attachmentCount = 1,
    .pAttachments = &color_attachment,
    .subpassCount = 1,
    .pSubpasses = &subpass
};

vkCreateRenderPass(device, &render_pass_ci, nullptr, &render_pass);

// Create framebuffer
VkFramebufferCreateInfo framebuffer_ci = {
    .sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO,
    .renderPass = render_pass,
    .attachmentCount = 1,
    .pAttachments = &image_view,
    .width = width,
    .height = height,
    .layers = 1
};

vkCreateFramebuffer(device, &framebuffer_ci, nullptr, &framebuffer);

// Create pipeline
VkGraphicsPipelineCreateInfo pipeline_ci = {
    .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
    .renderPass = render_pass,
    .subpass = 0,
    // ...
};

vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipeline_ci, nullptr, &pipeline);

// Record commands
VkRenderPassBeginInfo begin_info = {
    .sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
    .renderPass = render_pass,
    .framebuffer = framebuffer,
    .renderArea = {{0, 0}, {width, height}},
    .clearValueCount = 1,
    .pClearValues = &clear_value
};

vkCmdBeginRenderPass(cmd_buffer, &begin_info, VK_SUBPASS_CONTENTS_INLINE);
vkCmdBindPipeline(cmd_buffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
vkCmdDraw(cmd_buffer, 3, 1, 0, 0);
vkCmdEndRenderPass(cmd_buffer);
```

### After: Dynamic Rendering

```c
// Create pipeline (no render pass!)
VkPipelineRenderingCreateInfo pipeline_rendering_ci = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_RENDERING_CREATE_INFO,
    .colorAttachmentCount = 1,
    .pColorAttachmentFormats = &(VkFormat){VK_FORMAT_B8G8R8A8_SRGB}
};

VkGraphicsPipelineCreateInfo pipeline_ci = {
    .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
    .pNext = &pipeline_rendering_ci,
    .renderPass = VK_NULL_HANDLE, // NULL!
    // ...
};

vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipeline_ci, nullptr, &pipeline);

// Record commands
VkRenderingAttachmentInfo color_attachment = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
    .imageView = image_view,
    .imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
    .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
    .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
    .clearValue = clear_value
};

VkRenderingInfo rendering_info = {
    .sType = VK_STRUCTURE_TYPE_RENDERING_INFO,
    .renderArea = {{0, 0}, {width, height}},
    .layerCount = 1,
    .colorAttachmentCount = 1,
    .pColorAttachments = &color_attachment
};

vkCmdBeginRendering(cmd_buffer, &rendering_info);
vkCmdBindPipeline(cmd_buffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
vkCmdDraw(cmd_buffer, 3, 1, 0, 0);
vkCmdEndRendering(cmd_buffer);
```

**Key Differences**:
1. ❌ No `vkCreateRenderPass`
2. ❌ No `vkCreateFramebuffer`
3. ✅ Simpler pipeline creation
4. ✅ Inline attachment specification
5. ✅ More flexible rendering

---

## Appendix B: Teikitu Coding Conventions

### Naming Conventions

- **Types**: `STg2_` prefix for structures (e.g., `STg2_KN_GPU_Pipeline`)
- **Functions**: `tgKN_GPU_EXT__` prefix (e.g., `tgKN_GPU_EXT__Execute__Begin_Rendering`)
- **Results**: Use `TgRESULT` return type
- **Constants**: `KTgKN_GPU_` prefix (e.g., `KTgKN_GPU_MAX_ATTACHMENTS`)
- **Member variables**: Hungarian notation
  - `vk_` for Vulkan handles
  - `en` for enums
  - `ui` for unsigned integers
  - `ps` for pointers to structures
  - `as` for arrays of structures

### Function Style

```c
TgRESULT tgKN_GPU_EXT__Function_Name(
    StructType *OUT psOutput,      // Output parameters first
    StructType *IN psInput,        // Then input parameters
    TgUINT_E32 IN uiValue          // Scalar values last
) {
    // Validate inputs
    TgPARAM_CHECK(nullptr != psOutput);
    TgPARAM_CHECK(nullptr != psInput);
    
    // Implementation
    
    return KTgS_OK; // Or KTgE_FAIL, etc.
}
```

### Error Handling

```c
TgRESULT result;

result = tgKN_GPU_EXT__Some_Function(/* ... */);
if (TgFAILED(result)) {
    // Cleanup and return
    return result;
}
```

---

## Summary

This migration plan provides a comprehensive strategy for transitioning from traditional Vulkan render passes to dynamic rendering. The migration is broken into manageable phases, with clear deliverables and validation criteria.

**Key Takeaways**:
1. Dynamic rendering simplifies code and improves flexibility
2. Migration is largely mechanical with clear patterns
3. Proper testing ensures functional equivalence
4. Performance should improve or remain equivalent
5. Alignment with modern Vulkan best practices

**Success Criteria**:
- ✅ All render pass/framebuffer code removed
- ✅ Dynamic rendering used throughout
- ✅ All tests passing
- ✅ Visual output unchanged
- ✅ Performance maintained or improved
- ✅ No validation errors

---

**Document Version**: 1.0  
**Last Updated**: 2025-10-24  
**Author**: Teikitu Engineering Team  
**Status**: Migration Plan - Ready for Implementation
