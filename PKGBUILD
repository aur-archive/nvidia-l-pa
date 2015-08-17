# 
# Maintainer: Ninez
# Contributors: Det, Ng Oon-Ee, Dan Vratil
# Based on [extra]'s nvidia: https://www.archlinux.org/packages/extra/x86_64/nvidia/
#
# Please note the depends() array. I have included nvidia-utils-beta, since that is what i 
# am using, but thought i should track Archlinux nvidia packages by default. 
#
# So you can switch to nvidia-beta by commenting/uncommenting the correct line.

pkgname=nvidia-l-pa
pkgver=352.09
_major=$(uname -r | cut -d '.' -f-2)
_extramodules=extramodules-3.18-l-pa
pkgrel=2
pkgdesc="NVIDIA driver for linux (for Linux-l-pa)"
arch=('i686' 'x86_64')
url="http://www.nvidia.com/"
#depends=('linux-l-pa>=3.18' 'linux-l-pa<3.19' "nvidia-libgl" "nvidia-utils-beta=${pkgver}")
depends=('linux-l-pa>=3.18' 'linux-l-pa<3.19' "nvidia-libgl" "nvidia-utils=${pkgver}")
makedepends=('linux-l-pa-headers>=3.18' 'linux-l-pa-headers<3.19')
conflicts=('nvidia-96xx' 'nvidia-173xx')
provides=()
license=('custom:NVIDIA')
install=${pkgname}.install
options=('!strip')

if [ "$CARCH" = "i686" ]; then
    _arch='x86'
    _pkg="NVIDIA-Linux-${_arch}-${pkgver}"
elif [ "$CARCH" = "x86_64" ]; then
    _arch='x86_64'
    _pkg="NVIDIA-Linux-${_arch}-${pkgver}-no-compat32"
fi

#source=("http://us.download.nvidia.com/XFree86/Linux-${_arch}/${pkgver}/${_pkg}.run"
source=("ftp://download.nvidia.com/XFree86/Linux-${_arch}/${pkgver}/${_pkg}.run"
         'nvidia-rt_explicit.patch'
         'nvidia-rt_no_wbinvd.patch'
         'nvidia-rt_mutexes.patch'
         'nvidia_kbuildCommands_verbose.patch'
         'nvidia_useLDbfd_avoid_LDgold.patch')

md5sums=('eb5ad6a07dc03e0a19d5f6fa069c494b'
         'e376dbfd525b444b831c61653d27fde1'
         '095e596d9745c31561cf6385201721de'
         'f08f15f364595e24268d2cd7bb17cf5c'
         '709ea9fb4ecca0c5b0e515f2616ea09f'
         '3da42b65865d8eb1463517ae2ef26bd6')

build() {
    _kernver="$(cat /usr/lib/modules/${_extramodules}/version)"

    # Remove previous builds
    if [ -d ${_pkg} ]; then
        rm -rf ${_pkg}
    fi

    # Extract
    sh ${_pkg}.run --extract-only
    cd ${_pkg}/kernel

####### explicitly set #define NV_CONFIG_PREEMPT_RT 1 but allow for swapping patch defines 

    msg2 "apply nvidia-rt_explicit.patch"
    patch -Np1 --dry-run -i $srcdir/nvidia-rt_explicit.patch

####### Disable wbinvdt() call [Intel instruction] on PREEMPT_RT_FULL 

    # wbinvdt() causes huge latency spikes, but not using it may be unstable
    # on some systems. you have now been WARNED... I've done some testing with CUDA 5.5/6.0, to which
    # i have not found problems, but my nvidia cards are older / 2.1 compute support. it could be that
    # older cards are fine here, while compute 3.0+ devices could reveal serious instability. 
    # this i don't know - I'll probably pick up a new NV card sometime soon to test...
    #
    # You must also uncomment to use, at your own risk. [where it works, it's awesome]

    # msg2 "apply nvidia-rt-no-wbinvd.patch"
    patch -Np1 -i $srcdir/nvidia-rt_no_wbinvd.patch

####### replace nVidia's semaphore code with Mutexes on PREEMPT_RT_FULL

    # this resolves [non-fatal] scheduling bugs reported in kernel ring buffer / dmesg. 
    # nvidia-rt_mutexes.patch was considered experimental, but after months of use 
    # on several peices of H/W - i am confident it will work best for everyone.

    # Add supporting mutex code [switching struct[s], semaphore/mutex, #defines, add supporting code] 

    # msg2 "apply nvidia-rt_mutexes.patch"
    patch -Np1 -i $srcdir/nvidia-rt_mutexes.patch

####### Pulled from Debian - avoid ld.gold, as it can be problematic for nvidia

    msg2 "apply nvidia_useLDbfd_avoid_LDgold.patch"
    patch -Np1 -i $srcdir/nvidia_useLDbfd_avoid_LDgold.patch

####### Pulled from Debian/SteamOS - make kbuild commands verbose

#    msg2 "apply nvidia_kbuildCommands_verbose.patch"
#    patch -Np1 -i $srcdir/nvidia_kbuildCommands_verbose.patch

####### Build module

    msg2 "Starting make module..."
    make IGNORE_PREEMPT_RT_PRESENCE=1 SYSSRC=/usr/lib/modules/"${_kernver}/build" module

    # Build nvidia-uvm module

    cd uvm
    msg2 "Building UVM module for ${_kernel}...: https://devblogs.nvidia.com/parallelforall/unified-memory-in-cuda-6/"
    make IGNORE_PREEMPT_RT_PRESENCE=1 SYSSRC=/usr/lib/modules/"${_kernver}/build" module
}

package() {
    # Install/compress modules

    # nvidia module
    install -D -m644 "${srcdir}/${_pkg}/kernel/nvidia.ko" \
        "${pkgdir}/usr/lib/modules/${_extramodules}/nvidia.ko"

    # Unified Memory module
    install -Dm644 "${srcdir}/${_pkg}/kernel/uvm/nvidia-uvm.ko" \
        "${pkgdir}/usr/lib/modules/${_extramodules}/nvidia-uvm.ko"

    # Compress
    gzip "${pkgdir}/usr/lib/modules/${_extramodules}/nvidia.ko"
    gzip "${pkgdir}/usr/lib/modules/${_extramodules}/nvidia-uvm.ko"

    # Write $_extramodules to .install
    sed -i "s/_extramodules='.*'/_extramodules='${_extramodules}'/" "${startdir}/${install}"

    # Blacklist Nouveau
    install -d  -m755 "${pkgdir}/usr/lib/modprobe.d"
    echo "blacklist nouveau" >> "${pkgdir}/usr/lib/modprobe.d/nvidia-l-pa.conf"

    # Enable MSI for nvidia-l-pa (ie: prevent nvidia from sharing an interrupt)
    install -d -m755 "${pkgdir}/etc/modprobe.d"
    echo "blacklist nouveau" >> "${pkgdir}/etc/modprobe.d/nouveau_blacklist-nvidia-l-pa.conf"
}
