# Customizing Listmonk Templates with Docker and Coolify

## The Problem

Listmonk embeds static templates (email templates, public pages) into its binary at build time. To customize these templates, you traditionally need to:

1. Rebuild the entire binary with modified templates, OR
2. Use the `--static-dir` command-line flag to point to a custom directory

Neither option works well with Docker deployments like Coolify, where:
- You can't easily modify the container's startup command
- You don't want to maintain a full fork with custom builds
- You want to pull the official docker-compose without modifications

## The Solution

We added environment variable support for the `--static-dir` flag, allowing:

```bash
LISTMONK_static_dir=/path/to/custom/static
```

This works alongside the existing volume mount in docker-compose (`./uploads:/listmonk/uploads`), so you can place custom templates in `uploads/static/` and set:

```bash
LISTMONK_static_dir=/listmonk/uploads/static
```

## Technical Changes

### 1. Environment Variable Parsing Fix

Modified `cmd/main.go` to convert underscores to hyphens for top-level CLI flags:

```go
// Before: LISTMONK_static_dir → static_dir (didn't match --static-dir flag)
// After:  LISTMONK_static_dir → static-dir (matches the flag)

if err := ko.Load(env.Provider("LISTMONK_", ".", func(s string) string {
    key := strings.ToLower(strings.TrimPrefix(s, "LISTMONK_"))
    key = strings.Replace(key, "__", ".", -1)
    // Only convert underscore to hyphen for top-level keys (CLI flags)
    // Nested config keys (containing dots) keep underscores (e.g., db.ssl_mode)
    if !strings.Contains(key, ".") {
        key = strings.Replace(key, "_", "-", -1)
    }
    return key
}), nil)
```

**Key insight**: Double underscores (`__`) become dots for nested config (e.g., `LISTMONK_db__host` → `db.host`), while single underscores in top-level keys become hyphens for CLI flags (e.g., `LISTMONK_static_dir` → `static-dir`).

### 2. Custom Static Files Structure

The static directory needs this structure:

```
static/
├── email-templates/
│   ├── subscriber-optin.html
│   ├── subscriber-optin-campaign.html
│   └── ... other email templates
└── public/
    └── templates/
        └── optin.html
        └── ... other public templates
```

### 3. Template Modifications (Example: Show Private List Names)

Original template hid private list names:

```html
{{ range $i, $l := .Lists }}
    {{ if eq .Type "public" }}
        <li>{{ .Name }}</li>
    {{ else }}
        <li>{{ L.Ts "email.optin.privateList" }}</li>
    {{ end }}
{{ end }}
```

Modified to show actual names:

```html
{{ range $i, $l := .Lists }}
    <li>{{ .Name }}{{ if .Description }} - {{ .Description }}{{ end }}</li>
{{ end }}
```

## Deployment Setup

### Fork Structure

- **`master` branch**: Clean code changes only (for PRs to upstream)
- **`develop` branch**: Custom docker-compose, goreleaser config, test files

### Building Custom Docker Image

1. Fork the repo
2. Modify `.goreleaser.yml` to use your Docker Hub username
3. Create a git tag (e.g., `v4.0.0-staticdir`)
4. GitHub Actions builds and pushes to your Docker Hub

### Coolify Deployment

1. Point Coolify to your fork's `develop` branch
2. The docker-compose.yml references your custom image
3. Add environment variable: `LISTMONK_static_dir=/listmonk/uploads/static`
4. Upload custom templates to the uploads volume under `static/`

### Local Testing

Use `docker-compose.test.yml` with Mailpit for email testing:

```bash
docker compose -f docker-compose.test.yml up
```

- Listmonk: http://localhost:9000
- Mailpit (email inbox): http://localhost:8025

## Files Changed

| File | Purpose |
|------|---------|
| `cmd/main.go` | Env var parsing for static-dir/i18n-dir |
| `.goreleaser.yml` | Build config for custom Docker image |
| `docker-compose.yml` | References custom image |
| `docker-compose.test.yml` | Local testing with Mailpit |

## Pull Request

The core env var change was submitted upstream: https://github.com/knadh/listmonk/pull/2807

If merged, future versions of Listmonk will support `LISTMONK_static_dir` natively, eliminating the need for a custom image.