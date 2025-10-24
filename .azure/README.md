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

### Main Migration Plan
📄 **[DYNAMIC_RENDERING_MIGRATION_PLAN.md](./DYNAMIC_RENDERING_MIGRATION_PLAN.md)**

A comprehensive guide covering:
- Current architecture analysis
- Target architecture design
- Phase-by-phase migration strategy
- Technical implementation details
- File-by-file change specifications
- Validation and testing procedures
- Performance considerations
- Complete code examples

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
