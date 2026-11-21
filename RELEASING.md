# Releasing

Two packages are released together, with the same version:

| Package | Registry | Version format |
|---|---|---|
| `juizi.dashboard` (backend) | [PyPI](https://pypi.org/project/juizi.dashboard/) | `1.0.0b2` in `src/juizi/dashboard/__init__.py` |
| `volto-juizi-dashboard` (frontend) | [npm](https://www.npmjs.com/package/volto-juizi-dashboard) | `1.0.0-beta.2` in `frontend/package.json` |

Releases are published from **tkimnguyen/juizi.dashboard**, a fork of
juizi-com/juizi.dashboard, by `.github/workflows/release.yml`. No tokens
are stored anywhere: PyPI and npm both trust that workflow directly
(trusted publishing).

## Making a release

1. Bring the fork up to date with juizi-com, if anything was merged there:

       git fetch upstream && git merge upstream/main

   If the two version lines conflict, keep the fork's.
2. On a branch, set the new version in both places, e.g. `1.0.0b3` and
   `1.0.0-beta.3`, and merge it.
3. Publish a GitHub Release on the fork, with tag `v` + the Python
   version. Tick "Set as a pre-release" for a beta:

       gh release create v1.0.0b3 --repo tkimnguyen/juizi.dashboard \
         --target main --prerelease --generate-notes --title "1.0.0b3"

4. In the release's run under Actions, **Review deployments**: approve
   `pypi` and `npm`.
5. Check both registries. npm can take a few minutes to show a new
   version after the workflow says `+ volto-juizi-dashboard@…`.
6. Update the README's **Try it** sites: they use Plone in a Box's
   frontend image `ghcr.io/collective/plone-in-a-box-frontend:19-juizi-dashboard`,
   which is built from pinned versions. In
   [tkimnguyen/plone-in-a-box](https://github.com/tkimnguyen/plone-in-a-box),
   open a PR that sets the new versions in the `juizi-dashboard` entry of
   `.github/workflows/frontend-images.yml`:

       packages: "volto-juizi-dashboard@1.0.0-beta.3"
       backend_addons: "juizi.dashboard==1.0.0b3"

   Once it's merged, the workflow rebuilds the image under the same name,
   and new Try it servers get the new release. Sync it to
   collective/plone-in-a-box as that repo's RELEASING.md describes, since
   the published image is built there. Nothing else changes: the Linode
   StackScript (2318969), the AWS link and the startup lines all name the
   image, not a version.

### What the workflow does

- **check:** the tag must match both versions (npm's `1.0.0-beta.2` reads
  as `1.0.0b2`), and `.dev` versions are refused. Nothing is published if
  this fails.
- **pypi:** `uv build`, `twine check`, `uv publish`.
- **npm:** `npm publish --provenance`, under the `beta` tag for
  pre-releases and `latest` otherwise. Provenance links each npm version
  to the workflow run that built it.

Both publishing jobs wait for approval, and run in parallel once approved.

## One-time setup (done)

Recorded so it can be checked or recreated.

- **PyPI**, Publishing settings of `juizi.dashboard`: trusted publisher
  GitHub, owner `tkimnguyen`, repository `juizi.dashboard`, workflow
  `release.yml`, environment `pypi`.
- **npm**, Settings of `volto-juizi-dashboard`:
  - Trusted Publisher: GitHub Actions, `tkimnguyen` / `juizi.dashboard` /
    `release.yml` / environment `npm`, with **npm publish** allowed.
    New connections only allow `npm stage publish`, and then the
    workflow's `npm publish` fails with a misleading `E404 Not Found`.
    A new connection also expires unless it publishes within 2 days.
  - Publishing access: require two-factor authentication and disallow
    tokens. Publishing by hand needs the maintainer's security key; the
    workflow doesn't use a token, so it isn't affected.
- **GitHub**, the fork's Settings → Environments: `pypi` and `npm` each
  require the maintainer's approval.
- `frontend/package.json`'s `repository.url` names the fork. npm refuses a
  provenance-signed publish whose repository doesn't match the one the
  workflow runs in.

## Publishing from somewhere else

- **From juizi-com instead:** add `release.yml` there, add trusted
  publishers for owner `juizi-com` on PyPI and npm (with npm publish
  allowed), create the `pypi` and `npm` environments, and change
  `repository.url` back to juizi-com.
- **By hand**, e.g. if GitHub Actions is down. npm needs Node 22.14 or
  later for a browser login with a security key:

      uv build && uv publish                          # asks for a PyPI token
      cd frontend && npm login && npm publish --tag beta

## History

- **1.0.0b1** (2026-11-21): PyPI published by the workflow; npm published
  by hand to create the package, since npm only lets you set up trusted
  publishing for a package that already exists.
- **1.0.0b2** (2026-11-21): the first release where the workflow
  published both. npm's `latest` tag was then moved by hand from
  `1.0.0-beta.1` to `1.0.0-beta.2`.
