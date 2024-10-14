<!-- DO NOT EDIT THIS FILE MANUALLY -->
<!-- Please read https://github.com/linuxserver/docker-baseimage-kasmvnc/blob/master/.github/CONTRIBUTING.md -->
# KasmVNC Base Images from LinuxServer

The purpose of these images is to provide a full featured web native Linux desktop experience for any Linux application or desktop environment. These images replace our old base images at [Rdesktop Web](https://github.com/linuxserver/docker-baseimage-rdesktop-web) for greatly increased performance, fidelity, and feature set. They ship with passwordless sudo to allow easy package installation, testing, and customization. By default they have no logic to mount out anything but the users home directory, meaning on image updates anything outside of `/config` will be lost.

These images contain the following services: 

* [KasmVNC](https://www.kasmweb.com/kasmvnc) - The core technology for interacting with a containerized desktop from a web browser.
* [Kclient](https://github.com/linuxserver/kclient) - NodeJS Iframe wrapper for KasmVNC providing audio and file access.
* [NGINX](https://www.nginx.com/) - Used to serve the mix of KasmVNC and Kclient with the appropriate headers and provide basic auth.
* [Docker](https://www.docker.com/) - Can be used for interacting with a mounted in Docker socket or if the container is run in privileged mode will start a [DinD](https://www.docker.com/blog/docker-can-now-run-within-docker/) setup.
* [PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/) - Sound subsystem used to capture audio from the active desktop session and send it to the browser via the Kclient helper application.

# Options

**Authentication for these containers is included as a convenience and to keep in sync with the previous xrdp containers they replace. We use bash to substitute in settings user/password and some strings might break that. In general this authentication mechanism should be used to keep the kids out not the internet**

If you are looking for a robust secure application gateway please check out [SWAG](https://github.com/linuxserver/docker-swag). 

All application settings are passed via environment variables:

| Variable | Description |
| :----: | --- |
| CUSTOM_PORT | Internal port the container listens on for http if it needs to be swapped from the default 3000. |
| CUSTOM_HTTPS_PORT | Internal port the container listens on for https if it needs to be swapped from the default 3001. |
| CUSTOM_USER | HTTP Basic auth username, abc is default. |
| PASSWORD | HTTP Basic auth password, abc is default. If unset there will be no auth |
| SUBFOLDER | Subfolder for the application if running a subfolder reverse proxy, need both slashes IE `/subfolder/` |
| TITLE | The page title displayed on the web browser, default "KasmVNC Client". |
| FM_HOME | This is the home directory (landing) for the file manager, default "/config". |
| START_DOCKER | If set to false a container with privilege will not automatically start the DinD Docker setup. |
| DRINODE | If mounting in /dev/dri for [DRI3 GPU Acceleration](https://www.kasmweb.com/kasmvnc/docs/master/gpu_acceleration.html) allows you to specify the device to use |
| DISABLE_DRI | When using privilged mode or mounting in a video card, do not attempt to use it for DRI3 acceleration in KasmVNC |
| DISABLE_IPV6 | If set to true or any value this will disable IPv6 |
| LC_ALL | Set the Language for the container to run as IE `fr_FR.UTF-8` `ar_AE.UTF-8` |
| NO_DECOR | If set the application will run without window borders for use as a PWA. (Decor can be enabled and disabled with Ctrl+Shift+d) |
| NO_FULL | Do not autmatically fullscreen applications when using openbox. |

## Language Support - Internationalization

The environment variable `LC_ALL` can be used to start this image in a different language than English simply pass for example to launch the Desktop session in French `LC_ALL=fr_FR.UTF-8`. Some languages like Chinese, Japanese, or Korean will be missing fonts needed to render properly known as cjk fonts, but others may exist and not be installed. We only ensure fonts for Latin characters are present. Fonts can be installed with a mod on startup.

To install cjk fonts on startup as an example pass the environment variables(Alpine):

```
-e DOCKER_MODS=linuxserver/mods:universal-package-install
-e INSTALL_PACKAGES=font-noto-cjk
-e LC_ALL=zh_CN.UTF-8
```

The web interface has the option for "IME Input Mode" in Settings which will allow non english characters to be used from a non en_US keyboard on the client. Once enabled it will perform the same as a local Linux installation set to your locale.

# Available Distros

All base images are built for x86_64 and aarch64 platforms.

| Distro | Current Tag |
| :----: | --- |
| Alpine | alpine320 |
| Arch | arch |
| Debian | debianbookworm |
| Fedora | fedora39 |
| Fedora | fedora40 |
| Ubuntu | ubuntujammy |
| Ubuntu | ubuntunoble |

# PRoot Apps

All images include [proot-apps](https://github.com/linuxserver/proot-apps) which allow portable applications to be installed to persistent storage in the user's `$HOME` directory. These applications and their settings will persist upgrades of the base container and can be mounted into different flavors of KasmVNC containers. IE if you are running an Alpine based container you will be able to use the same `/config` directory mounted into a Debian based container and retain the same applications and settings as long as they were installed with `proot-apps install`.

A list of linuxserver.io supported applications is located [HERE](https://github.com/linuxserver/proot-apps?tab=readme-ov-file#supported-apps).

# I like to read documentation

## Building images

### Application containers

Included in these base images is a simple [Openbox DE](http://openbox.org/) and the accompanying logic needed to launch a single application. Lets look at the bare minimum needed to create an application container starting with a Dockerfile: 

```
FROM ghcr.io/linuxserver/baseimage-kasmvnc:alpine320
RUN apk add --no-cache firefox
COPY /root /
```

And we can define the application to start using: 

```
mkdir -p root/defaults
echo "firefox" > root/defaults/autostart
```

Resulting in a folder that looks like this: 

```
├── Dockerfile
└── root
  └── defaults
    └── autostart
```

Now build and test:

```
docker build -t firefox .
docker run --rm -it -p 3000:3000 firefox bash
```

On http://localhost:3000 you should be presented with a Firefox web browser interface.

This similar setup can be used to embed any Linux Desktop application in a web accesible container.

**If building images it is important to note that many application will not work inside of Docker without `--security-opt seccomp=unconfined`, they may have launch flags to not use syscalls blocked by Docker like with chromium based applications and `--no-sandbox`. In general do not expect every application will simply work like a native Linux installation without some modifications**

#### In container application launching

Also included in the init logic is the ability to define application launchers. As the user has the ability to close the application or if they want to open multiple instances of it this can be useful. Here is an example of a menu definition file for Firefox:

```
<?xml version="1.0" encoding="utf-8"?>
<openbox_menu xmlns="http://openbox.org/3.4/menu">
<menu id="root-menu" label="MENU">
<item label="xterm" icon="/usr/share/pixmaps/xterm-color_48x48.xpm"><action name="Execute"><command>/usr/bin/xterm</command></action></item>
<item label="FireFox" icon="/usr/share/icons/hicolor/48x48/apps/firefox.png"><action name="Execute"><command>/usr/bin/firefox</command></action></item>
</menu>
</openbox_menu>
```

Simply create this file and add it to your defaults folder as `menu.xml`: 

```
├── Dockerfile
└── root
  └── defaults
    └── autostart
    └── menu.xml
```

This allows users to right click the desktop background to launch the application.


### Full Desktop environments

When building an application container we are leveraging the Openbox DE to handle window management, but it is also possible to completely replace the DE that is launched on container init using the `startwm.sh` script, located again in defaults: 

```
├── Dockerfile
└── root
  └── defaults
    └── startwm.sh
```

If included in the build logic it will be launched in place of Openbox. Examples for this kind of configuration can be found in our [Webtop repository](https://github.com/linuxserver/docker-webtop)

### Kasm Workspaces compatibility

Included in these base images are binary blobs `/kasmbins` and a special init process `/kasminit` to maintain compatibility with [Kasm Workspaces](https://www.kasmweb.com/), If using this base image as reccomended with the `startwm.sh` or `autostart` entrypoints. They will be able to be used on that platform without issue.

## Docker in Docker (DinD)

These base images include an installation of Docker that can be used in two ways. The simple method is simply leveraging the Docker/Docker Compose cli bins to manage the host level Docker installation by mounting in `-v /var/run/docker.sock:/var/run/docker.sock`. 

The base images can also run an isolated in container DinD setup simply by passing `--privileged` to the container when launching. If for any reason the application needs privilege but Docker is not wanted the `-e START_DOCKER=false` can be set at runtime or in the Dockerfile. 
In container Docker (DinD) will most likely use the fuse-overlayfs driver for storage which is not as fast as native overlay2. To increase perormance the `/var/lib/docker/` directory in the container can be mounted out to a Linux host and will use overlay2. Keep in mind Docker runs as root and the contents of this directory will not respect the PUID/PGID environment variables available on all LinuxServer.io containers.

## DRI3 GPU Acceleration

For accelerated apps or games, render devices can be mounted into the container and leveraged by applications using: 

`--device /dev/dri:/dev/dri`

This feature only supports **Open Source** GPU drivers:

| Driver | Description |
| :----: | --- |
| Intel | i965 and i915 drivers for Intel iGPU chipsets |
| AMD | AMDGPU, Radeon, and ATI drivers for AMD dedicated or APU chipsets |
| NVIDIA | nouveau2 drivers only, closed source NVIDIA drivers lack DRI3 support |

The `DRINODE` environment variable can be used to point to a specific GPU.
Up to date information can be found [here](https://www.kasmweb.com/kasmvnc/docs/master/gpu_acceleration.html)

### Display Compositing (desktop effects)

When using this image in tandem with a supported video card, compositing will function albeit with a performance hit when syncing the frames with pixmaps for the applications using it. This can greatly increase app compatibility if the application in question requires compositing, but requires a real GPU to be mounted into the container. By default we disable compositing at a DE level for performance reasons on our downstream images, but it can be enabled by the user and programs using compositing will still function even if the DE has it disabled in its settings. When building desktop images be sure you understand that with it enabled by default only users that have a compatible GPU mounted in will be able to use your image.

## Nvidia GPU Support

**Nvidia is not compatible with Alpine based images**

Nvidia support is available by leveraging Zink for OpenGL support. This can be enabled with the following run flags:

| Variable | Description |
| :----: | --- |
| --gpus all | This can be filtered down but for most setups this will pass the one Nvidia GPU on the system |
| --runtime nvidia | Specify the Nvidia runtime which mounts drivers and tools in from the host |

The compose syntax is slightly different for this as you will need to set nvidia as the default runtime:

```
sudo nvidia-ctk runtime configure --runtime=docker --set-as-default
sudo service docker restart
```

And to assign the GPU in compose:

```
services:
  myimage:
    image: myname/myimage:mytag
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [compute,video,graphics,utility]
```

## Lossless 

These images support all the native KasmVNC encoding methods including a true 24 bit RGB lossless mode using the [Quite OK Image Format](https://qoiformat.org/). This mode will use all the bandwidth you give it so just keep that in mind for remote sessions. This mode also might require special configuration depending on how you are accessing the container. Lossless will only work over http (default port 3000) on localhost, when accessing remotely or even over a local network you need to use https (default port 3001) to support [SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer). This is needed to leverage a fast memory pipeline in the browser during the threaded WebAssembly based decoding. This can be enabled in the sidebar under settings>stream quality>lossless.

If putting this container behind a proxy of some kind some headers will need to be set to again support SharedArrayBuffers here is a default NGINX configuration format: 

```
add_header 'Cross-Origin-Embedder-Policy' 'require-corp';
add_header 'Cross-Origin-Opener-Policy' 'same-origin';
add_header 'Cross-Origin-Resource-Policy' 'same-site';
```

More information [here](https://www.kasmweb.com/docs/latest/how_to/lossless.html)

The following line is only in this repo for loop testing:
- { date: "01.01.50:", desc: "I am the release message for this internal repo." }
