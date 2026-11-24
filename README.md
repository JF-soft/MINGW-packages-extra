# MINGW-packages-extra

Additional package recipes for building MinGW-w64 packages in MSYS2. Each
package has its own directory containing a `PKGBUILD` and any supporting
patches.

## Building a package

From an MSYS2 environment with the required build dependencies installed:

```bash
cd <package-directory>
MINGW_ARCH=ucrt64 makepkg-mingw -sLf
```

When using the UCRT64 terminal, `MINGW_ARCH` is set automatically, so you can
build directly with:

```bash
cd <package-directory>
makepkg-mingw -sLf
```

Install the resulting package with `pacman -U`.

## License

The BSD-3-Clause license in [LICENSE](LICENSE) applies to original contributions
in this repository. Package recipes and other third-party material retain their
respective notices and licenses. The `license` field in each `PKGBUILD`
describes the packaged software, not the license of this repository.
