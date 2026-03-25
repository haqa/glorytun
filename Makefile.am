ACLOCAL_AMFLAGS = -I m4 --install

bin_PROGRAMS = glorytun

glorytun_CFLAGS = $(libsodium_CFLAGS)
glorytun_LDADD = $(libsodium_LIBS)
glorytun_SOURCES = \
    argz/argz.c \
    argz/argz.h \
    mud/aegis256/aegis256.c \
    mud/aegis256/aegis256.h \
    mud/mud.c \
    mud/mud.h \
    src/argz.c \
    src/argz.h \
    src/bench.c \
    src/bind.c \
    src/common.c \
    src/common.h \
    src/ctl.c \
    src/ctl.h \
    src/iface.c \
    src/iface.h \
    src/ip.h \
    src/keygen.c \
    src/list.c \
    src/main.c \
    src/path.c \
    src/set.c \
    src/show.c \
    src/tun.c \
    src/tun.h \
    src/version.c

EXTRA_DIST = \
    LICENSE \
    Makefile \
    README.md \
    VERSION \
    autogen.sh \
    meson.build \
    meson_options.txt \
    mud/LICENSE \
    mud/README.md \
    sodium.sh \
    scripts/ci/build-static-binary.sh \
    scripts/client/openrc/README.md \
    scripts/client/openrc/glorytun-client \
    scripts/client/openrc/glorytun-client.conf \
    scripts/client/systemd/README.md \
    scripts/client/systemd/glorytun-client \
    scripts/client/systemd/glorytun-client-start \
    scripts/client/systemd/glorytun-client-teardown \
    scripts/client/systemd/glorytun-client.service \
    scripts/server/openrc/README.md \
    scripts/server/openrc/glorytun-server \
    scripts/server/openrc/glorytun-server.conf \
    scripts/server/systemd/README.md \
    scripts/server/systemd/glorytun-server \
    scripts/server/systemd/glorytun-server-start \
    scripts/server/systemd/glorytun-server@.service \
    version.sh
