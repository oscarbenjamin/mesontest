project('gmp', 'c',
  version: '6.2.1',
  meson_version: '>= 0.65.0',
  license: ['LGPL-3.0-only', 'GPL-2.0-only'],
)

mod = import('unstable-external_project')

p = mod.add_project('configure',
  configure_options: [
    '--prefix=@PREFIX@',
    '--libdir=@PREFIX@/@LIBDIR@',
    '--includedir=@PREFIX@/@INCLUDEDIR@',
    '--enable-fat',
    '--enable-shared=yes',
    '--enable-static=no',
  ],
)

gmp_dep = p.dependency('gmp')
meson.override_dependency('gmp', gmp_dep)
