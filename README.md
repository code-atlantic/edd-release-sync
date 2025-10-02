# EDD Release Sync

Automatically sync WordPress plugin releases to [Easy Digital Downloads](https://easydigitaldownloads.com) via the [EDD Release Manager](https://github.com/code-atlantic/edd-release-manager) plugin.

## Features

- ✅ **Zero Opinion**: Just sends webhooks - use any build process
- ✅ **Simple**: One step in your workflow
- ✅ **Flexible**: Works with any GitHub release strategy
- ✅ **Reliable**: Proper error handling and validation
- ✅ **Composable**: Use with other actions (@10up, WP Engine, etc.)

## Prerequisites

1. Install [EDD Release Manager](https://github.com/code-atlantic/edd-release-manager) plugin on your WordPress site
2. Configure webhook token in `wp-config.php`:
   ```php
   define( 'EDD_RELEASE_WEBHOOK_TOKEN', 'your_secure_token_here' );
   ```
3. Add GitHub secrets to your repository (Settings → Secrets → Actions)

## Quick Start

### Minimal Example

```yaml
- name: Sync to EDD Store
  uses: code-atlantic/edd-release-sync@v1
  with:
    edd_id: '123456'
    version: '1.2.3'
    release_url: 'https://github.com/user/repo/releases/tag/v1.2.3'
    download_url: 'https://github.com/user/repo/releases/download/v1.2.3/plugin.zip'
    asset_api_url: 'https://api.github.com/repos/user/repo/releases/assets/12345'
    webhook_url: ${{ secrets.EDD_WEBHOOK_URL }}
    webhook_token: ${{ secrets.EDD_WEBHOOK_TOKEN }}
```

### Complete Example

```yaml
name: Release My Plugin

on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      # Build your plugin (your way)
      - uses: actions/checkout@v4

      - name: Build plugin
        run: |
          composer install --no-dev
          npm ci && npm run build
          zip -r my-plugin.zip . -x "*.git*" "node_modules/*"

      # Create GitHub release (your way)
      - name: Create Release
        id: create_release
        uses: softprops/action-gh-release@v1
        with:
          files: my-plugin.zip

      # Get asset API URL for Git Updater
      - name: Get Asset API URL
        id: asset
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          ASSET_URL=$(gh api repos/${{ github.repository }}/releases/tags/${{ github.ref_name }} \
            --jq '.assets[] | select(.name == "my-plugin.zip") | .url')
          echo "url=$ASSET_URL" >> $GITHUB_OUTPUT

      # Sync to EDD (our action)
      - name: Sync to EDD Store
        uses: code-atlantic/edd-release-sync@v1
        with:
          edd_id: ${{ secrets.EDD_PRODUCT_ID }}
          version: ${{ github.ref_name }}
          release_url: ${{ steps.create_release.outputs.url }}
          download_url: 'https://github.com/${{ github.repository }}/releases/download/${{ github.ref_name }}/my-plugin.zip'
          asset_api_url: ${{ steps.asset.outputs.url }}
          webhook_url: ${{ secrets.EDD_WEBHOOK_URL }}
          webhook_token: ${{ secrets.EDD_WEBHOOK_TOKEN }}
```

## Inputs

### Required

| Input | Description | Example |
|-------|-------------|---------|
| `edd_id` | EDD Download/Product ID | `123456` |
| `version` | Release version | `1.2.3` |
| `release_url` | GitHub release URL | `https://github.com/user/repo/releases/tag/v1.2.3` |
| `download_url` | Browser download URL | `https://github.com/user/repo/releases/download/v1.2.3/plugin.zip` |
| `asset_api_url` | GitHub API asset URL | `https://api.github.com/repos/user/repo/releases/assets/12345` |
| `webhook_url` | EDD webhook endpoint | `https://yoursite.com/wp-json/edd-release-manager/v1/webhook` |
| `webhook_token` | Bearer token | `your_secret_token` |

**Important URL Formats:**
- **`download_url`**: Browser download URL (what users click) - used for file processing
- **`asset_api_url`**: GitHub API URL (for Git Updater dropdown) - **NOT** the browser download URL

### Optional

| Input | Description | Default |
|-------|-------------|---------|
| `plugin_slug` | Plugin slug (fallback if `edd_id` invalid) | - |
| `readme_url` | README URL (reserved for future use) | - |
| `test_mode` | Test without updating | `false` |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | `success` or `failed` |
| `http_code` | HTTP response code |
| `response` | Response body from webhook |

## GitHub Secrets Setup

Add these secrets to your repository (Settings → Secrets → Actions):

| Secret | Value | Description |
|--------|-------|-------------|
| `EDD_PRODUCT_ID` | Your EDD product ID | Found in EDD Downloads |
| `EDD_WEBHOOK_URL` | `https://yoursite.com/wp-json/edd-release-manager/v1/webhook` | Your webhook endpoint |
| `EDD_WEBHOOK_TOKEN` | Your webhook token | Configured in wp-config.php |

## Understanding URL Formats

This action requires **two different URL formats** for different purposes:

### Browser Download URL (`download_url`)
```
https://github.com/user/repo/releases/download/v1.2.3/plugin.zip
```
- **Purpose**: File download and processing by EDD Release Manager
- **Format**: Standard GitHub browser download URL
- **Used for**: Customer downloads from their EDD account

### GitHub API Asset URL (`asset_api_url`)
```
https://api.github.com/repos/user/repo/releases/assets/12345
```
- **Purpose**: Git Updater dropdown compatibility
- **Format**: GitHub API URL (not browser URL!)
- **Used for**: Version selection in WordPress admin dashboard

**Why both?** Git Updater's dropdown requires API URLs to fetch asset metadata, while actual file downloads need browser URLs. The EDD Release Manager plugin uses both appropriately.

## Advanced Usage

### Getting Asset API URLs

Use GitHub CLI to get the correct API URL for your release asset:

```yaml
- name: Get Asset API URL
  id: asset
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    # Get API URL for specific asset
    ASSET_URL=$(gh api repos/${{ github.repository }}/releases/tags/${{ github.ref_name }} \
      --jq '.assets[] | select(.name == "my-plugin.zip") | .url')
    echo "url=$ASSET_URL" >> $GITHUB_OUTPUT

- name: Sync to EDD
  uses: code-atlantic/edd-release-sync@v1
  with:
    asset_api_url: ${{ steps.asset.outputs.url }}
    # ... other inputs
```

### Test Mode

Test webhook connectivity without updating your product:

```yaml
- name: Test webhook
  uses: code-atlantic/edd-release-sync@v1
  with:
    edd_id: '123456'
    version: '1.0.0'
    release_url: 'https://github.com/user/repo/releases/tag/v1.0.0'
    download_url: 'https://github.com/user/repo/releases/download/v1.0.0/plugin.zip'
    asset_api_url: 'https://api.github.com/repos/user/repo/releases/assets/12345'
    webhook_url: ${{ secrets.EDD_WEBHOOK_URL }}
    webhook_token: ${{ secrets.EDD_WEBHOOK_TOKEN }}
    test_mode: 'true'
```

### With Error Handling

```yaml
- name: Sync to EDD
  id: edd_sync
  continue-on-error: true
  uses: code-atlantic/edd-release-sync@v1
  with:
    edd_id: ${{ secrets.EDD_PRODUCT_ID }}
    version: ${{ github.ref_name }}
    release_url: ${{ github.event.release.html_url }}
    download_url: 'https://github.com/${{ github.repository }}/releases/download/${{ github.ref_name }}/plugin.zip'
    asset_api_url: ${{ steps.asset.outputs.url }}
    webhook_url: ${{ secrets.EDD_WEBHOOK_URL }}
    webhook_token: ${{ secrets.EDD_WEBHOOK_TOKEN }}

- name: Handle failure
  if: steps.edd_sync.outputs.status == 'failed'
  run: |
    echo "Webhook failed with status ${{ steps.edd_sync.outputs.http_code }}"
    echo "Response: ${{ steps.edd_sync.outputs.response }}"
```

### With Multiple Products

```yaml
- name: Sync Free Version
  uses: code-atlantic/edd-release-sync@v1
  with:
    edd_id: ${{ secrets.EDD_FREE_ID }}
    version: ${{ github.ref_name }}
    release_url: ${{ steps.release.outputs.url }}
    download_url: ${{ steps.release.outputs.download_url }}
    asset_api_url: ${{ steps.asset.outputs.url }}
    webhook_url: ${{ secrets.EDD_WEBHOOK_URL }}
    webhook_token: ${{ secrets.EDD_WEBHOOK_TOKEN }}

- name: Sync Pro Version
  uses: code-atlantic/edd-release-sync@v1
  with:
    edd_id: ${{ secrets.EDD_PRO_ID }}
    version: ${{ github.ref_name }}
    release_url: ${{ steps.release.outputs.url }}
    download_url: ${{ steps.release.outputs.download_url }}
    asset_api_url: ${{ steps.asset.outputs.url }}
    webhook_url: ${{ secrets.EDD_WEBHOOK_URL }}
    webhook_token: ${{ secrets.EDD_WEBHOOK_TOKEN }}
```

## Troubleshooting

### "Authentication failed"
- Verify `EDD_WEBHOOK_TOKEN` secret matches token in wp-config.php
- Check Authorization header is being sent correctly

### "Download not found"
- Verify `EDD_PRODUCT_ID` matches actual EDD download ID in your store
- Check product exists and is published

### "Webhook timeout"
- Check your WordPress site is accessible from GitHub Actions
- Verify firewall isn't blocking GitHub's IP addresses
- Check WordPress debug log for errors

### Enable Debug Output

Add `ACTIONS_STEP_DEBUG` secret with value `true` to see detailed action output.

## Why This Action?

Unlike opinionated workflow templates, this action:
- ✅ Works with **any** build process (webpack, vite, composer, whatever)
- ✅ Works with **any** release strategy (tags, releases, manual)
- ✅ Works with **any** other actions (10up, WP Engine, etc.)
- ✅ Does **one thing well**: sends webhooks
- ❌ Doesn't force you into a specific build process
- ❌ Doesn't create releases for you
- ❌ Doesn't build your assets

**You control everything**. We just send the webhook. 🚀

## Related Projects

- [EDD Release Manager](https://github.com/code-atlantic/edd-release-manager) - WordPress plugin for automated EDD releases
- [Git Updater](https://git-updater.com/) - Plugin updater for GitHub-hosted plugins

## License

GPL-2.0-or-later - Same as WordPress

## Support

- **Issues**: [GitHub Issues](https://github.com/code-atlantic/edd-release-sync/issues)
- **Documentation**: [EDD Release Manager Docs](https://github.com/code-atlantic/edd-release-manager#readme)
- **Community**: [WordPress.org Forums](https://wordpress.org/support/)

## Credits

Built with ❤️ by [Code Atlantic](https://code-atlantic.com) for the WordPress community.

---

**Make your EDD releases automatic.** Install the [EDD Release Manager plugin](https://github.com/code-atlantic/edd-release-manager) and use this action to automate everything.
