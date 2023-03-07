_mode=cross
_kernelname=sm8150-nabu
pkgbase="linux-$_kernelname"
_desc="Xiaomi Nabu kernel"
_srcname=linux
pkgver=5.19.0
pkgrel=1
_arches=specific
arch=(aarch64)
license=(GPL2)
url=https://github.com/torvalds/linux.git
makedepends=(
    xmlto
    docbook-xsl
    kmod
    inetutils
    bc
    dtc
    cpio
)
options=(!strip)
source=(
    "https://github.com/map220v/sm8150-mainline/archive/refs/heads/nabu-5.19.zip"
    config-nabu
    linux.preset
    60-linux.hook
    90-linux.hook
)
sha256sums=(
    SKIP
    SKIP
    66644820faa950a5fc59181f5aefcbed6d7ed652b29aee69979a2be2a032025d
    ae2e95db94ef7176207c690224169594d49445e04249d2499e9d2fbc117a0b21
    c20dce380025de175fe9a551b247b3fb05ecf068eef63e773f82b36be9568270
)

pkgver() {
    cd ${_srcname}
    make kernelversion | cut -d "-" -f 1
}

prepare() {
    mv sm8150-mainline-nabu-5.19 ${_srcname}
    mv config-nabu ${_srcname}/.config
    cd ${_srcname}
    
    # don't run depmod on "make install". We'll do this ourselves in packaging
    sed -i "2iexit 0" scripts/depmod.sh
}

build() {
    cd ${_srcname}
    unset LDFLAGS
    # shellcheck disable=SC2086
    make ${MAKEFLAGS} Image.gz modules
    # shellcheck disable=SC2086
    make ${MAKEFLAGS} DTC_FLAGS="-@" dtbs
}

_package() {
    pkgdesc="The Linux Kernel and modules - ${_desc}"
    depends=(coreutils kmod mkinitcpio)
    optdepends=("crda: to set the correct wireless channels of your country")
    backup=(etc/mkinitcpio.d/linux.preset)
    install=linux.install

    cd ${_srcname}

    KARCH=arm64

    # get kernel version
    _kernver="$(make kernelrelease)"
    _basekernel=${_kernver%%-*}
    _basekernel=${_basekernel%.*}

    mkdir -p "${pkgdir}"/{boot,usr/lib/modules}
    make INSTALL_MOD_PATH="${pkgdir}/usr" modules_install
    mkdir -p "${pkgdir}/boot/dtbs/qcom"
    cp arch/arm64/boot/dts/qcom/sm8150-xiaomi-nabu.dtb \
        "${pkgdir}/boot/dtbs/qcom"
    cp arch/$KARCH/boot/Image.gz "${pkgdir}/boot"

    # make room for external modules
    local _extramodules="extramodules-${_basekernel}${_kernelname}"
    ln -s "../${_extramodules}" "${pkgdir}/usr/lib/modules/${_kernver}/extramodules"

    # add real version for building modules and running depmod from hook
    echo "${_kernver}" |
        install -Dm644 /dev/stdin "${pkgdir}/usr/lib/modules/${_extramodules}/version"

    # remove build and source links
    rm "${pkgdir}/usr/lib/modules/${_kernver}/"{source,build}

    # now we call depmod...
    depmod -b "${pkgdir}/usr" -F System.map "${_kernver}"

    # sed expression for following substitutions
    local _subst="
    s|%PKGBASE%|${pkgbase}|g
    s|%KERNVER%|${_kernver}|g
    s|%EXTRAMODULES%|${_extramodules}|g
  "

    # install mkinitcpio preset file
    sed "${_subst}" ../linux.preset |
        install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/linux.preset"

    # install pacman hooks
    sed "${_subst}" ../60-linux.hook |
        install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/60-${pkgbase}.hook"
    sed "${_subst}" ../90-linux.hook |
        install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/90-${pkgbase}.hook"
}

_package-headers() {
    pkgdesc="Header files and scripts for building modules for linux kernel - ${_desc}"
    provides=("linux-headers=${pkgver}")
    conflicts=(linux-headers linux-aarch64-headers)

    cd ${_srcname}
    local _builddir="${pkgdir}/usr/lib/modules/${_kernver}/build"

    install -Dt "${_builddir}" -m644 Makefile .config Module.symvers
    install -Dt "${_builddir}/kernel" -m644 kernel/Makefile

    mkdir "${_builddir}/.tmp_versions"

    cp -t "${_builddir}" -a include scripts

    install -Dt "${_builddir}/arch/${KARCH}" -m644 arch/${KARCH}/Makefile
    install -Dt "${_builddir}/arch/${KARCH}/kernel" -m644 arch/${KARCH}/kernel/asm-offsets.s #arch/$KARCH/kernel/module.lds

    cp -t "${_builddir}/arch/${KARCH}" -a arch/${KARCH}/include
    mkdir -p "${_builddir}/arch/arm"
    cp -t "${_builddir}/arch/arm" -a arch/arm/include

    install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
    install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

    # http://bugs.archlinux.org/task/13146
    install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

    # http://bugs.archlinux.org/task/20402
    install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
    install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
    install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

    # add xfs and shmem for aufs building
    mkdir -p "${_builddir}"/{fs/xfs,mm}

    # copy in Kconfig files
    find . -name Kconfig\* -exec install -Dm644 {} "${_builddir}/{}" \;

    # remove unneeded architectures
    local _arch
    for _arch in "${_builddir}"/arch/*/; do
        [[ ${_arch} == */${KARCH}/ || ${_arch} == */arm/ ]] && continue
        rm -r "${_arch}"
    done

    # remove files already in linux-docs package
    rm -r "${_builddir}/Documentation"

    # remove now broken symlinks
    find -L "${_builddir}" -type l -printf "Removing %P\n" -delete

    # Fix permissions
    chmod -R u=rwX,go=rX "${_builddir}"

    # strip scripts directory
    local _binary _strip
    while read -rd "" _binary; do
        case "$(file -bi "${_binary}")" in
        *application/x-sharedlib*) _strip="${STRIP_SHARED}" ;;    # Libraries (.so)
        *application/x-archive*) _strip="${STRIP_STATIC}" ;;      # Libraries (.a)
        *application/x-executable*) _strip="${STRIP_BINARIES}" ;; # Binaries
        *) continue ;;
        esac
        /usr/bin/strip "${_strip}" "${_binary}"
    done < <(find "${_builddir}/scripts" -type f -perm -u+w -print0 2>/dev/null)
}

pkgname=("${pkgbase}" "${pkgbase}-headers")
for _p in "${pkgname[@]}"; do
    eval "package_${_p}() {
    _package${_p#${pkgbase}}
  }"
done
