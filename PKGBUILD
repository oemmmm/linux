_flavor=${LEKSOS_FLAVOR:-desktop}
_stable=7.3.3
_stable_rev=1
_lts=6.18.62
_lts_rev=1

case $_flavor in
	desktop) _suffix=''      _what='обычное' ;;
	game)    _suffix=-game   _what='игровое' ;;
	server)  _suffix=-server _what='серверное' ;;
	rt)      _suffix=-rt     _what='реального времени' ;;
	lts)     _suffix=-lts    _what='запасное' ;;
	*)
		echo "неизвестный вид ядра" >&2
		exit 1
		;;
esac
if [[ $_flavor == lts ]]; then
	_ver=$_lts _rev=$_lts_rev
else
	_ver=$_stable _rev=$_stable_rev
fi

pkgbase=linux-leksos$_suffix
pkgver=$_ver
pkgrel=$_rev
pkgdesc="Ядро LeksOS: $_what"
url=https://github.com/oemmmm/linux
arch=(x86_64)
license=(GPL-2.0-only)
makedepends=(
	bc
	cpio
	gettext
	libelf
	pahole
	perl
	python
	rust
	rust-bindgen
	rust-src
	tar
	xz
	zstd
)
options=(
	!debug
	!strip
)
_tag=v$_ver-leksos$_rev
source=(
	check-config
	leksos-game.service
	leksos-game-scx.service
)
sha256sums=('SKIP' 'SKIP' 'SKIP')
if [[ -z ${LEKSOS_TREE:-} ]]; then
	source+=("linux-$_tag.tar.gz::$url/archive/refs/tags/$_tag.tar.gz")
	sha256sums+=('SKIP')
fi

export KBUILD_BUILD_HOST=leksos
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

_fragments=(leksos.config "leksos-$_flavor.config")

prepare() {
	rm -rf linux
	if [[ -n ${LEKSOS_TREE:-} ]]; then
		echo "исходники: $LEKSOS_TREE"
		mkdir linux
		tar -C "$LEKSOS_TREE" --exclude=.git -cf - . | tar -C linux -xf -
	else
		local dir
		for dir in "linux-${_tag#v}" "linux-$_tag"; do
			if [[ -d $dir ]]; then
				mv "$dir" linux
				break
			fi
		done
		if [[ ! -d linux ]]; then
			echo "в архиве $_tag нет каталога с исходниками" >&2
			return 1
		fi
	fi
	cd linux

	make -s mrproper
	echo "-$pkgrel" > localversion.10-pkgrel
	echo "${pkgbase#linux}" > localversion.20-pkgname

	echo "настройки: leksos_defconfig ${_fragments[*]}"
	make leksos_defconfig "${_fragments[@]}"
	bash "$srcdir/check-config" .config "${_fragments[@]/#/kernel/configs/}"

	make -s kernelrelease > version
	echo "подготовлено: $pkgbase $(<version)"
}

build() {
	cd linux
	make all
	make -C tools/bpf/bpftool vmlinux.h feature-clang-bpf-co-re=1
}

_package() {
	depends=(
		coreutils
		initramfs
		kmod
	)
	optdepends=(
		'linux-firmware: прошивки для многих устройств'
		'wireless-regdb: правильные каналы Wi-Fi для вашей страны'
	)
	if [[ $_flavor == game ]]; then
		optdepends+=('scx-scheds: игровой планировщик scx_lavd')
	fi
	provides=(
		KSMBD-MODULE
		NTSYNC-MODULE
		VIRTUALBOX-GUEST-MODULES
		WIREGUARD-MODULE
	)

	cd linux
	local modulesdir="$pkgdir/usr/lib/modules/$(<version)"

	echo "ядро..."
	install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"
	echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

	echo "модули..."
	ZSTD_CLEVEL=19 make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
		DEPMOD=/doesnt/exist modules_install

	rm "$modulesdir"/build

	if [[ $_flavor == game ]]; then
		echo "игровые службы..."
		local unit wants="$pkgdir/usr/lib/systemd/system/multi-user.target.wants"
		install -d "$wants"
		for unit in leksos-game.service leksos-game-scx.service; do
			install -Dm644 "$srcdir/$unit" "$pkgdir/usr/lib/systemd/system/$unit"
			ln -s "../$unit" "$wants/$unit"
		done
	fi
}

_package-headers() {
	pkgdesc="Заголовки ядра LeksOS для сборки модулей: $_what"
	depends=(pahole)

	cd linux
	local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

	echo "файлы сборки..."
	install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
		localversion.* version vmlinux tools/bpf/bpftool/vmlinux.h
	install -Dt "$builddir/kernel" -m644 kernel/Makefile
	install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
	cp -t "$builddir" -a scripts
	ln -srt "$builddir" "$builddir/scripts/gdb/vmlinux-gdb.py"
	install -Dt "$builddir/tools/objtool" tools/objtool/objtool
	install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/resolve_btfids

	echo "заголовки..."
	cp -t "$builddir" -a include
	cp -t "$builddir/arch/x86" -a arch/x86/include
	install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s
	install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
	install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h
	install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h
	install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
	install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
	install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h
	install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

	echo "файлы Kconfig..."
	find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

	if grep -q '^CONFIG_RUST=y' .config; then
		echo "файлы Rust..."
		install -Dt "$builddir/rust" -m644 rust/*.rmeta
		install -Dt "$builddir/rust" rust/*.so
	fi

	echo "vDSO без удаления символов..."
	make INSTALL_MOD_PATH="$pkgdir/usr" vdso_install \
		link=

	echo "лишние архитектуры..."
	local arch
	for arch in "$builddir"/arch/*/; do
		[[ $arch = */x86/ ]] && continue
		rm -r "$arch"
	done

	echo "документация..."
	rm -r "$builddir/Documentation"

	echo "битые ссылки..."
	find -L "$builddir" -type l -printf 'удалено: %P\n' -delete

	echo "объектные файлы..."
	find "$builddir" -type f -name '*.o' -printf 'удалено: %P\n' -delete

	echo "удаление отладочных символов из программ сборки..."
	local file
	while read -rd '' file; do
		case "$(file -Sib "$file")" in
			application/x-sharedlib\;*)
				strip -v $STRIP_SHARED "$file" ;;
			application/x-archive\;*)
				strip -v $STRIP_STATIC "$file" ;;
			application/x-executable\;*)
				strip -v $STRIP_BINARIES "$file" ;;
			application/x-pie-executable\;*)
				strip -v $STRIP_SHARED "$file" ;;
		esac
	done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

	echo "удаление отладочных символов из vmlinux..."
	strip -v $STRIP_STATIC "$builddir/vmlinux"

	echo "ссылка в /usr/src..."
	mkdir -p "$pkgdir/usr/src"
	ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=(
	"$pkgbase"
	"$pkgbase-headers"
)
for _p in "${pkgname[@]}"; do
	eval "package_$_p() {
		$(declare -f "_package${_p#$pkgbase}")
		_package${_p#$pkgbase}
	}"
done
