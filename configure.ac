
AC_INIT([kvazaar], m4_esyscmd([printf $(awk '/#define KVZ_VERSION/ { print $3 }' src/global.h)]))
AC_CONFIG_SRCDIR([src/encmain.c])

# Library version number, modify:
# - When modifying kvazaar.h.
#   - Modify either major or minor.
# - When making a new release.
#   - If major or minor did not change since last release.
#      - Check git history to see if someone forgot to increment major or minor.
#      - Increment release.
#
# major:
#   - Increment when ABI changes, meaning lib users need to be recompiled.
#   - ABI changes when anything existing gets modified, including sizes of structs.
# minor:
#   - Increment when only API changes in a backwards compatible way without breaking ABI.
#   - We count adding parameters to bottom of kvz_config as ABI compatible, because user
#     shouldn't copy that struct or care about it's size.
#   - If not sure, increment major instead.
# release:
#   - Increment when making new releases and major or minor was not changed since last release.
#
# Here is a somewhat sane guide to lib versioning: http://apr.apache.org/versioning.html
ver_major=7
ver_minor=4
ver_release=0

# Prevents configure from adding a lot of defines to the CFLAGS
AC_CONFIG_HEADERS([config.h])

AC_CONFIG_MACRO_DIR([m4])
AC_CONFIG_AUX_DIR([build-aux])

AM_INIT_AUTOMAKE([-Wall dist-bzip2 dist-xz foreign subdir-objects])
AM_SILENT_RULES([yes])

AC_PROG_CC
AC_PROG_CC_C99
AM_PROG_AR
AC_PROG_CXX

# Get fread that can read more than 2GB on 32 bit systems.
AC_SYS_LARGEFILE

LT_INIT([win32-dll])

AC_CANONICAL_HOST

flag_gcc_on_mingw="false"
case x"${host_os}" in
  x"cygwin"*|x"mingw"*)
    if test x"${CC}" = x"gcc" ; then
      flag_gcc_on_mingw="true"
    fi
esac

AX_CHECK_COMPILE_FLAG([-maltivec],[flag_altivec="true"])
AX_CHECK_COMPILE_FLAG([-mavx2],   [flag_avx2="true"])
AX_CHECK_COMPILE_FLAG([-msse4.1], [flag_sse4_1="true"])
AX_CHECK_COMPILE_FLAG([-msse2],   [flag_sse2="true"])
AX_CHECK_COMPILE_FLAG([-mbmi],    [flag_bmi="true"])
AX_CHECK_COMPILE_FLAG([-mabm],    [flag_abm="true"])
AX_CHECK_COMPILE_FLAG([-mpopcnt], [flag_popcnt="true"])
AX_CHECK_COMPILE_FLAG([-mlzcnt],  [flag_lzcnt="true"])
AX_CHECK_COMPILE_FLAG([-mbmi2],   [flag_bmi2="true"])

# Do we need -mpopcnt and -mlzcnt, or -mabm to use POPCNT and LZCNT
# instructions? Ask GCC and Clang, and they have different answers.
AM_CONDITIONAL([HAVE_ALTIVEC], [test x"$flag_altivec" = x"true"])
AM_CONDITIONAL([HAVE_AVX2_GCC], [test x"$flag_avx2" = x"true" -a x"$flag_bmi" = x"true" -a x"$flag_abm" = x"true" -a x"$flag_bmi2" = x"true" -a x"$flag_gcc_on_mingw" = x"false"])
AM_CONDITIONAL([HAVE_AVX2_CLANG], [test x"$flag_avx2" = x"true" -a x"$flag_bmi" = x"true" -a x"$flag_popcnt" = x"true" -a x"$flag_lzcnt" = x"true" -a x"$flag_bmi2" = x"true" -a x"$flag_gcc_on_mingw" = x"false"])
AM_CONDITIONAL([HAVE_SSE4_1], [test x"$flag_sse4_1" = x"true"])
AM_CONDITIONAL([HAVE_SSE2], [test x"$flag_sse2" = x"true"])

KVZ_CFLAGS="-Wall -Wextra -Wvla -Wno-sign-compare -Wno-unused-parameter -I$srcdir/src -I$srcdir/src/extras -ftree-vectorize -fvisibility=hidden"
CFLAGS="$KVZ_CFLAGS $CFLAGS"

AC_SEARCH_LIBS([log], [m c], [], [exit 1])
AC_SEARCH_LIBS([pow], [m c], [], [exit 1])
AC_SEARCH_LIBS([sqrt], [m c], [], [exit 1])

AC_ARG_WITH([cryptopp],
    AS_HELP_STRING([--with-cryptopp],
        [Build with cryptopp Enables selective encryption.]))
AS_IF([test "x$with_cryptopp" = "xyes"],
    [PKG_CHECK_MODULES([cryptopp], [cryptopp],
        [AC_DEFINE([KVZ_SEL_ENCRYPTION], [1], [With cryptopp])],
        [PKG_CHECK_MODULES([cryptopp], [libcrypto++],
            [AC_DEFINE([KVZ_SEL_ENCRYPTION], [1], [With cryptopp])],
            [PKG_CHECK_MODULES([cryptopp], [libcryptopp],
                [AC_DEFINE([KVZ_SEL_ENCRYPTION], [1], [With cryptopp])],
                [AC_MSG_ERROR([neither cryptopp, libcrypto++ nor libcryptopp found with pkg-config])]
            )]
        )]
    )]
)

AM_CONDITIONAL([USE_CRYPTOPP], [test "x$with_cryptopp" = "xyes"])
CPPFLAGS="$CPPFLAGS $cryptopp_CFLAGS"
LIBS="$LIBS $cryptopp_LIBS"


CPPFLAGS="-DKVZ_DLL_EXPORTS $CPPFLAGS"


# We need to force AX_PTHREAD to check -pthread -lpthread since otherwise
# it only outputs -pthread for GCC. Without -lpthread GCC does not link the
# shared library against the pthread library (even though it does link the
# executable).
PTHREAD_CFLAGS=-pthread
PTHREAD_LIBS=-lpthread

# This does workarounds for pthreads on various compilers.
AX_PTHREAD([],[AC_MSG_ERROR([POSIX threads not found])])

CFLAGS="$PTHREAD_CFLAGS $CFLAGS"
LIBS="$PTHREAD_LIBS $LIBS"
CC="$PTHREAD_CC"

# --enable-werror
AC_ARG_ENABLE([werror], [AS_HELP_STRING([--enable-werror], [treat warnings as errors [no]])],
              [CFLAGS="-Werror $CFLAGS"], []
)


# host and cpu specific settings
AS_CASE([$host_cpu],
        [i?86], [BITS="32" ASFLAGS="$ASFLAGS -DARCH_X86_64=0" X86="true"],
        [x86_64], [BITS="64" ASFLAGS="$ASFLAGS -DARCH_X86_64=1 -m amd64" X86="true"],
        [powerpc*], [PPC="true"]
)

AS_CASE([$host_os],
        [darwin*], [
         ASFLAGS="$ASFLAGS -f macho$BITS -DPREFIX"
        ],
        [cygwin*|msys*|mingw*], [
         CFLAGS="$CFLAGS -D__USE_MINGW_ANSI_STDIO=1"
         # Fix a bug in mingw gcc where stack doesn't get aligned properly, force all AVX instructions to be unaligned
         AS_CASE([$CC], [*gcc],[
             AX_CHECK_COMPILE_FLAG([-Wa,-muse-unaligned-vector-move],
                 [CFLAGS="-Wa,-muse-unaligned-vector-move $CFLAGS"],
                 [
                    AC_MSG_CHECKING([if compiler is gcc])
                    AS_IF([$CC --version | grep "gcc" >/dev/null 2>&1], [
                        AS_ECHO(["yes"])
                        AC_MSG_ERROR([-Wa,-muse-unaligned-vector-move not supported, required with mingw+gcc to fix alignment bugs, update the used gcc])
                    ],[
                        AS_ECHO(["no"])
                        AS_ECHO(["Compiler not gcc, -Wa,-muse-unaligned-vector-move not needed"])
                    ])
                ]
             )
         ])
         AS_IF(
               [test "x$BITS" = "x32"], [
                ASFLAGS="$ASFLAGS -fwin32 -DPREFIX -DHAVE_ALIGNED_STACK=0"
               ], [
                ASFLAGS="$ASFLAGS -fwin64 -DHAVE_ALIGNED_STACK=1"
               ]
         )
        ],
         [midipix*], [
         AS_IF(
               [test "x$BITS" = "x32"], [
                ASFLAGS="$ASFLAGS -fwin32 -DPREFIX -DHAVE_ALIGNED_STACK=0"
               ], [
                ASFLAGS="$ASFLAGS -fwin64 -DHAVE_ALIGNED_STACK=1"
               ]
         )
        ],
        [linux*|*kfreebsd*], [
         ASFLAGS="$ASFLAGS -f elf$BITS"
         LDFLAGS="$LDFLAGS -Wl,-z,noexecstack"
         LIBS="$LIBS -lrt"
        ], [
         ASFLAGS="$ASFLAGS -f elf$BITS"
         LDFLAGS="$LDFLAGS -Wl,-z,noexecstack"
        ]
)


AM_CONDITIONAL([HAVE_X86], [test "x$X86" = "xtrue"])
AM_CONDITIONAL([HAVE_PPC], [test "x$PPC" = "xtrue"])
AM_CONDITIONAL([HAVE_ARM], [test "x$ARM" = "xtrue"])

AC_ARG_VAR([ASFLAGS], [ASFLAGS to use for assembler])
AC_SUBST([ASFLAGS])

KVZ_API_VERSION="$ver_major:$ver_minor:$ver_release"
AC_SUBST([KVZ_API_VERSION])

AC_CONFIG_FILES([Makefile
                 src/Makefile
                 src/kvazaar.pc
                 tests/Makefile])
AC_OUTPUT
