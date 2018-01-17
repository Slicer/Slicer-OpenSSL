
This project allows to configure, build, archive and upload an OpenSSL install tree.

# instructions

Steps using cmake-gui:

1. Download CMakeLists.txt into a folder such as `C:\path\OpenSSL-packages`
2. Create a build folder such as `C:\path\OpenSSL-packages-build`
3. Run CMake, set source code and binary paths to the folders listed above, and press Configure
4. Choose a generator
5. Set configuration variables such as username, API key, and path to Perl executable, then press Generate to create the build files
6. Open the solution file in `C:\path\OpenSSL-packages-build` and build the solution

# license

This project is covered by the OSI-approved BSD 3-clause License.

OpenSSL is distributed under the [OpenSSL License][openssl-license]. For more information about OpenSSL, visit https://www.openssl.org/

[openssl-license]: https://www.openssl.org/source/license.html
