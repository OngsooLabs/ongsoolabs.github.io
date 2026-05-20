# IapClient

IapClient provides a unified in-app purchase workflow for Unity projects.

## What It Covers
- Product purchase flow integration
- Purchase validation flow coordination with backend services
- Purchase restore support for supported stores
- Runtime status and error handling patterns

## Recommended Setup Flow
1. Import the IapClient package into your project.
2. Configure the purchase settings asset used by your environment.
3. Register your product identifiers for each target store.
4. Connect your validation endpoint and test credentials.
5. Validate purchase and restore flows in a sandbox environment.

## Operational Notes
- Keep product IDs aligned between store consoles and runtime configuration.
- Test both success and failure paths before release.
- Use environment-specific configuration for development, staging, and production.

## Related Documentation
- [Quick Start](quick_start.md)
- [Core Concepts](core_concepts.md)
