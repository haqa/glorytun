project('glorytun', 'c',
    version: run_command('./version.sh', check: false).stdout().strip(),
    license: 'BSD-3-Clause',
    default_options : [
      'buildtype=debugoptimized',
      'c_std=gnu99'
    ]
)

add_global_arguments('-DPACKAGE_VERSION="'+meson.project_version()+'"', language : 'c')
add_global_arguments('-DPACKAGE_NAME="'+meson.project_name()+'"', language : 'c')

static_link = get_option('static')
libsodium_dep = dependency('libsodium', version : '>=1.0.4', static: static_link)

exe_link_args = []
if static_link and host_machine.system() == 'linux'
  exe_link_args += ['-static']
endif

executable('glorytun', install: true,
  sources: [
    'argz/argz.c',
    'mud/aegis256/aegis256.c',
    'mud/mud.c',
    'src/argz.c',
    'src/bench.c',
    'src/bind.c',
    'src/common.c',
    'src/ctl.c',
    'src/iface.c',
    'src/keygen.c',
    'src/list.c',
    'src/main.c',
    'src/path.c',
    'src/set.c',
    'src/show.c',
    'src/tun.c',
    'src/version.c',
  ],
  dependencies: [libsodium_dep],
  link_args: exe_link_args,
)

# Systemd units and network snippets live under scripts/{client,server}/systemd/
# and are installed manually or by packaging; see those READMEs.
