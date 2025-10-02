# Changelog

All notable changes to the EDD Release Sync action will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-10-02

### Added
- Initial release of EDD Release Sync action
- Composite action for sending EDD release webhooks
- Support for required inputs: edd_id, version, release_url, webhook_url, webhook_token
- Support for optional inputs: download_url, asset_api_url, test_mode
- Outputs: status, http_code, response
- Comprehensive error handling and validation
- Test mode for webhook validation without updating
- Detailed documentation with multiple usage examples

### Features
- Zero-opinion design - works with any build process
- Git Updater compatibility via asset_api_url
- Customer download support via download_url
- Simple one-step integration
- Proper HTTP error handling
- JSON payload validation
- Output variables for debugging

[1.0.0]: https://github.com/code-atlantic/edd-release-sync/releases/tag/v1.0.0
