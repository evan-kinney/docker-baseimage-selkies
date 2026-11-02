# Contributing to baseimage-selkies

This is a fork of [linuxserver/docker-baseimage-selkies](https://github.com/linuxserver/docker-baseimage-selkies) with multi-user Active Directory support. Changes that are not specific to this fork are better sent upstream.

## Common files

| File | Use case |
| :----: | --- |
| `Dockerfile` | Dockerfile used to build amd64 images |
| `Dockerfile.aarch64` | Dockerfile used to build 64bit ARM images |
| `root/` | Copied to `/` in the image: s6-overlay services, defaults and helper scripts |
| [selkies-layers](https://github.com/evan-kinney/selkies-layers) `layers/multiuser/` | The multi-user broker, session launcher and services, copied in from the `multiuser` layer image |
| `README.md` | Documentation for the repository and Docker Hub, edited directly |
| `.github/workflows/build.yml` | Builds, tests and publishes the image |

## Guidelines

* Keep `Dockerfile` and `Dockerfile.aarch64` in step. Installed packages are listed one per line in alphabetical order.
* Service scripts start with `#!/usr/bin/with-contenv bash` and `# shellcheck shell=bash` and should be shellcheck clean.
* Service `run` files and scripts in `root/usr/local/bin` must be committed executable (`git add --chmod=+x <file>`). CI fails the build otherwise.
* Multi-user logic stays behind `MULTI_USER=true`. Without it the image must behave exactly like upstream.
* Document user facing changes in `README.md`.

## Testing your changes

```bash
git clone https://github.com/evan-kinney/docker-baseimage-selkies.git
cd docker-baseimage-selkies
docker build \
  --no-cache \
  --pull \
  -t evandashkinney/baseimage-selkies:test .
```

On Windows, clone with `git -c core.autocrlf=false clone ...`; CRLF line endings break every script in the image.

The ARM variant can be built on x86_64 hardware and vice versa using `lscr.io/linuxserver/qemu-static`

```bash
docker run --rm --privileged lscr.io/linuxserver/qemu-static --reset
```

Once registered you can define the dockerfile to use with `-f Dockerfile.aarch64`.

Check that init completes (`[ls.io-init] done.`) both with and without `MULTI_USER=true`, and that the container survives a restart against an existing `/config`. Pull requests are built and smoke tested by CI on both architectures.
