# container-images

Personal container images, built and published to the GitHub Container Registry
as `ghcr.io/osnabrugge/<app>`.

These are images that exist because **upstream doesn't publish one**. Most are
MCP servers that ship as source, an npm package or a Python package only, so
something has to build them before [ToolHive](https://github.com/stacklok/toolhive)
can run them in the cluster.

## Which repo does what

This is one of three related repos. They are deliberately separate:

| repo | purpose |
| ---- | ------- |
| [`home-ops`](https://github.com/osnabrugge/home-ops) | GitOps for the Talos cluster — manifests only, **no image builds** |
| **`container-images`** (this repo) | Personal images with no value to anyone else |
| [`containers`](https://github.com/home-operations/containers) (fork) | Staging area for changes intended as **upstream PRs** to home-operations |

The rule of thumb: if someone else would plausibly want it, it belongs in the
`containers` fork as a PR to upstream. If it only makes sense for this estate,
it belongs here.

## Layout

```
apps/
  <app>/
    Dockerfile      # the build
    metadata.json   # published version — bump this to cut a release
```

## Images

| app | upstream | why we build it |
| --- | -------- | --------------- |
| `brocade-icx-mcp` | [vespo92/BrocadeICXMCP](https://github.com/vespo92/BrocadeICXMCP) | TypeScript source only, no image |
| `onshape-mcp` | [hedless/onshape-mcp](https://github.com/hedless/onshape-mcp) | Python package only, no image |
| `opnsense-mcp` | [@richard-stovall/opnsense-mcp-server](https://www.npmjs.com/package/@richard-stovall/opnsense-mcp-server) | npm package only, no image |
| `printer-mcp` | [mcp-3d-printer-server](https://www.npmjs.com/package/mcp-3d-printer-server) | npm package only, no image |
| `spoolman-mcp` | [Disane87/spoolman-mcp](https://github.com/Disane87/spoolman-mcp) | TypeScript source only, no image |
| `talos-mcp` | [eleboucher/talos-mcp](https://github.com/eleboucher/talos-mcp) | Canonical source is a private Forgejo; upstream's prebuilt image is on a personal registry |

## Other build artifacts

| artifact | upstream | why we build it |
| -------- | -------- | --------------- |
| PrintStash browser extension | [xiao-villamor/PrintStash](https://github.com/xiao-villamor/PrintStash/tree/main/browser-extension) | Upstream only publishes it as a 30-day CI artifact |

`.github/workflows/printstash-extension.yaml` reads the PrintStash tag that
`home-ops` deploys, builds the extension at that same tag and publishes the
Chrome, Edge and Firefox zips as the `printstash-extension-<version>` release.
It runs every six hours and skips versions that are already published.

## How builds work

`.github/workflows/build.yaml` detects which `apps/<app>/` directories changed in
a push and builds only those. `workflow_dispatch` can target a single app, or
rebuild everything.

Each image is tagged twice:

- `ghcr.io/osnabrugge/<app>:<version>` — from `metadata.json`
- `ghcr.io/osnabrugge/<app>:latest`

**Bumping `metadata.json` is what cuts a release.** Renovate updates the pinned
upstream ref inside the Dockerfile; deciding that a given build is worth
publishing under a new version number stays a human decision.

## Adding an app

1. `mkdir -p apps/<app>` with a `Dockerfile` and a `metadata.json`
2. Pin the upstream ref with a build `ARG` and annotate it with a
   `# renovate:` comment so Renovate can track it
3. Add a row to the table above
4. Push — CI picks it up from the changed path

## Conventions

- **Pin upstream by commit SHA or exact version**, never a moving branch or tag.
  A `# renovate:` comment above each `ARG` keeps it updated deliberately rather
  than silently.
- **Run as non-root.** Prefer distroless or `-slim` runtime stages.
- **Multi-stage**: build toolchains must not reach the runtime image.
- **Comment the *why*.** Several of these Dockerfiles encode a hard-won
  workaround (see `onshape-mcp`'s MCP SDK pin). Those comments are the reason
  the image works; don't strip them.

## Related

- Cluster manifests that consume these images: [`home-ops`](https://github.com/osnabrugge/home-ops)
  under `kubernetes/apps/toolhive/mcp-servers/`
