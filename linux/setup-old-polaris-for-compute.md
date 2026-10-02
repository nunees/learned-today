# Setup old Polaris Graphics Card for Compute

> **Note:** This guide is intended for users who want to set up an older Polaris graphics card (such as the RX 400 or RX 500 series) for compute tasks on a Linux system. The steps below will help you configure your system to utilize the GPU for tasks like machine learning, scientific computing, or other GPU-accelerated workloads. Since these cards are older, some features may not be supported, and performance may vary compared to newer GPUs.

## Setup

### 1. Check whether the firmware is actually missing

```
ls -l /lib/firmware/amdgpu/polaris10_sdma.bin
```

If you get:

```
No such file or directory
```

install the firmware package.

### 2. Installing AMD GPU firmware

```
sudo apt update
sudo apt install firmware-amd-graphics
```

Debian 13's `firmware-amd-graphics` package contains the AMD/ATI firmware used by `amdgpu`.

Then verify:

```
ls -lh /lib/firmware/amdgpu/polaris10_sdma.bin
```

You should now see the bin file.

### 3. Rebuild the initramfs

This is important because the GPU firmware may be needed during early boot:

```
sudo update-initramfs -u -k all
```

Then reboot:

```
sudo reboot
```

### 4. Check the driver after reboot

Run:

```
dmesg | grep -iE 'amdgpu|firmware'
```

You should no longer see:

```
Failed to load firmware "polaris10_sdma.bin"
```

Also check the GPU:

```
lspci -nnk | grep -A 3 -i 'vga\|display'
```

You want something similar to:

```
Kernel driver in use: amdgpu
Kernel modules: amdgpu
```

## Then, make the GPU usable for compute

The firmware problem is **only the first layer**. Having `amdgpu` loaded does not automatically mean that CUDA/OpenCL/ROCm compute is ready.

For a Polaris GPU, I'd first establish exactly what GPU you have:

```
lspci -nn | grep -iE 'vga|3d|display'
```

and:

```
sudo dmesg | grep -i amdgpu
```

Also:

```
ls -l /dev/dri/
```

Normally you should see something like:

```
card0
renderD128
```

The important device for unprivileged GPU compute is generally the **render node**, e.g.:

```
/dev/dri/renderD128
```

Check its permissions:

```
ls -l /dev/dri/renderD128
```

and your groups:

```
groups
```

If your user isn't in `render`, you can add it:

```
sudo usermod -aG render $USER
```

Then log out and back in.


### Confirm `amdgpu` is actually driving the card

Run:

```
lspci -nnk -s 05:00.0
```

You want:

```
Kernel driver in use: amdgpu
Kernel modules: amdgpu
```

Also:

```
ls -l /dev/dri/
```

Ideally you'll have:

```
card0
renderD128
```

The `renderD128` device is particularly important for compute applications.

### Check the GPU from the kernel

Run:

```
sudo dmesg | grep -i amdgpu | tail -50
```

## OpenCL

Polaris is an older architecture, so **current ROCm support is much more restrictive than for newer AMD GPUs**. Therefore, I'd first get the kernel driver and OpenCL working rather than blindly installing the newest ROCm packages.

### Check OpenCL support

After fixing `amdgpu`, install the basic OpenCL tooling:

```
sudo apt install clinfo
```

Then:

```
clinfo
```

If the GPU is exposed through OpenCL, you should see an AMD GPU platform/device.

You can also check:

```
clinfo | grep -iE 'platform|device|amd|gpu' | head -50
```

## Setup OpenCL for Polaris
### 1. First check the GPU/driver state

Run:

```
lspci -nnk -s 05:00.0
```

You should have:

```
Kernel driver in use: amdgpu
```

Then:

```
ls -l /dev/dri/
```

You should have a `renderD*` device, for example:

```
card0
renderD128
```

Also check:

```
ls -l /etc/OpenCL/vendors/
```

If that directory is empty or doesn't exist, that explains your `clinfo` result.

## 2. Install the Mesa OpenCL implementation

For a Polaris card, **Mesa's Rusticl OpenCL implementation** is a good modern route to investigate.

On Debian 13:

```
sudo apt update
sudo apt install mesa-vulkan-drivers mesa-opencl-icd
```

Then check:

```
ls -l /etc/OpenCL/vendors/
```

and:

```
clinfo
```

You want `Number of platforms` to become at least:

```
Number of platforms  1
```

with an AMD GPU device underneath.

---

## 3. If `mesa-opencl-icd` isn't available

Check what Debian offers:

```
apt search opencl | grep -E 'mesa|rusticl|rocm|amd'
```

Also:

```
apt policy mesa-opencl-icd
```

If Debian doesn't provide the package in your enabled repositories, **don't randomly install an AMD `.deb` from the internet**. We can determine the appropriate repository/package based on your Debian configuration.

---

## 4. Check whether Mesa/Rusticl sees the GPU

Install:

```
sudo apt install mesa-utils
```

Then:

```
glxinfo -B
```

You should see something identifying the renderer as your AMD GPU.

You can also check Vulkan:

```
vulkaninfo --summary
```

if `vulkan-tools` is installed:

```
sudo apt install vulkan-tools
```

## Check ICD

### 1. First, check what those ICD files contain

Run:

```
cat /etc/OpenCL/vendors/mesa.icd
cat /etc/OpenCL/vendors/rusticl.icd
```

I'd expect something along the lines of:

```
libRusticlOpenCL.so.1
```

or another Mesa OpenCL library.

Then check whether the libraries actually exist:

```
ldconfig -p | grep -iE 'OpenCL|rusticl'
```

Also:

```
dpkg -L mesa-opencl-icd | grep -E '\.so|rusticl'
```

## 2. Check Rusticl directly

Your `rusticl.icd` is particularly interesting because Rusticl is Mesa's modern OpenCL implementation.

Try:

```
RUSTICL_ENABLE=radeonsi clinfo
```

If that still gives:

```
Number of platforms 0
```

try:

```
RUSTICL_ENABLE=all clinfo
```

If Rusticl isn't automatically selecting RadeonSI, `RUSTICL_ENABLE=radeonsi` can expose that problem.

## 3. Check Mesa's GPU detection independently

Install the diagnostic tools:

```
sudo apt install mesa-utils
```

Then:

```
glxinfo -B
```

Look for:

```
OpenGL renderer string:
```

It should identify your RX 580 / RadeonSI GPU.

You can also run:

```
MESA_DEBUG=1 glxinfo -B
```

If you are running this on a machine without a graphical session, `glxinfo` may fail because there is no X/Wayland display. That's okay; we'll use another test.

# 4. Check whether the render node is accessible

Your permissions are:

```
crw-rw---- 1 root render 226,128 renderD128
```

That means access is controlled by the `render` group.

Check:

```
id
```

You should see:

```
groups=...,render,...
```

If `render` isn't there:

```
sudo usermod -aG render $USER
```

Then **log out and log back in**.

You can verify with:

```
id
```

This is particularly important if you're eventually going to run GPU applications or Docker containers as a non-root user.

# 5. Check the firmware problem again

Because we originally had:

```
Failed to load firmware "polaris10_sdma.bin"
```

I would also verify that this is completely gone:

```
dmesg | grep -iE 'amdgpu|firmware' | grep -iE 'error|fail|firmware'
```

And specifically:

```
dmesg | grep -i polaris10
```

If you still see:

```
polaris10_sdma.bin
```

then fix that before going further.

Check:

```
ls -lh /lib/firmware/amdgpu/polaris10_sdma.bin
```

---

References:
- [Debian Wiki: AMDGPU](https://wiki.debian.org/AMDGPU)
- [Mesa OpenCL](https://docs.mesa3d.org/opencl.html)
- [AMD ROCm Documentation](https://rocmdocs.amd.com/en/latest/)
- [Vulkan Tools](https://packages.debian.org/bullseye/vulkan-tools)
- [Mesa Utils](https://packages.debian.org/bullseye/mesa-utils)