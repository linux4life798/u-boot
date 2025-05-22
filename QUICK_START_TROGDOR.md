# Quick Start

On Debian, do the following:

```bash
sudo apt install crossbuild-essential-arm64 libgnutls28-dev

export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64

make clean -j
make chromebook_trogdor_defconfig
time make -j$(nproc)
```


## Resources

* [Chromium OS Support in U-Boot](https://docs.u-boot.org/en/stable/chromium/overview.html)
