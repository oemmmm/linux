# Maintainer: JustAMannn
#
# Ядра LeksOS. Этот PKGBUILD собирает любой из пяти видов ядра; вид задаёт
# переменная LEKSOS_FLAVOR:
#
#   desktop  linux-leksos         обычное: работа, учёба, ноутбуки
#   game     linux-leksos-game    игровое
#   server   linux-leksos-server  серверное
#   rt       linux-leksos-rt      реального времени
#   lts      linux-leksos-lts     запасное, на ветке с долгой поддержкой
#
#   LEKSOS_FLAVOR=game makepkg -s
#
# Исходники берутся по метке форка на GitHub. Если задан LEKSOS_TREE, ядро
# собирается из готового дерева: так собирает GitHub Actions, и так удобно
# проверять свои правки до коммита.
#
#   LEKSOS_TREE=~/linux LEKSOS_FLAVOR=desktop makepkg -s

_flavor=${LEKSOS_FLAVOR:-desktop}

# Версии ядра и номера сборок LeksOS: метки форка называются
# v<версия>-leksos<номер>, а номер сборки становится pkgrel.
# Эти строки меняет ci/update.
_stable=7.3.1
_stable_rev=1
_lts=6.18.60
_lts_rev=1

case $_flavor in
	desktop) _suffix=''      _what='обычное: работа, учёба, ноутбуки' ;;
	game)    _suffix=-game   _what='игровое' ;;
	server)  _suffix=-server _what='серверное' ;;
	rt)      _suffix=-rt     _what='реального времени' ;;
	lts)     _suffix=-lts    _what='запасное, с долгой поддержкой' ;;
	*)
		echo "неизвестный вид ядра '$_flavor': desktop, game, server, rt или lts" >&2
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
pkgdesc="ядро LeksOS ($_what)"
url=https://github.com/JustAMannn/linux
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
sha256sums=('8bdfd269004e9463c61e0b3b9d9df42ab6e31aa6457df64fba052819454fc549'
            '7e556c303dcb58bef7a4dfc7b7bde7823a459b9ace31c52a55bf603fa8b306a4'
            'eb70f3cebeb2eee1adcaeaadd4ddc34237c920b43896d0863842b85539d91a35')
if [[ -z ${LEKSOS_TREE:-} ]]; then
	# Архив метки из вашего же форка. GitHub не подписывает такие архивы,
	# поэтому сумма не проверяется.
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
		# GitHub называет каталог архива без «v» в начале метки.
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

	# uname -r: 7.3.1-1-leksos, 7.3.1-1-leksos-game и т. д.
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
	pkgdesc="$pkgdesc — ядро и модули"
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
	# systemd ищет ядро здесь (нужно для гибернации)
	install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

	# По этому файлу mkinitcpio называет ядро в /boot
	echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

	echo "модули..."
	ZSTD_CLEVEL=19 make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
		DEPMOD=/doesnt/exist modules_install # depmod запустит хук mkinitcpio

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
	pkgdesc="$pkgdesc — заголовки и скрипты для сборки модулей"
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

	# нужно при STACK_VALIDATION
	install -Dt "$builddir/tools/objtool" tools/objtool/objtool

	# нужно при DEBUG_INFO_BTF_MODULES
	install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/resolve_btfids

	echo "заголовки..."
	cp -t "$builddir" -a include
	cp -t "$builddir/arch/x86" -a arch/x86/include
	install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

	install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
	install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

	# https://bugs.archlinux.org/task/13146
	install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

	# https://bugs.archlinux.org/task/20402
	install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
	install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
	install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

	# https://bugs.archlinux.org/task/71392
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
		link= # без ссылок build-id

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
			application/x-sharedlib\;*)      # библиотеки .so
				strip -v $STRIP_SHARED "$file" ;;
			application/x-archive\;*)        # библиотеки .a
				strip -v $STRIP_STATIC "$file" ;;
			application/x-executable\;*)     # программы
				strip -v $STRIP_BINARIES "$file" ;;
			application/x-pie-executable\;*) # перемещаемые программы
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
