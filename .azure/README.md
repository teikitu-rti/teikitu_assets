# Teikitu Vulkan Dynamic Rendering Migration

## About This Documentation

This directory contains documentation for migrating the Teikitu Vulkan GPU backend to use dynamic rendering instead of traditional render passes.

## Important Notice: Repository Context

⚠️ **Note**: This documentation is hosted in the `teikitu_assets` repository, which is an asset library repository containing reference images and test assets used in Teikitu development.

The **actual implementation** of the Vulkan dynamic rendering migration should occur in the main Teikitu engine repositories:
- **teikitu_release** - Public release repository
- **teikitu_private** - Private development repository (contains actual source code)

The source files referenced in the migration plan (e.g., `teikitu_private/src/TgS KERNEL/TgS (VULKAN) Kernel [GPU] - *.c`) do not exist in this assets repository.

## Purpose of This Documentation

This documentation serves as a **central reference** for the migration strategy that can be:
1. Referenced by developers working on the main engine repositories
2. Used as a blueprint for planning and tracking migration progress
3. Shared across the Teikitu development ecosystem
4. Versioned alongside asset changes that may be affected by rendering changes

## Documentation Contents

### 📄 Main Documents

1. **[DYNAMIC_RENDERING_MIGRATION_PLAN.md](./DYNAMIC_RENDERING_MIGRATION_PLAN.md)** (42KB, 1,398 lines)
   
   Comprehensive migration guide covering:
   - Executive summary and motivation
   - Current architecture analysis
   - Target architecture design
   - 7-phase migration strategy (week-by-week breakdown)
   - Technical implementation details
   - File-by-file change specifications (all 6 files)
   - Validation and testing procedures
   - Performance considerations
   - Complete before/after code examples
   - Teikitu coding conventions
   - References and appendices

2. **[QUICK_REFERENCE.md](./QUICK_REFERENCE.md)** (10KB, 377 lines)
   
   Developer quick reference card with:
   - API comparison table (old vs. new)
   - Minimal code examples
   - Structure and function changes
   - Migration phase checklist
   - Common rendering patterns
   - Troubleshooting guide
   - Performance optimization tips

### 📊 Documentation Statistics

- **Total documentation**: 1,857 lines across 3 files
- **Code examples**: 50+ complete Vulkan snippets
- **Coverage**: All 6 source files from requirements
- **Technical depth**: Full VkRenderingInfo structures, MSAA, MRT examples

## Using This Documentation

### For Engine Developers

If you're implementing the Vulkan dynamic rendering migration in the main Teikitu engine:

1. **Read the full migration plan** to understand the scope and strategy
2. **Follow the phases sequentially** - each phase builds on the previous
3. **Use the code examples** as templates for implementation
4. **Validate at each phase** before proceeding to the next
5. **Reference the technical details** when making architectural decisions

### For Asset Pipeline Developers

If you're working on the asset pipeline and rendering validation:

1. **Understand the rendering changes** to anticipate asset format impacts
2. **Prepare test assets** for validating the new rendering path
3. **Update reference images** after migration is complete
4. **Coordinate with engine developers** on validation requirements

## Migration Status

This is a **planning document**. Implementation status should be tracked in the main engine repository's issue tracker.

## Related Resources

- **Vulkan Specification**: [VK_KHR_dynamic_rendering](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_dynamic_rendering.html)
- **Khronos Samples**: [Dynamic Rendering Sample](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/extensions/dynamic_rendering)
- **Teikitu Main Repository**: (Link to main engine repository)

## Questions or Feedback

For questions about this migration plan or to report issues:
- Open an issue in the appropriate Teikitu repository
- Contact the Teikitu development team
- Refer to the Teikitu contributor guidelines

---

**Last Updated**: 2025-10-24  
**Version**: 1.0  
**Maintained By**: Teikitu Engineering Team
