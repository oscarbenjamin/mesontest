This is a test of meson-python attempting to build a project with a Cython
extension module that depends on GMP. It tries to use a system GMP if installed
with a new enough version or otherwise uses a meson subproject to download and
build GMP.

The intention is to satisfy these uses:

- It should be possible to setup a development environment from a VCS checkout
  ideally in editable mode.
- It should be possible to build/install from an sdist
- It should be possible to build wheels in CI and those should be relocatable.

When installing from VCS/sdist there are two further cases:

- The system already has GMP installed and the extension module should link to
  that.
- The system does not have GMP or does not have a new enough version of GMP and
  then ideally meson would download and build GMP.

In the case of the VCS checkout and editable mode we want to have incremental
rebuilds so e.g. each rebuild of the extension module should be able to reuse
the existing build of GMP.

Since GMP is has an autotools build system we use meson's
`unstable-external_project` feature which handles a configure then make setup.
We need to provide a subprojects/packagefiles/meson.build which tells meson how
to do this. We use a `wrap-file` stub to tell meson to download the code for
GMP.

So far this does not succeed in buliding a wheel.

The extension module can be built with:
```bash
meson setup build
ninja -C build
```
That will produce the extension module linking against system GMP if it is
provided. Otherwise it downloads, configures and builds GMP and then makes an
extension module linking against the locally built GMP. The built files for GMP
look like:
```console
 tree build/subprojects/gmp-6.2.1/dist/
build/subprojects/gmp-6.2.1/dist/
└── usr
    └── local
        ├── include
        │   └── gmp.h
        ├── lib
        │   └── x86_64-linux-gnu
        │       ├── libgmp.la
        │       ├── libgmp.so -> libgmp.so.10.4.1
        │       ├── libgmp.so.10 -> libgmp.so.10.4.1
        │       ├── libgmp.so.10.4.1
        │       └── pkgconfig
        │           └── gmp.pc
        └── share
            └── info
                ├── dir
                ├── gmp.info
                ├── gmp.info-1
                └── gmp.info-2

8 directories, 10 files
```
If we `cd` into the `build` directory we can import and use the extension
module and call `gmp` functions:
```console
$ cd build/
oscar@nuc:~/current/active/tmp/mesontest/build$ python
Python 3.11.3 (main, Apr  5 2023, 23:03:48) [GCC 11.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import meson_test
>>> meson_test.pow1000(2)
"b'10715086071862673209484250490600018105614048117055336074437503883703510511249361224931983788156958581275946729175531468251871452856923140435984577574698574803934567774824230985421074605062371141877954182153046474983581941267398767559165543946077062914571196477686542167660429831652624386837205668069376'"
>>> 2**1000
10715086071862673209484250490600018105614048117055336074437503883703510511249361224931983788156958581275946729175531468251871452856923140435984577574698574803934567774824230985421074605062371141877954182153046474983581941267398767559165543946077062914571196477686542167660429831652624386837205668069376
```
So everything is built correctly and seems to work.

We can also build a wheel when the system has GMP installed:
```console
$ sudo apt-get install libgmp3-dev
...
The following additional packages will be installed:
  libgmp-dev libgmpxx4ldbl
...
```
Now building a wheel succeeds:
```console
$ python -m build
...
Run-time dependency gmp found: YES 6.2.1
...
Copying files to wheel...
[0/0] meson_test.cpython-311-x86_64-linux-gnu.soo                                                
Successfully built meson_test-0.0.1.tar.gz and meson_test-0.0.1-cp311-cp311-linux_x86_64.whl
```
The wheel can be installed and the extension module works with the system GMP
at runtime.

What fails is building a wheel with mesonpy when there is no system GMP
installed:
```console
$ sudo apt-get remove libgmp3-dev libgmp-dev libgmpxx4ldbl
$ python -m build
...

...
meson-python: error: Could not map installation path to an equivalent wheel
directory: '{prefix}'
```
Everything builds correctly but mesonpy does not understand how to package the
artifacts into a wheel.

The same is seen if using mesonpy without build isolation:
```console
$ python -c 'import mesonpy; mesonpy.build_wheel(".")'
...
meson-python: error: Could not map installation path to an equivalent wheel
directory: '{prefix}'
```
Without build isolation though I can patch mesonpy with this diff:
```diff
--- __init__.py.backup	2023-04-23 11:50:55.660441459 +0100
+++ __init__.py	2023-04-23 11:51:03.836722723 +0100
@@ -160,7 +160,7 @@ def _map_to_wheel(sources: Dict[str, Dic
 
             path = _INSTALLATION_PATH_MAP.get(anchor)
             if path is None:
-                raise BuildError(f'Could not map installation path to an equivalent wheel directory: {str(destination)!r}')
+                continue
 
             if path == 'purelib' or path == 'platlib':
                 package = destination.parts[1]

```
Now it builds with
```console
$ python -c 'import mesonpy; mesonpy.build_wheel(".")'
...
Copying files to wheel...
[0/0] meson_test.cpython-311-x86_64-linux-gnu.soo
```
The resulting wheel does not contain libgmp.so though:
```console
$ unzip meson_test-0.0.1-cp311-cp311-linux_x86_64.whl 
Archive:  meson_test-0.0.1-cp311-cp311-linux_x86_64.whl
 extracting: meson_test-0.0.1.dist-info/METADATA  
 extracting: meson_test-0.0.1.dist-info/WHEEL  
 extracting: meson_test.cpython-311-x86_64-linux-gnu.so  
 extracting: meson_test-0.0.1.dist-info/RECORD  
$ tree
.
├── meson_test-0.0.1-cp311-cp311-linux_x86_64.whl
├── meson_test-0.0.1.dist-info
│   ├── METADATA
│   ├── RECORD
│   └── WHEEL
└── meson_test.cpython-311-x86_64-linux-gnu.so

1 directory, 5 files
```

