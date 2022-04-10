# pkg2pkgdmg

***Usage: pkg2pkgdmg inputfile outputfile***

Converts a pkg (xar) file with an internal dmg into a hybrid pkgdmg file that will open as either a disk image or installer package, depending on the extension. This pkgdmg can then be optionally signed using tools that support signing a xar or pkg file. These signed files are often used by the macOS installer, preventing the installation dmg from being modified by malicious parties

This should have never been written as a bash script. Bash (and shells in general) are terrible for manipulating binary files but It Works(tm) so I'm not changing it now Feel free to reimpliment in the language of your choice

A pkgdmg is a combination file that acts as a dmg with the extension .dmg, and as a macOS installation file (a package) with the extension .pkg. The only thing you need to do to change the behaviour of the file is to change the extension. This works because packages are effectively xar archives (good primer [here](https://github.com/mackyle/xar/wiki/xarformat)) which don't care about any data that isn't in the file's XML-based Table of Contents. The pkgdmg file format uses this as an advantage as it simply concatenates dmg-relevant information (a 'koly block', good primer [here](http://newosxbook.com/DMG.html)) on to the end of the xar file, which is ignored when the OS treats the pkgdmg as a pkg file.

Things of note:

- A pkgdmg is essentially a dmg enclosed in an (optionally) signed xar file, with supporting files expected per a distribution-style package (good primer [here](http://s.sudre.free.fr/Stuff/Ivanhoe/FLAT.html))
- The files in a pkgdmg are not compressed. Though parts may be - such as the dmg itself - the significant files are uncompressed as the system needs to know exact byte offset within the pkgdmg to make things work, and they need to be able to be read byte-for-byte from the xar archive
- The inner dmg at /Archive/of/OS/installer/OSInstall.pkg/InstallESD.dmg appears to be a normal, zlib-compressed dmg - one that we can create with hdiutil

So what we need to do is patch the pkg file by extracting the koly block from the dmg within the archive, appending it to the file and editing the offsets within the block to point to the dmg within the archive.

    mkdir InstallESD
    cd InstallESD
    sudo xar -x -P -f ../InstallESD.dmg
    < Edits to InstallOS.pkg/InstallESD.dmg go here >
    sudo xar -c --compression=none -f ../New\ InstallESD.pkg *
    sudo ./pkg2pkgdmg New\ InstallESD.pkg New\ InstallESD.dmg

![](https://github.com/toru173/pkg2pkgdmg/blob/main/InstallESD%20koly%20block.png)

![](https://github.com/toru173/pkg2pkgdmg/blob/main/New%20InstallESD%20koly%20block.png)
