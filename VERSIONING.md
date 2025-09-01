# Documentation Versioning Guide

This guide explains how to add new versions to the PreBoot documentation.

## Current Setup

The documentation uses a **hybrid versioning approach**:
- **Latest version** uses clean URLs without version prefix (`/docs/`)
- **Older versions** use versioned URLs (`/docs/v1.0/`, `/docs/v1.1/`, etc.)
- **Version configuration** is centralized in `_config.yml`
- **Navigation** can be version-specific using separate navigation files

## File Structure

```
_config.yml                    # Version configuration
_data/
├── navigation.yml             # Latest version navigation
├── navigation-v1.0.yml        # Version-specific navigation (when needed)
└── navigation-v1.1.yml        # Version-specific navigation (when needed)
_docs/                         # Latest version content
├── getting-started/
├── backend/
├── frontend/
└── reference-app/
_docs/v1.0/                    # Archived version content
├── getting-started/
├── backend/
├── frontend/
└── reference-app/
_layouts/documentation.html    # Version-aware layout
```

## Adding a New Version

### Scenario 1: Release New Version (Archive Current as v1.0)

When releasing PreBoot v1.1 and you want to archive current docs as v1.0:

#### 1. Update Version Configuration

Edit `_config.yml`:

```yaml
# Versioning
preboot:
  version: "1.1.0"  # New current version
  versions:
    - name: "latest"
      label: "Latest (1.1.0)"
      version: "1.1.0"
      default: true
    - name: "v1.0"
      label: "v1.0.3"
      version: "1.0.3"
```

#### 2. Create Version-Specific Directory

```bash
# Create directory for old version
mkdir -p _docs/v1.0

# Copy current documentation to versioned directory
cp -r _docs/* _docs/v1.0/
# Note: This will also copy the v1.0 directory into itself, which is fine
# or use rsync to exclude it:
rsync -av --exclude 'v1.0' _docs/ _docs/v1.0/
```

#### 3. Create Version-Specific Navigation

```bash
# Copy current navigation for the old version
cp _data/navigation.yml _data/navigation-v1.0.yml
```

#### 4. Update URLs in Version-Specific Navigation

Edit `_data/navigation-v1.0.yml` and update all URLs:

```yaml
# Change from:
url: /docs/getting-started/quick-start/
# To:
url: /docs/v1.0/getting-started/quick-start/
```

**Tip**: Use find and replace:
- Find: `/docs/`
- Replace: `/docs/v1.0/`

#### 5. Update Permalinks in Versioned Content

For each `.md` file in `_docs/v1.0/`, update the permalink and add version:

```yaml
---
layout: documentation
title: "Quick Start"
subtitle: "Build your first application with PreBoot in minutes."
permalink: /docs/v1.0/getting-started/quick-start/  # Add v1.0 prefix
version: v1.0  # Add version field
section: getting-started
---
```

**Tip**: Use this bash command to update all files:

```bash
# Add version field to all files
find _docs/v1.0 -name "*.md" -exec sed -i '' '2a\
version: v1.0' {} \;

# Update permalinks (macOS)
find _docs/v1.0 -name "*.md" -exec sed -i '' 's|permalink: /docs/|permalink: /docs/v1.0/|g' {} \;
```

#### 6. Update Current Documentation

Update `_docs/` with new v1.1 content and ensure navigation.yml reflects new structure.

#### 7. Restart Jekyll

```bash
# Stop server (Ctrl+C)
bundle exec jekyll serve --livereload
```

### Scenario 2: Add Pre-release Version (Beta/RC)

To add a beta version alongside current stable:

#### 1. Update Configuration

```yaml
preboot:
  version: "1.1.0"
  versions:
    - name: "latest"
      label: "Latest (1.0.3)"
      version: "1.0.3"
      default: true
    - name: "beta"
      label: "Beta (1.1.0-beta)"
      version: "1.1.0-beta"
```

#### 2. Create Beta Content

```bash
mkdir -p _docs/beta
# Add beta-specific content
```

#### 3. Create Beta Navigation

```bash
cp _data/navigation.yml _data/navigation-beta.yml
# Update URLs to use /docs/beta/ prefix
```

## Maintenance

### Updating Version Numbers

When releasing patch versions (1.0.3 → 1.0.4), just update `_config.yml`:

```yaml
preboot:
  version: "1.0.4"
  versions:
    - name: "latest"
      label: "Latest (1.0.4)"
      version: "1.0.4"
      default: true
```

### Removing Old Versions

1. Remove from `_config.yml` versions array
2. Delete `_docs/vX.X/` directory
3. Delete `_data/navigation-vX.X.yml` file
4. Restart Jekyll

## URL Structure

| Version | URL Pattern | Example |
|---------|-------------|---------|
| Latest | `/docs/path/` | `/docs/getting-started/quick-start/` |
| v1.0 | `/docs/v1.0/path/` | `/docs/v1.0/getting-started/quick-start/` |
| v1.1 | `/docs/v1.1/path/` | `/docs/v1.1/getting-started/quick-start/` |
| Beta | `/docs/beta/path/` | `/docs/beta/getting-started/quick-start/` |

## Version Switching Logic

The version switcher in `_layouts/documentation.html`:

- **Latest**: Uses current structure (no version prefix)
- **Specific versions**: Adds version prefix to URL
- **JavaScript**: Handles URL transformation when switching versions

## Testing

After adding a version:

1. Restart Jekyll server
2. Check version dropdown shows new option
3. Test switching between versions
4. Verify all links work in each version
5. Check that search works in each version

## Troubleshooting

### Version Not Showing in Dropdown
- Check `_config.yml` syntax
- Restart Jekyll server
- Clear browser cache

### 404 Errors in Versioned Content
- Verify directory structure
- Check permalink formatting in front matter
- Ensure navigation URLs match actual file paths

### Version Switching Not Working
- Check JavaScript in `documentation.html`
- Verify URL patterns match expected structure
- Test with browser developer tools

## Best Practices

1. **Always test locally** before deploying version changes
2. **Keep version numbers consistent** across config, navigation, and content
3. **Archive before making breaking changes** to preserve working documentation
4. **Document breaking changes** in version release notes
5. **Use semantic versioning** for clear version communication
6. **Regular cleanup** of very old versions to reduce maintenance overhead

## Quick Commands

```bash
# Start documentation server
./local-testing.sh

# Copy current docs to new version
rsync -av --exclude 'v*' _docs/ _docs/v1.0/

# Update all permalinks in versioned docs
find _docs/v1.0 -name "*.md" -exec sed -i '' 's|/docs/|/docs/v1.0/|g' {} \;

# Add version field to all versioned docs
find _docs/v1.0 -name "*.md" -exec sed -i '' '2a\
version: v1.0' {} \;
```

---

This versioning system provides flexibility while maintaining clean URLs for the latest documentation. The infrastructure scales well as new versions are added.