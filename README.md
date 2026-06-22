# Docker CE Packaging

This repo contains the open source scripts for packaging the
[Docker Engine](https://docs.docker.com/engine/), the Docker CLI, CLI plugins,
and rootless-extras packages.

The repository contains Dockerfiles to build packages for various distributions,
which can be found in the "rpm" and "deb" subdirectories, as well as scripts to
build static binaries.

Docker uses these recipes to build and release packages that are available on the
https://download.docker.com package repositories. We welcome contributions to
this repository, including the addition of new distros or distro-versions. Note,
however, that Docker makes a subselection of distros and architectures for release,
and not all distros available in this repository may be released to download.docker.com,
but you can use these scripts to build your own packages.

## Build static Docker 28.5.1 for loong64

Use the `Build Static Docker` GitHub Actions workflow when the local machine does not have Docker installed. The workflow builds only the static `docker-28.5.1.tgz` tarball and does not build deb or rpm packages.

Default component versions:

- Docker Engine / CLI: `v28.5.1`
- Go: `1.24.8`
- containerd: `v1.7.28`
- runc: `v1.3.0`
- tini / docker-init: `v0.19.0`

The static build validates every binary with `readelf` and `ldd` during the Dockerfile build. A valid artifact must not contain an ELF interpreter and must not depend on dynamic system libraries.

Manual workflow inputs:

- `docker_version`: keep the default `v28.5.1` for the offline package.
- `upload_release`: use `false` to download the artifact from the workflow run, or `true` to upload it to a GitHub release with the same tag.

The artifact layout is compatible with the offline installer:

```text
docker/
  containerd
  containerd-shim-runc-v2
  ctr
  docker
  docker-init
  docker-proxy
  dockerd
  runc
```

After downloading the artifact, place `docker-28.5.1.tgz` under the offline package architecture directory, for example `loong64/docker-28.5.1.tgz`.
