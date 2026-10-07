# panvk-faux123-driver-ace

Mesa **PanVK**, the open source Vulkan driver for Arm Mali GPUs, built for
Android and packaged for [AdrenoTools](https://github.com/bylaws/libadrenotools)
style driver loading on the **GameForce Ace** (Mali-G610). The driver itself
needs no root.

The driver talks to the `mali_kbase` kernel driver that already ships on the
device, so there is no kernel to flash, and the system driver stays in place.
The stock driver on this device offers Vulkan 1.3. This build reports Vulkan 1.4
and adds features the stock driver lacks, such as geometry and tessellation
shaders, transform feedback, and BC texture formats. The version number is what
the driver reports. It is not a claim that every Vulkan 1.4 feature is there:
see [What works and what does not](#what-works-and-what-does-not).

I apply fixes that are not in the base I build from, and I verify every one of
them on my own hardware before I release it. Each release lists what it changes.

**This is an early, experimental driver. It is not conformant, it is somewhat
slower than the stock driver, and no game has been run on this exact build.**

**This is a proof of concept, released as is, with no support.** See
[No support](#no-support).

**Everything here is verified on one device: my GameForce Ace, Mali-G610 MC4.**
It is built for that device's kernel driver and I can make no promises anywhere
else. See [Other devices](#other-devices-not-tested-no-guarantees).

Grab the latest `.adpkg.zip` from [Releases](../../releases) and import it in an
app whose driver picker accepts a PanVK package on Mali.

---

## Where the driver comes from

Nothing here is a repackaged binary from somewhere else. It is built from source.

| | |
|---|---|
| Source | [`gitlab.freedesktop.org/mesa/mesa`](https://gitlab.freedesktop.org/mesa/mesa), 26.3.0-devel at commit `5a07217f` |
| `mali_kbase` support | the patch series from [`zenithblue-oss/panvk-kbase-android`](https://github.com/zenithblue-oss/panvk-kbase-android) at `b70eddd` |
| My patches | summarized under [What I added](#what-i-added) |
| Toolchain | Android NDK 27.3, meson cross build, `aarch64`, API level 33 |
| Packaging | `.adpkg.zip`: `libvulkan_panfrost.so` plus `meta.json` |

**Every release names the exact build**, in its release notes and in the
`meta.json` inside the package.

The build container, the patches, the test harnesses and a recreate guide live in
a private companion repository.

### Why the stock kernel driver and not Panthor

Upstream PanVK expects the open source Panthor kernel driver for this GPU. The
Ace ships Arm's `mali_kbase` kernel driver on a 5.10 kernel instead. Running
PanVK on top of `mali_kbase` means there is no kernel to flash to use it.

The patch series I build from was validated by its authors on a newer GPU, the
Mali-G615, and marked the Mali-G610 path as untested. The fixes below are what
the Ace needed.

### What I added

- **Hard freezes.** Earlier builds could freeze the whole device, which needed a
  reset. The kernel driver was powering the GPU off and on between frames. The
  driver now keeps the GPU awake while an app is rendering.
- The Android swapchain did not work: the driver could not read this device's
  buffer layout.
- The GPU reported a fault after about 47 seconds because of how shared buffers
  were mapped on this chip.
- BC-compressed textures showed horizontal streaks on a game's title screen. The
  Mali-G610 has BC support in hardware and the driver now uses it.
- Speed: several changes to how frames are built and presented. The largest one
  stops the driver from blocking the app while the GPU finishes each frame.

---

## How it was tested

Everything below was measured on **one device**: my GameForce Ace, Android 13,
Rockchip RK3588S, Mali-G610 MC4, kernel 5.10.157, `mali_kbase` g18p0. The device
is rooted. My measuring tools use root. The driver does not.

The test rig is the Khronos [Vulkan-Samples](https://github.com/KhronosGroup/Vulkan-Samples)
app, rebuilt with AdrenoTools linked in so it loads a chosen driver directly. No
Wine, no DXVK, no Box64, no emulation. One variable: the driver `.so`.

- A soak of 23 runs across four scenes, about 50,000 presented frames, with the
  kernel's power settings at their defaults: every run finished, with no freeze
  and no lost device. That includes 8 long runs of the scene that used to freeze
  the device.
- My feature validator: 54 checks pass and none fail. A further 41 are features
  the driver does not offer, and 53 could not be exercised.
- I launched the samples one at a time: 51 run and 1 crashes
  (`swapchain_present_timing`). Every sample the stock driver runs also runs on
  this driver.
- The driver logged no errors in any of those sessions.

Frame times against the stock Mali driver, over the 36 samples both drivers run:
this driver takes 1.14 times as long by geometric mean. On five benchmark
scenes, in milliseconds per frame:

| sample | this driver | stock |
|---|---|---|
| `compute_nbody` | 18.8 | 17.9 |
| `oit_linked_lists` | 20.2 | 20.6 |
| `subpasses` | 19.4 | 17.5 |
| `oit_depth_peeling` | 4.9 | 5.4 |
| `terrain_tessellation` | 30.4 | does not run |

So this is not a speed upgrade today, though it is close on these scenes. What it
offers is features: the stock driver cannot run the tessellation scene at all.

## Other devices: not tested, no guarantees

I own one GameForce Ace. Nothing here has been run on anything else. That
includes other RK3588 and RK3588S devices and other Mali-G610 devices.

Concretely, on any other device:

- The driver may not load at all. Each vendor's `mali_kbase` differs, and so
  does each device's display buffer handling.
- Every fix I ship was written for the hardware I have. On a different device it
  may be unnecessary, and it may not be harmless. That applies most to the
  freeze fix, which depends on how this device's kernel manages GPU power.
- Nothing on this page was measured on that hardware, so none of the numbers
  apply to it.
- I will not look into problems on hardware I do not own.

Use it if you want to, but that is the honest state of it, and you are on your
own with it.

I build one driver per device I own, because that is the only way I can test
it: [Retroid Pocket 4 Pro](https://github.com/faux123/panvk-faux123-driver-rp4pro)
and [RG556](https://github.com/faux123/panvk-faux123-driver-rg556).

## Installing

**Read this first: you need an app that can load it.** Most driver pickers
were written for Adreno and refuse or ignore a Mali package.
[GameNative-Mali](https://github.com/faux123/GameNative-Mali/releases) can import
it from version 1.2.1-mali.11: open **Driver Manager**, import the zip, then
select it for a game under **Graphics**. That import screen is new and has had
little use.

1. Download `panvk_faux123_ace_<version>.adpkg.zip` from
   [Releases](../../releases). Do not unzip it.
2. Import it in your app's driver manager, the same way you would import a Turnip
   package on an Adreno device.
3. Select it, then restart the container or game.
4. In a Windows game container, set BC texture emulation to none. This driver
   has BC support in hardware, and the emulation path is what produced streaks.

The driver has to be loaded by an app that opts in. Android does not let you
replace the system Vulkan driver for arbitrary apps without root, so a normal
Play Store game cannot use this.

To go back, select the system driver again. Nothing on the device is replaced.

---

## What works and what does not

**If the device freezes:** hold the power button to reset it, go back to the
system driver. I saw several freezes on earlier builds and
none on this one, but I cannot rule it out on a device I have not tested.

**Games:** an earlier build rendered the title screen of one Windows game at 60
frames a second through GameNative-Mali with DXVK 1.10.3. No game has been run on
this build, and no level has been played on any build.

**DXVK 2.x:** on earlier builds it stopped at a feature (`robustBufferAccess2`)
that this driver does not provide on the Mali-G610. I have not retried it on
this build.

**Somewhat slower than stock**, as the table above shows.

**A known risk in how frames are presented.** To stop blocking the app, this
build lets the GPU wait for the previous frame. The authors of the patch series I
build from turned that mechanism off after it lost the device in rare cases in
their conformance runs on a Mali-G615. My soak did not show it, but a soak cannot
rule out something that rare. If an app stops rendering with this driver, go back to
v0.17.0. It does not use that mechanism, and it is slower: 1.64 times stock.

---

## No support

This is a proof of concept. I built this driver for my own device, and I am
releasing it so the community can see what is possible and build on it.

- There is no support from me. I do not answer bug reports or questions.
- I do not take requests for devices I do not own. If I do not have the device,
  I will not develop for it.
- I do not do remote debugging or beta testing.
- I may update this driver for myself from time to time and post it here. There
  is no schedule and no promise.

If you want to help, support the original developers listed under
[Credits](#credits).

---

## Credits

- **Mesa and the Panfrost team** wrote PanVK. This repository is a build of
  their work with patches on top. All the hard parts are theirs.
- **[zenithblue-oss](https://github.com/zenithblue-oss/panvk-kbase-android)**
  for the patch series that runs PanVK on `mali_kbase` on Android, which this
  build sits on.
- **[bylaws](https://github.com/bylaws/libadrenotools)** for AdrenoTools, which
  is the only reason a custom Vulkan driver can be loaded without root.
- **Khronos** for Vulkan-Samples, which turned out to be a far better driver
  test bench than any game.

## License

PanVK is [MIT licensed](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/docs/license.rst),
and so are the patches in this build. See [LICENSE](LICENSE).

The driver is compiled against Arm's `mali_kbase` interface headers, which are
GPL-2.0 with the Linux syscall note. That note is what allows a program to use
a kernel interface without taking on the kernel's license.

This project is not affiliated with or endorsed by Arm, Rockchip, GameForce,
Google, the Khronos Group, or the Mesa project.
