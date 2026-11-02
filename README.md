# Selkies Base Image with Multi-user Active Directory Support

A fork of [linuxserver/docker-baseimage-selkies](https://github.com/linuxserver/docker-baseimage-selkies) (Debian Trixie) that adds an opt-in **multi-user mode**: every Active Directory user who signs in through the browser gets their own concurrent, isolated desktop running as their own Linux account, all in one container.

The image provides a full featured web native Linux desktop for any Linux application or desktop environment. It bundles [Selkies](https://github.com/selkies-project/selkies) (the streaming server and web client), [pixelflux](https://github.com/linuxserver/pixelflux/) (video capture, encoding and the Wayland compositor), [pcmflux](https://github.com/linuxserver/pcmflux) (Opus audio), [NGINX](https://www.nginx.com/), [PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/), [Docker](https://www.docker.com/) and [labwc](https://labwc.github.io/). Without `MULTI_USER` it behaves exactly like the upstream image, so the [LinuxServer.io Selkies documentation](https://docs.linuxserver.io/selkies/) applies to everything except the multi-user additions described below.

> [!IMPORTANT]
> This fork is not affiliated with or supported by LinuxServer.io. Please do not take issues with this image to their GitHub, Discord or forum; open them on [this repository](https://github.com/evan-kinney/docker-baseimage-selkies/issues) instead.

- Images are published to Docker Hub as [`evandashkinney/baseimage-selkies`](https://hub.docker.com/r/evandashkinney/baseimage-selkies).
- As upstream, there is no `latest` tag. Breaking changes can land between builds, so pin a dated tag for anything you depend on.
- The base image ships with passwordless sudo for the single `abc` user and only persists `/config`; anything else is lost when the image is updated.

# Tags

All tags are built for x86_64 and aarch64 from the `master` branch.

| Tag | Description |
| :----: | --- |
| `debiantrixie` | Multi-arch, the latest build. |
| `debiantrixie-<YYYYMMDD>-<commit>` | Multi-arch, a specific build, for example `debiantrixie-20261102-a1b2c3d`. |
| `amd64-debiantrixie`, `arm64v8-debiantrixie` | Single architecture builds, plus the matching dated `amd64-...`/`arm64v8-...` tags. |

The image is rebuilt on every push to `master` and weekly, to pick up Debian security updates and new upstream component releases.

# Quick start

```
docker run --rm -it \
  --shm-size=1gb \
  -p 3001:3001 \
  -e PIXELFLUX_WAYLAND=true \
  evandashkinney/baseimage-selkies:debiantrixie bash
```

Open https://localhost:3001 and accept the self signed certificate. The upstream [Quickstart](https://docs.linuxserver.io/selkies/user-guide/quickstart/) explains the flags, and [Installation](https://docs.linuxserver.io/selkies/user-guide/installation/) covers compose, volumes, PUID/PGID and the rest of the conventions.

# Multi-user with Active Directory

Set `MULTI_USER=true` to give every Active Directory user who signs in their own concurrent desktop (Xvfb, Openbox, PulseAudio and Selkies), running as their own Linux account. The browser login is checked against AD through SSSD with an LDAP bind; no domain join is needed. Each session starts on the user's first login and keeps running until the container stops. Home folders persist in `/config/home/<user>`.

Sessions open the Selkies desktop shell by default. Set `SELKIES_DESKTOP=false` for a bare Openbox session with a terminal instead. Defaults are only copied into a home folder when they are missing, so a user who already signed in keeps their existing `~/.config/openbox/autostart`.

```
docker run -d \
  --name selkies \
  --shm-size=1gb \
  -p 3001:3001 \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Etc/UTC \
  -e MULTI_USER=true \
  -e LDAP_URI=ldaps://dc1.corp.example.com \
  -e LDAP_BASE_DN=DC=corp,DC=example,DC=com \
  -e LDAP_BIND_DN=CN=selkies,OU=Service,DC=corp,DC=example,DC=com \
  -e FILE__LDAP_BIND_PASSWORD=/run/secrets/ldap_bind_password \
  -e "LDAP_ALLOWED_GROUP=CN=Selkies Users,OU=Groups,DC=corp,DC=example,DC=com" \
  -e LDAP_TLS_CACERT=/config/ad-ca.pem \
  -e MULTI_USER_ADMIN_GROUP=selkies-admins \
  -v /path/to/config:/config \
  -v /path/to/ldap_bind_password:/run/secrets/ldap_bind_password:ro \
  evandashkinney/baseimage-selkies:debiantrixie
```

| Variable | Description |
| :----: | --- |
| `MULTI_USER` | `true` enables multi-user mode. The single `abc` desktop is not started. |
| `LDAP_URI` | Domain controller(s), comma separated, for example `ldaps://dc1.corp.example.com`. `ldap://` URIs use StartTLS. |
| `LDAP_BASE_DN` | Search base, for example `DC=corp,DC=example,DC=com`. |
| `LDAP_BIND_DN` | Service account used for lookups, for example `CN=selkies,OU=Service,DC=corp,DC=example,DC=com`. |
| `LDAP_BIND_PASSWORD` | Service account password. `FILE__LDAP_BIND_PASSWORD` is supported. |
| `LDAP_ALLOWED_GROUP` | Optional group DN. Only its direct members may sign in. |
| `LDAP_TLS_CACERT` | Optional path to the PEM of the CA that signed the domain controller certificates, for example `/config/ad-ca.pem`. |
| `MULTI_USER_ADMIN_GROUP` | Optional AD group name whose members get `sudo` with their AD password. Nobody else gets `sudo` or Docker. |
| `MULTI_USER_MAX_SESSIONS` | Concurrent session limit, default `8`. |

### How it works

1. nginx asks the browser for a login and checks it against AD through PAM and SSSD. Repeat logins are cached for five minutes.
2. For an authenticated user, nginx asks a small root broker which session socket to use. On the user's first login the broker starts a supervised session for them, which runs `selkies.session` to bring up their own Xvfb, PulseAudio, desktop and Selkies.
3. Each Selkies listens on a Unix socket in `/run/selkies/<user>/`, and nginx proxies that user's traffic to it.

### Isolation

- Sessions run as the user's own account with a private home folder (`700`) and runtime folder.
- Each Xvfb only admits its own session's cookie, so users cannot open each other's displays.
- A socket folder is sealed (made root owned) once its socket is verified, so a user cannot swap their socket for a link to someone else's. Only nginx can reach the sockets and the broker.
- The LDAP bind password is only kept in the root-only `sssd.conf`. It is removed from the environment that later services and user sessions inherit.
- Local accounts such as `root` and `abc` can never sign in through the web login.

### Limits

- X11 only. `PIXELFLUX_WAYLAND`, `PELORUS` and `DEV_MODE` cannot be combined with `MULTI_USER`.
- Gamepads, webcam and printing are disabled, because those devices are shared by the whole container.
- `RESTART_APP` does not apply to user sessions.
- Sessions are not reaped when idle; `MULTI_USER_MAX_SESSIONS` caps how many run at once.
- Browser basic auth has no logout. Close the browser to switch users.

# Configuration

Every facet of the container is configured with environment variables. The baseimage variables (ports, basic auth, subfolder, Wayland mode, GPU nodes, dashboard selection, and so on) and the complete list of Selkies application settings are documented in the upstream [Configuration Reference](https://docs.linuxserver.io/selkies/user-guide/configuration/). Other topics:

* [GPU Acceleration](https://docs.linuxserver.io/selkies/user-guide/gpu/) - Intel, AMD and Nvidia passthrough, zero copy encoding, multi GPU and the Nvidia host requirements.
* [Security and Hardening](https://docs.linuxserver.io/selkies/user-guide/security/) - authentication, the hardening variables for kiosk style deployments, locking client settings and the token control plane.
* [Reverse Proxy](https://docs.linuxserver.io/selkies/user-guide/reverse-proxy/) - SWAG, Nginx, Traefik and subfolder setups.
* [Using the Web Client](https://docs.linuxserver.io/selkies/user-guide/web-client/) - the sidebar, clipboard, file transfer, gamepads, webcam and session sharing.
* [WebRTC Transport](https://docs.linuxserver.io/selkies/user-guide/webrtc/) - the opt in UDP transport and the STUN/TURN setup it needs.
* [Pelorus](https://docs.linuxserver.io/selkies/components/pelorus/) and [Pixelflux](https://docs.linuxserver.io/selkies/components/pixelflux/) - the agentic web interface, the Computer Use API and session recording.
* [Troubleshooting](https://docs.linuxserver.io/selkies/user-guide/troubleshooting/) - the checklist to run before opening an issue.

**The single-user basic auth is included as a convenience. In general it should be used to keep the kids out, not the internet. If you are looking for a robust secure application gateway, put a reverse proxy such as [SWAG](https://github.com/linuxserver/docker-swag) in front of the container.**

# Installing applications

* The image includes [proot-apps](https://github.com/linuxserver/proot-apps), which install portable applications into the user's `$HOME` so they survive image updates. In multi-user mode every user gets their own copy. See [Installing Applications](https://docs.linuxserver.io/selkies/user-guide/installing-apps/).
* A built in Steam installer is included; run `steam` inside the container without any permissions. See the [Steam section](https://docs.linuxserver.io/selkies/user-guide/installing-apps/#steam-built-in-reinstalls-itself) for the limits and how to disable it.

# Building images on top

Included in the image is [labwc](https://labwc.github.io/) (or [Openbox](http://openbox.org/) in X11 mode) and the init logic needed to launch a single application. The bare minimum needed to create an application container is a Dockerfile:

```
FROM evandashkinney/baseimage-selkies:debiantrixie
RUN apt-get update && apt-get install -y --no-install-recommends firefox-esr
COPY /root /
```

and a `root/defaults/autostart_wayland` file containing the command to run, in this case `firefox-esr`. Build it, run it with the quick start command above and Firefox is streaming in the browser.

Multi-user sessions start the desktop through `/defaults/startwm.sh`, so an image that replaces it with a full desktop environment (as [Webtop](https://github.com/linuxserver/docker-webtop) does) gets that desktop for every user.

Right click menus, full desktop environments via `startwm.sh`, hardening and kiosk builds are covered in the upstream [Building Custom Images](https://docs.linuxserver.io/selkies/developer-guide/building-images/) guide, with [Baseimage Internals](https://docs.linuxserver.io/selkies/developer-guide/baseimage-internals/) describing the init chain, services and nginx layout.

# Docker in Docker (DinD)

The image includes an installation of Docker. Mount `-v /var/run/docker.sock:/var/run/docker.sock` to manage the host Docker installation from inside the session, or pass `--privileged` to start an isolated in container Docker daemon (set `-e START_DOCKER=false` to keep privilege without the daemon). In multi-user mode only the `abc` account is in the `docker` group, so AD users cannot reach either daemon. Details are in the upstream [Docker in Docker](https://docs.linuxserver.io/selkies/user-guide/installation/#docker-in-docker) docs.

# Building this image

Images are built and published by GitHub Actions ([`.github/workflows/build.yml`](.github/workflows/build.yml)):

- Pull requests build both architectures and run the smoke tests, but never publish.
- Pushes to `master`, the weekly schedule and manual runs also publish the arch tags to Docker Hub and then combine them into the multi-arch tags.
- amd64 builds from `Dockerfile` on an x64 runner, and arm64 from `Dockerfile.aarch64` on an arm runner. Each image is smoke tested in single-user and multi-user mode before it is pushed.
- Publishing needs two repository secrets: `DOCKERHUB_USERNAME`, and `DOCKERHUB_TOKEN` (a Docker Hub access token with read and write access).

The multi-user broker, session launcher, s6 services, nginx template and PAM config are not in this repository. They come from the `multiuser` layer of [evan-kinney/selkies-layers](https://github.com/evan-kinney/selkies-layers) (`ghcr.io/evan-kinney/selkies-layers:<arch>-debiantrixie-multiuser`), which the Dockerfiles copy in. This repository keeps the packages they need and the `MULTI_USER` guards in its own services.

To build locally:

```
git clone https://github.com/evan-kinney/docker-baseimage-selkies.git
cd docker-baseimage-selkies
docker build --no-cache --pull -t evandashkinney/baseimage-selkies:debiantrixie .
```

Use `-f Dockerfile.aarch64` for arm64. The ARM variant can be built on x86_64 after registering QEMU with `docker run --rm --privileged lscr.io/linuxserver/qemu-static --reset`.

Two gotchas:

- On Windows with `core.autocrlf=true`, the checkout has CRLF line endings, which break every script in the image. Build from a clone made with `git -c core.autocrlf=false clone ...`.
- Service `run` files and scripts in `root/usr/local/bin` must be committed executable (`git add --chmod=+x <file>`). CI fails the build if they are not.

# Development

This container and any downstream images can also be used as a rapid development environment for the Selkies Project. Clone the upstream repo and run the container as shown:

```
git clone https://github.com/selkies-project/selkies.git
cd selkies
docker run --rm -it \
  --shm-size=1gb \
  -e DEV_MODE=selkies-dashboard \
  -e PUID=1000 \
  -e PGID=1000 \
  -v $(pwd):/config/src \
  -p 3001:3001 evandashkinney/baseimage-selkies:debiantrixie bash
```

The application is restarted on code changes to the src directory you mounted, with feedback for debugging. The web side runs through Vite with hot module reload and is served at the root path of the container. `DEV_MODE` accepts the name of any dashboard directory under `addons/` in the Selkies repo, `core` to watch and rebuild `selkies-web-core` (the streaming engine itself), or `pixelflux` for the [pixelflux](https://github.com/linuxserver/pixelflux) repo. More on the workflow is in the upstream [Development Environment](https://docs.linuxserver.io/selkies/developer-guide/development/) docs.

# Credits

Built on the work of [LinuxServer.io](https://www.linuxserver.io/) and the [Selkies Project](https://github.com/selkies-project). Licensed under GPL-3.0, as upstream (see [LICENSE](LICENSE)).
