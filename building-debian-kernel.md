# Building Debian Linux Kernel

## Official Debian packages 

1. Grab the source using `# apt-get install linux-source-X.X`
2. Extract the source from `/usr/src/linux-source-X.X.tar.xz`
3. `$ make mrproper`
4. Copy configuration from `/boot/config-x` to `.config`
5. Comment out the lines `CONFIG_SYSTEM_TRUSTED_KEY` and `CONFIG_MODULE_SIG_KEY` or run `$ sed -ri '/CONFIG_SYSTEM_TRUSTED_KEYS/s/=.+/=""/g' .config` [1]
6. `$ make -j4 bindeb-pkg`

## Apply patches

Run `$ patch -p1 < x.patch` in the kernel source directory.

## Resources
- "Common kernel-related tasks" from Debian Linux Kernel Handbook: [https://kernel-team.pages.debian.net/kernel-handbook/ch-common-tasks.html](https://kernel-team.pages.debian.net/kernel-handbook/ch-common-tasks.html)

## References
- [1] [https://unix.stackexchange.com/a/294116](https://unix.stackexchange.com/a/294116)

