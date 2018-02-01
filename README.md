
This project allows to configure, build, archive and upload an OpenSSL install tree.

For any given Visual Studio generator, three archives are generated and uploaded to the corresponding [GitHub release](https://github.com/Slicer/Slicer-OpenSSL/releases):

* `OpenSSL_<OPENSSL_VERSION>-install-msvc<MSVC_VERSION>-<bitness>-Debug.tar.gz`
* `OpenSSL_<OPENSSL_VERSION>-install-msvc<MSVC_VERSION>-<bitness>-Release.tar.gz`
* `OpenSSL_<OPENSSL_VERSION>-install-msvc<MSVC_VERSION>-<bitness>.tar.gz`

where

* `<OPENSSL_VERSION>` is of the form `<major>_<minor>_<patch><letter>`
* `<MSVC_VERSION>` is a [four digit number](https://cmake.org/cmake/help/latest/variable/MSVC_VERSION.html) identifying visual studio
* `<bitness>` is either `32` or `64`

For example:

* `OpenSSL_1_0_2n-install-msvc1900-64-Debug.tar.gz`
* `OpenSSL_1_0_2n-install-msvc1900-64-Release.tar.gz`
* `OpenSSL_1_0_2n-install-msvc1900-64.tar.gz`


# prerequisites

* [Visual Studio](https://www.visualstudio.com/), [CMake](https://cmake.org/), [StawberryPerl](http://strawberryperl.com/) and [github-release](https://github.com/j0057/github-release#readme).

# instructions

Steps using cmake-gui:

1. Git clone this repository (or download `CMakeLists.txt`) into a folder such as `C:\path\Slicer-OpenSSL`
2. Create a build folder such as `C:\path\Slicer-OpenSSL-build`
3. Run cmake-gui, set source code and binary paths to the folders listed above, and press Configure
4. Choose a generator
5. Set configuration variables such as `PERL_COMMAND`, `GITHUB_RELEASE_EXECUTABLE` and `GITHUB_TOKEN`, then press Generate to create the build files
6. Open the solution file in `C:\path\Slicer-OpenSSL-build` and build the solution

# license

This project is covered by the OSI-approved BSD 3-clause License.

OpenSSL is distributed under the [OpenSSL License][openssl-license]. For more information about OpenSSL, visit https://www.openssl.org/

[openssl-license]: https://www.openssl.org/source/license.html
