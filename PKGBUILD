# Maintainer: Sven-Hendrik Haase <svenstaro@gmail.com>
# Contributor:  Bartłomiej Piotrowski <bpiotrowski@archlinux.org>
# Contributor: Allan McRae <allan@archlinux.org>

# toolchain build order: linux-api-headers->glibc->binutils->gcc->binutils->glibc
# NOTE: libtool requires rebuilt with each new gcc version

pkgname=(gcc10 gcc10-libs)
pkgver=10.0.1.r175763.ae6fc5ce437
_majorver=${pkgver%%.*}
_basever=${pkgver%%.r*}
_islver=0.21
pkgrel=1
pkgdesc='The GNU Compiler Collection (Git version)'
arch=(x86_64)
license=(GPL LGPL FDL custom)
url='http://gcc.gnu.org'
makedepends=(binutils libmpc doxygen python)
checkdepends=(dejagnu inetutils)
options=(!emptydirs)
source=(git+https://gcc.gnu.org/git/gcc.git#commit=ae6fc5ce43788db6962ca657e8c0d17017c1a6e2
        http://isl.gforge.inria.fr/isl-${_islver}.tar.xz
        c89 c99)
validpgpkeys=(F3691687D867B81B51CE07D9BBE43771487328A9  # bpiotrowski@archlinux.org
              86CFFCA918CF3AF47147588051E8B148A9999C34  # evangelos@foutrelis.com
              13975A70E63C361C73AE69EF6EEB81F8981C74C7  # richard.guenther@gmail.com
              33C235A34C46AA3FFB293709A328C3A2C3C45C06) # Jakub Jelinek <jakub@redhat.com>
sha256sums=('SKIP'
            '777058852a3db9500954361e294881214f6ecd4b594c00da5eee974cd6a54960'
            'de48736f6e4153f03d0a5d38ceb6c6fdb7f054e8f47ddd6af0a3dbf14f27b931'
            '2513c6d9984dd0a2058557bf00f06d8d5181734e41dcfe07be7ed86f2959622a')

_svnrev=264010
_svnurl=svn://gcc.gnu.org/svn/gcc/branches/gcc-${_majorver}-branch
#also set after pkgver() runs!
_libdir=usr/lib/gcc/$CHOST/${_basever}

# snapshot() is only used by core's maintainers, so removing it here

pkgver() {
  cd gcc
  echo $(cat gcc/BASE-VER).r$(git rev-list --count HEAD).$(git rev-parse --short HEAD)
}

prepare() {
  [[ ! -d gcc ]] && ln -s gcc-${pkgver/+/-} gcc
  cd gcc

  # link isl for in-tree build
  ln -s ../isl-${_islver} isl

  # Do not run fixincludes
  sed -i 's@\./fixinc\.sh@-c true@' gcc/Makefile.in

  # Arch Linux installs x86_64 libraries /lib
  sed -i '/m64=/s/lib64/lib/' gcc/config/i386/t-linux64

  # hack! - some configure tests for header files using "$CPP $CPPFLAGS"
  sed -i "/ac_cpp=/s/\$CPPFLAGS/\$CPPFLAGS -O2/" {libiberty,gcc}/configure

  mkdir -p "$srcdir/gcc-build"
}

build() {
  cd gcc-build

  # using -pipe causes spurious test-suite failures
  # http://gcc.gnu.org/bugzilla/show_bug.cgi?id=48565
  CFLAGS=${CFLAGS/-pipe/}
  CXXFLAGS=${CXXFLAGS/-pipe/}

  #Revsion 261304 removed libmpx
  #https://gcc.gnu.org/viewcvs/gcc?view=revision&revision=261304
  #https://gcc.gnu.org/wiki/Intel%20MPX%20support%20in%20the%20GCC%20compiler
  #https://www.phoronix.com/scan.php?page=news_item&px=MPX-Removed-From-GCC9
  "$srcdir/gcc/configure" --prefix=/usr \
      --libdir=/usr/lib \
      --libexecdir=/usr/lib \
      --mandir=/usr/share/man \
      --infodir=/usr/share/info \
      --with-bugurl=https://bugs.archlinux.org/ \
      --enable-languages=c,c++,lto \
      --enable-shared \
      --enable-threads=posix \
      --with-system-zlib \
      --with-isl \
      --enable-__cxa_atexit \
      --disable-libunwind-exceptions \
      --enable-clocale=gnu \
      --disable-libstdcxx-pch \
      --disable-libssp \
      --enable-gnu-unique-object \
      --enable-linker-build-id \
      --enable-lto \
      --enable-plugin \
      --enable-install-libiberty \
      --with-linker-hash-style=gnu \
      --enable-gnu-indirect-function \
      --disable-multilib \
      --disable-werror \
      --enable-checking=release \
      --enable-default-pie \
      --enable-default-ssp \
      --enable-cet=auto \
      --program-suffix=-${_majorver} \
      --enable-version-specific-runtime-libs

  make ${MAKEFLAGS}

  # make documentation
  make -C $CHOST/libstdc++-v3/doc doc-man-doxygen
}

check() {
  cd gcc-build

  # do not abort on error as some are "expected"
  make -k check || true
  "$srcdir/gcc/contrib/test_summary"
}

package_gcc10-libs() {
  pkgdesc='Runtime libraries shipped by GCC (Git version)'
  depends=('glibc>=2.27')
  options+=(!strip)

  cd gcc-build
  make -C $CHOST/libgcc DESTDIR="$pkgdir" install-shared
  mv "$pkgdir"/$_libdir/../lib/* "$pkgdir"/$_libdir
  rmdir "$pkgdir"/$_libdir/../lib
  rm -f "$pkgdir/$_libdir/libgcc_eh.a"

  for lib in libatomic \
             libgomp \
             libitm \
             libquadmath \
             libsanitizer/{a,l,ub,t}san \
             libstdc++-v3/src \
             libvtv; do
    make -C $CHOST/$lib DESTDIR="$pkgdir" install-toolexeclibLTLIBRARIES
  done

  # Install Runtime Library Exception
  install -Dm644 "$srcdir/gcc/COPYING.RUNTIME" \
    "$pkgdir/usr/share/licenses/gcc10-libs/RUNTIME.LIBRARY.EXCEPTION"
}

package_gcc10() {
  pkgdesc="The GNU Compiler Collection - C and C++ frontends (Git version)"
  depends=("gcc10-libs=$pkgver-$pkgrel" 'binutils>=2.28' libmpc)
  options+=(staticlibs)

  cd gcc-build

  make -C gcc DESTDIR="$pkgdir" install-driver install-cpp install-gcc-ar \
    c++.install-common install-headers install-plugin install-lto-wrapper

  install -m755 -t "$pkgdir/${_libdir}/" gcc/{cc1,cc1plus,collect2,lto1,gcov,gcov-tool}

  make -C $CHOST/libgcc DESTDIR="$pkgdir" install
  rm -r "$pkgdir"/${_libdir}/../lib

  make -C $CHOST/libstdc++-v3/src DESTDIR="$pkgdir" install
  make -C $CHOST/libstdc++-v3/include DESTDIR="$pkgdir" install
  make -C $CHOST/libstdc++-v3/libsupc++ DESTDIR="$pkgdir" install
  make -C $CHOST/libstdc++-v3/python DESTDIR="$pkgdir" install
  rm "$pkgdir"/${_libdir}/libstdc++.so*

  make DESTDIR="$pkgdir" install-fixincludes
  make -C gcc DESTDIR="$pkgdir" install-mkheaders

  make -C lto-plugin DESTDIR="$pkgdir" install

  make -C $CHOST/libgomp DESTDIR="$pkgdir" install-nodist_{libsubinclude,toolexeclib}HEADERS
  make -C $CHOST/libitm DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS
  make -C $CHOST/libquadmath DESTDIR="$pkgdir" install-nodist_libsubincludeHEADERS
  make -C $CHOST/libsanitizer DESTDIR="$pkgdir" install-nodist_{saninclude,toolexeclib}HEADERS
  make -C $CHOST/libsanitizer/asan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS
  make -C $CHOST/libsanitizer/tsan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS
  make -C $CHOST/libsanitizer/lsan DESTDIR="$pkgdir" install-nodist_toolexeclibHEADERS

  make -C libcpp DESTDIR="$pkgdir" install

  # many packages expect this symlink
  ln -s gcc-10 "$pkgdir"/usr/bin/cc-10

  # byte-compile python libraries
  python -m compileall "$pkgdir/usr/share/gcc-${_majorver}/"
  python -O -m compileall "$pkgdir/usr/share/gcc-${majorver}/"

  # Install Runtime Library Exception
  install -d "$pkgdir/usr/share/licenses/$pkgname/"
  ln -s /usr/share/licenses/gcc10-libs/RUNTIME.LIBRARY.EXCEPTION \
    "$pkgdir/usr/share/licenses/$pkgname/"

  # Remove conflicting files
  rm -r "$pkgdir"/usr/share/locale
}

