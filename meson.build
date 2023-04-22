project(
  'meson-test', 'c', 'cython',
  meson_version: '>= 0.64.0',
  default_options: [
    'buildtype=debugoptimized',
    'c_std=c99',
  ]
)

gmp = dependency('gmp', version: '>= 6.2.1', fallback: ['gmp', 'gmp_dep'])

py = import('python').find_installation(pure: false)

py.extension_module(
  'meson_test',
  'meson_test.pyx',
  dependencies: gmp,
  install: true,
)

#pyx = custom_target(
  #  'meson-test.pyx',
  #output: 'meson-test.pyx',
  #input: 'meson-test.pyx.in',
  #command: [
    #  py,
    #'-c',
    #'import sys; sys.stdout.write(sys.stdin.read().replace("@@VERSION@@", "@0@"))'.format(meson.project_version()),
    #'@INPUT@',
    #],
  #)


