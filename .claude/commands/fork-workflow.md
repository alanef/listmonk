# Listmonk Fork Workflow

This is a customized fork of knadh/listmonk with our own modifications.

## Branch Strategy

- **`master`** - Pure upstream mirror of `knadh/listmonk`. Never commit custom changes here.
- **`develop`** - Our customized fork for deployment. Contains our modifications:
  - Custom docker-compose.yml (uses `alanef/listmonk` image)
  - Custom GitHub Actions workflows (goreleaser builds our image)
  - Any code fixes/features we need

## Common Tasks

### Sync master with upstream

```bash
git fetch upstream master
git checkout master
git reset --hard upstream/master
git push origin master --force
```

### Merge upstream changes into develop

```bash
git checkout develop
git merge master --no-edit
# Resolve any conflicts, keeping our customizations
git push origin develop
```

### Create a release from develop

1. Update `docker-compose.yml` with the new version tag
2. Commit the change
3. Tag and push:

```bash
git tag vX.Y.Z-description
git push origin vX.Y.Z-description
```

This triggers the goreleaser GitHub Action which builds and pushes the Docker image.

**Important:** Always update docker-compose.yml to reference the new tag before or after tagging!

### Contribute a fix back to upstream

To submit a PR to knadh/listmonk without our custom files:

1. Create a branch from master (not develop):
   ```bash
   git checkout master
   git checkout -b fix/description
   ```

2. Cherry-pick only the relevant commits:
   ```bash
   git cherry-pick <commit-hash>
   ```

3. Push and create PR:
   ```bash
   git push -u origin fix/description
   gh pr create --repo knadh/listmonk --title "fix: Description" --body "..."
   ```

4. Switch back to develop:
   ```bash
   git checkout develop
   ```

## Files We Customize (do not submit upstream)

- `docker-compose.yml` - Uses our Docker image
- `docker-compose.test.yml` - Test configuration
- `.github/workflows/release.yml` - Our goreleaser config
- Any other workflow files we've modified

## Remotes

- `origin` - git@github.com:alanef/listmonk.git (our fork)
- `upstream` - https://github.com/knadh/listmonk.git (original repo)