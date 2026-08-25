# Galaxy Import — Parked Work

Status: Not started. Resume when ready to publish to Ansible Galaxy.

## Problem

The CI publish step in `.github/workflows/ci.yml` uses `ansible-galaxy collection build` and `collection publish`. This is wrong — this is a **role**, not a collection. There is no `galaxy.yml`, so `collection build` will fail. Roles are not distributed as tarballs.

## Correct Approach

Roles are published to Galaxy by telling Galaxy to import directly from GitHub:

```bash
ansible-galaxy role import lingfish ansible-role-cloudflare-aop --token "${{ secrets.GALAXY_API_KEY }}"
```

Galaxy clones the repo, scans git tags matching semver format, and registers them as versions.

## CI Workflow Changes

Replace the publish job steps:

```yaml
publish:
  name: Publish to Galaxy
  needs: [lint, molecule]
  runs-on: ubuntu-latest
  if: github.event_name == 'release'
  permissions:
    contents: read
  steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-python@v5
      with:
        python-version: "3.12"

    - name: Install Ansible
      run: pip install ansible-core

    - name: Import role to Galaxy
      run: >
        ansible-galaxy role import lingfish ansible-role-cloudflare-aop
        --token "${{ secrets.GALAXY_API_KEY }}"
```

## Tag Convention

- Semver without `v` prefix: `1.0.0`, `1.1.0`, `2.0.0`
- Galaxy strips `v` if present, but keeping it clean avoids confusion

## Release Workflow (once CI is fixed)

```bash
git tag -a X.Y.Z -m "Description"
git push origin main --tags
gh release create X.Y.Z --title "X.Y.Z" --notes "..."
# CI triggers on release → role import → Galaxy picks up new version
```

## meta/main.yml

Already correct for Galaxy — has `role_name`, `namespace`, `platforms`, `galaxy_tags`, `argument_specs`. No changes needed.

## Consumer Usage

Once published, users install from Galaxy directly:

```bash
ansible-galaxy install lingfish.cloudflare_aop
```

Or pin a version in `requirements.yml`:

```yaml
roles:
  - name: lingfish.cloudflare_aop
    version: "1.0.0"
```

## Notes

- Galaxy API token needs to be created at https://galaxy.ansible.com/ui/token
- The `role import` command is idempotent — re-running it on the same tag is safe
- Deleted git tags cause the corresponding Galaxy version to be removed
