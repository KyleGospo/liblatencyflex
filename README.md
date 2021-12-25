# LatencyFleX (LFX)

Vendor and game agnostic latency reduction middleware. An alternative to NVIDIA Reflex.

## Why LatencyFleX?

There is a phenomenon commonly known among gamers, where input lag would increase when the game
FPS is left uncapped and the game reaches maximum GPU utilization. [[video]](https://www.youtube.com/watch?v=7CKnJ5ujL_Q)

The reason this happens is that GPU work is dispatched asynchronously, and when the GPU reaches
maximum utilization, it fails to catch up to the CPU and creates a queue of work, resulting in
extra latency.

[//]: # (TODO: write about RHI thread queueing)

The traditional solution to this was to cap the framerate at the highest number the machine can sustain:
it creates a CPU-bound scenario, and combined with variable refresh-rate support like FreeSync, it
could consistently achieve the lowest possible input lag. The obvious drawback with this method is
that the limit sacrifices the frame rate even when machine could run at a higher frame rate.

LatencyFleX is an easy-to-integrate solution that achieves the same minimal latency as capping the
frame rate, while dynamically adapting to workload, unleashing the system's full rendering potential.
It takes inspiration from researches in internet congestion control, where a similar problem called 
Bufferbloat has been studied.

## Limitations

- LatencyFleX current does not provide any benefits when VSync is enabled.  
  This is blocked on [presentation timing](https://github.com/KhronosGroup/Vulkan-Docs/pull/1364) support.
- LatencyFleX introduces jitter in frame time as a part of its algorithm, which results in microstutters.  
  Though, most games tend to have a larger frame time fluctuation already, so this is likely unperceivable.

## Known issues

- Minor stutters might happen during gaming session. The algorithm is still being tuned, but open an issue if the stutter
  is severe.
- GPU utilization will be lower when running with LatencyFleX. Overall LatencyFleX should be able to achieve ~95%
  utilization (assuming GPU bound). Open an issue if utilization is significantly low.

## Building from source

The layer (`layer/`) depends on CMake, Meson and the Vulkan SDK.

Build and install with:
```shell
cd layer
meson build
ninja -C build
meson install -C build --skip-subprojects
```

---

The Wine extension (`layer/wine/`) additionally depends on a Wine installation.

Build with:

```shell
cd layer/wine
export LIBRARY_PATH="$PWD/../build/" # Required if the layer has not been installed globally
meson build
ninja -C build
```

See [install instructions](#installation) for the locations to copy the files to.

## Installation

### LatencyFleX Vulkan layer (essential)

For Debian-like distros, copy the following files from [release artifacts](https://github.com/ishitatsuyuki/LatencyFleX/actions) to your root filesystem.

```
/usr/lib/x86_64-linux-gnu/liblatencyflex_layer.so
/usr/share/vulkan/implicit_layer.d/latencyflex.json
```

For Arch-like distros, you need to copy `/usr/lib/x86_64-linux-gnu/liblatencyflex_layer.so -> /usr/lib/liblatencyflex_layer.so`
and additionally update the path specified in `/usr/share/vulkan/implicit_layer.d/latencyflex.json`.

### LatencyFleX Wine extensions (required for Proton Reflex integration)

Copy the following files from [release artifacts](https://github.com/ishitatsuyuki/LatencyFleX/actions) to your Wine installation location.

For Wine 7.x (including Proton-GE-Custom): change `/usr/lib/wine` to wherever Wine/Proton is installed.
For Proton and certain distros, you also need to change `lib` to `lib64`. Copy the following files.

```
/usr/lib/wine/x86_64-unix/latencyflex_wine.dll.so
/usr/lib/wine/x86_64-windows/latencyflex_wine.dll
```

For Wine <= 6.x and current Proton versions: copy the files as follows:

```
/usr/lib/wine/x86_64-unix/latencyflex_wine.dll.so -> lib64/wine/latencyflex_wine.dll.so
/usr/lib/wine/x86_64-windows/latencyflex_wine.dll -> lib64/wine/fakedlls/latencyflex_wine.dll
```

### DXVK-NVAPI with LatencyFleX integration (required for Proton Reflex integration)

Obtain binaries from [GitHub Actions](https://github.com/ishitatsuyuki/dxvk-nvapi/actions?query=branch%3Alfx).

For Proton, copy `nvapi64.dll` into `dist/lib64/wine/nvapi`.

For other Wine installations, see [DXVK-NVAPI documentation](https://github.com/jp7677/dxvk-nvapi#how-to-use).

### MangoHud with metric support (optional)

Obtain binaries from [GitHub Actions](https://github.com/ishitatsuyuki/MangoHud/actions?query=branch%3Acustom-metrics)
and install it to your system.

Put the following line in `MangoHud.conf` to have real-time latency metrics:

```
graphs=custom_Latency
```

## Usage

For now, LatencyFleX can be used through one of the following injection method. Game engine integration is planned.

### Running games with LatencyFleX

**Warning:** Avoid using LatencyFleX with anti-cheat enabled games! It can almost certainly get you banned.

Tested games:

| Game        | Support | Method         |
|-------------|---------|----------------|
| Splitgate   | ✅       | Linux UE4 Hook |
| Ghostrunner | ✅       | Proton NVAPI   |

Game supported but not in list? File a PR to update the table.

#### Proton NVAPI (for games that already have NVIDIA Reflex integration)

[Install](#installation) the Vulkan layer, wine extension and the modified branch of DXVK-NVAPI.

Put the following in `dxvk.conf` (only for DX11 games):

```ini
dxgi.nvapiHack = False
dxgi.customVendorId = 10de # If running on non-NVIDIA GPU
```

Launch with the following environment variables:

```shell
PROTON_ENABLE_NVAPI=1 DXVK_NVAPI_DRIVER_VERSION=49729 DXVK_NVAPI_ALLOW_OTHER_DRIVERS=1 LFX=1 %command%
```

#### UE4 Hook

**Note:** for now, the UE4 hook only supports Linux UE4 builds with PIE disabled.

[Install](#installation) the Vulkan layer.

The first step is to obtain an offset to `FEngineLoop::Tick`. If the game ships with debug symbols, the
offset can be obtained with the command:

```shell
readelf -Ws PortalWars/Binaries/Linux/PortalWars-Linux-Shipping.debug | c++filt | grep FEngineLoop::Tick
```

Find the line corresponding to the actual function (other entries are for types used in the function and unrelated):

```
268: 00000000026698e0  9876 FUNC    LOCAL  HIDDEN    15 FEngineLoop::Tick()
```

Here `26698e0` is the offset we need. We will call it `<OFFSET>` below.

Then, modify the launch command-line as follows.

```shell
LFX=1 LFX_UE4_HOOK=0x<OFFSET> %command%
```