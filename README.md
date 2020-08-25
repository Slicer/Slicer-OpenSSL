
# overview

This project has two roles:
* configure, build, archive and automatically upload Release and Debug OpenSSL install trees as [release assets](https://github.com/Slicer/Slicer-OpenSSL/releases)
* organize OpenSSL source archives in a [dedicated release](https://github.com/Slicer/Slicer-OpenSSL/releases/tag/sources)

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

* install [Visual Studio](https://www.visualstudio.com/), [CMake](https://cmake.org/), [StawberryPerl](http://strawberryperl.com/) and [github-release](https://github.com/j0057/github-release#readme).

# guides

## upload a new source archives

1. Download official OpenSSL archive from https://www.openssl.org/source/
2. Upload archive in the [source release](https://github.com/Slicer/Slicer-OpenSSL/releases/tag/sources)
3. Update release description

## add support for building a new version of OpenSSL

1. Attempt to configure and build OpenSSL disabling upload (see instructions below)
2. Two possible scenarios:
  - If it succeeds, publish a new tag associated with the current master branch
  - If it fails, update the [CMakeLists.txt](CMakeLists.txt) and try again

## build and upload install tree archives for a specific Visual Studio version

Archives for OpenSSL version associated with the CMake variable `OPENSSL_VERSION` are built. Inspect the
[CMakeLists.txt](CMakeLists.txt) to find out which version is built by default.

1. Git clone this repository (or download `CMakeLists.txt`) into a folder such as `C:\path\Slicer-OpenSSL`
2. Create a build folder such as `C:\path\Slicer-OpenSSL-build`
3. Open Visual Studio terminal for the desired version and architecture
4. Configure specifying matching Visual Studio generator and architecture as well as `PERL_COMMAND`, `GITHUB_RELEASE_EXECUTABLE` and `GITHUB_TOKEN` options:
    ```
    set PERL_COMMAND=C:\StrawberryPerl-x64\perl\bin\perl.exe
    set GITHUB_RELEASE_EXECUTABLE=D:\Support\github-release-venv\Scripts\githubrelease.exe
    set GITHUB_TOKEN=xyz

    cd C:\path\Slicer-OpenSSL-build
    cmake.exe ^
      -DPERL_COMMAND=%PERL_COMMAND% ^
      -DGITHUB_RELEASE_EXECUTABLE=%GITHUB_RELEASE_EXECUTABLE% ^
      -DGITHUB_TOKEN=%GITHUB_TOKEN% ^
      -G "Visual Studio 16 2019" ^
      -A x64 ^
      -DUPLOAD_PACKAGES=ON ^
    ../Slicer-OpenSSL
    ```
5. Build
    Since `UPLOAD_PACKAGES` is `ON`, install tree archives are automatically uploaded as
    assets associated with the release named after the selected `OPENSSL_VERSION`.
    ```
    cmake.exe --build .
    ```
6. Once archives are uploaded, update the release description (see below)

## update release description

The following snippet was designed for updating the description of releases organizing
install tree archives and it should be executed in bash.

It also expects `githubrelease` command line tool to be in the `PATH`.


```bash
version=1.1.1g

cd /tmp

#
# download assets
#
githubrelease asset Slicer/Slicer-OpenSSL download ${version}

#
# compute SHA256 and generate release description
#
description_file=/tmp/Slicer-OpenSSL-${version}-release-description.txt
download_url_base=https://github.com/Slicer/Slicer-OpenSSL/releases/download/${version}
IFS=$'\n'
for line in $(sha256sum OpenSSL_${version//./_}*); do
  sha256=$(echo ${line} | cut -f1 -d" ");
  file=$(echo ${line} | cut -f3 -d" ");
  md5sum=$(md5sum ${file} | cut -f1 -d" ")
  file=${file##*/}
  echo -e "* [${file}]($download_url_base/${file})\n  * SHA256: \`${sha256}\`\n  * MD5: \`${md5sum}\`";
done > ${description_file}

cat ${description_file}

#
# Update release description
#
githubrelease release Slicer/Slicer-OpenSSL edit ${version} --body "$(cat ${description_file})"
```

# license

This project is covered by the OSI-approved BSD 3-clause License.

OpenSSL is distributed under the [OpenSSL License][openssl-license]. For more information about OpenSSL, visit https://www.openssl.org/

[openssl-license]: https://www.openssl.org/source/license.html
