gmp = dependency('gmp', version: '>= 6.2.1', fallback: ['gmp', 'gmp_dep'])

# py = import('python').find_installation(pure: false)

py.extension_module(
  '_meson_test',
  '_meson_test.pyx',
  dependencies: gmp,
  install: true,
  subdir: 'meson_test',
)

py.install_sources(
  ['__init__.py'],
  subdir: 'meson_test',
)
