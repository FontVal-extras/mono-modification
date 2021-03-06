# Modifications to Mono for FontVal on recent Mac OS X

Recent Mac OS X have tighter security. Specifically, The material in this repository is to address the two code-signing related issues with using
mono's mkbundle:

- https://github.com/mono/mono/issues/18826 . There is a pull request to merge https://github.com/HinTak/mono/tree/issue18826-fix . This was merged
in 6.13 onwards and backported in mono-6.12.0.98 onwards; but not yet available to `mkbundle --fetch-target ...` yet.

- https://github.com/mono/mono/issues/17881

## Using mkbundle for Mac OS X

Seeing as the problem is not going to have a clean and short solution any time soon, here is the detailed process of using mkbundle
to build standalone binaries compatible with Apple's Hardened Runtime, System Integrity Protection (SIP) and Notarization:

- You need at least mono 6.12.0.98+ or mono 6.13+

- Remove original signature if using recent native default mono runtime on Mac OS X. Either
`remove-code-signature.py /Library/Frameworks/Mono.framework/Versions/Current/bin/mono-sgen64` or `codesign -v --remove-signature ...` should be fine.

- Run `mkbundle ...` . The libgdiplus-dependent older Winforms GUI is very troublesome to build, and has its own
1000-character command line in `mkbundle-support/FontVal32-osx-build` and extensive commentary therein.

- Run `mono-codesign-fix.py` to fix up offsets.

- Create entitlement file and re-sign with extra entitlement. `com.apple.security.cs.disable-library-validation` is definitely needed.
The original mono binary was signed with
`com.apple.security.cs.allow-jit`,
`com.apple.security.cs.allow-unsigned-executable-memory`,
`com.apple.security.cs.allow-dyld-environment-variables`,
and
`com.apple.security.cs.disable-library-validation`. So the other 3 might be needed for other more sophisticated usage of `mkbundle`.

- Set up some provisioning profile to test?

## Unpacking Mono's Mac OS X installer on non-Mac OS X

```
wget -m https://download.mono-project.com/archive/6.12.0/macos-10-universal/MonoFramework-MDK-6.12.0.122.macos10.xamarin.universal.pkg
mkdir -p /tmp/Mono
xar -C /tmp/Mono -xpvf "download.mono-project.com/archive/6.12.0/macos-10-universal/MonoFramework-MDK-6.12.0.122.macos10.xamarin.universal.pkg"
zcat /tmp/Mono/mono.pkg/Payload | cpio -m -d --extract
```

`install_name_script` in the `mkbundle-support` directory.


```
find libs -type f -name '*.dylib' -exec install_name_script {} \;
find libs -type f -name '*.dylib' -exec x86_64-apple-darwin9-otool -L {} \;
```

The dependency chains are:

```
libexpat.1.6.7.dylib
libgif.4.1.6.dylib
libintl.8.dylib
libjpeg.8.dylib
libpixman-1.0.dylib
libpng14.14.dylib
libpng14.14.dylib -> libfreetype.6.dylib
libpng14.14.dylib -> libfreetype.6.dylib -> libfontconfig.1.dylib
libjpeg.8.dylib -> libtiff.5.dylib
libpixman-1.0.dylib -> [ libpng14.14.dylib -> libfreetype.6.dylib -> libfontconfig.1.dylib ] -> libcairo.2.dylib
libintl.8.dylib -> libglib-2.0.0.dylib
#
libexpat.1.6.7.dylib -> libgif.4.1.6.dylib -> libpixman-1.0.dylib ->
[ libjpeg.8.dylib -> libtiff.5.dylib ] -> [ libintl.8.dylib -> libglib-2.0.0.dylib ] -> [ [ libpng14.14.dylib -> libfreetype.6.dylib -> libfontconfig.1.dylib ] -> libcairo.2.dylib ] -> libgdiplus.dylib
```

Pre-modified libraries are in `mkbundle-support/libs/`.

## Others

Winforms applications do not work under 64-bit mono. See [Running WinForms apps in x64 mode on macOS](https://github.com/mono/mono/issues/6701).
Mac OS X 10.15 (Catalina) onwards is 64-bit only, and loses the ability to run 32-bit applications.

Anything (even non-GUI) that depends on libfontconfig takes a while to start the first time, from rebuilding the font cache.
