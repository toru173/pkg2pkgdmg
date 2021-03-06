# pkg2pkgdmg

***Usage: pkg2pkgdmg inputfile outputfile***

Converts a pkg (xar) file with an internal dmg into a hybrid pkgdmg file that will open as either a disk image or installer package, depending on the extension. This pkgdmg can then be optionally signed using tools that support signing a xar or pkg file. These signed files are often used by the macOS installer, preventing the installation dmg from being modified by malicious parties.

This should have never been written as a bash script. Bash (and shells in general) are terrible for manipulating binary files but *It Works(tm)* so I'm not changing it now. I initially started as a shell script to get extra practice writing them and things just snowballed out of control! Feel free to reimpliment in the language of your choice.

## Background

A pkgdmg is a combination file that acts as a macOS disk Image (a dmg) with the extension .dmg, and as a macOS installation file (a package) with the extension .pkg. The only thing one needs to do to change the behaviour of the file is to change the extension. This works because packages are effectively xar archives (good primer [here](https://github.com/mackyle/xar/wiki/xarformat)) which don't care about any data that isn't in the file's xml-based Table of Contents.

![](https://github.com/toru173/pkg2pkgdmg/blob/main/Images/InstallESD%20Package%20xar%21%20magic.png)

![](https://github.com/toru173/pkg2pkgdmg/blob/main/Images/InstallESD%20Internal%20koly%20block.png)

The pkgdmg file format uses this as an advantage as it simply concatenates dmg-relevant information (a 'koly block', good primer [here](http://newosxbook.com/DMG.html)) on to the end of the xar file, which is ignored when the OS treats the pkgdmg as a pkg file. Likewise, the plist that defines the behaviour of the disk image is at a fixed offset in the middle of the image, and contains a number of 'mish' blocks detailing the information contained within the disk image. If it's not in the mish blocks, it's not in the disk image so the leading (and possibly trailing) xar information is ignored.

![](https://github.com/toru173/pkg2pkgdmg/blob/main/Images/InstallESD%20koly%20block%20footer.png)

Key things of note:

- A pkgdmg is essentially a dmg enclosed in an (optionally) signed xar file, with supporting files expected per a distribution-style package (good primer [here](http://s.sudre.free.fr/Stuff/Ivanhoe/FLAT.html))
- The files in a pkgdmg are not compressed. Though parts may be - such as the dmg itself - the significant files are uncompressed as the system needs to know exact byte offset within the pkgdmg to make things work, and they need to be able to be read byte-for-byte from the xar archive
- The inner dmg at /Archive/of/OS/installer.dmg appears to be a normal, zlib-compressed dmg - one that can be create with hdiutil

![](https://github.com/toru173/pkg2pkgdmg/blob/main/Images/InstallESD%20Package.png)

## Usage

Editing a pkgdmg using normal tools results in the loss of the duality - depending on the tools used, it devolves into a pkg or dmg file. We can extract the pkg file easily to a local directory to edit using xar and can rebuild the pkgdmg once we've finished editing the pkg. To do this, we need to patch the pkg file by extracting the koly block from the dmg within the archive, append it to the file and edit the offsets within the block to point to the dmg within the archive.

![](https://github.com/toru173/pkg2pkgdmg/blob/main/Images/InstallESD%20koly%20block.png)

![](https://github.com/toru173/pkg2pkgdmg/blob/main/Images/New%20InstallESD%20koly%20block.png)

This is automated by pkg2pkgdmg, so editing and repackaging the pkgdmg becomes as simple as:

    cp /path/to/InstallESD.dmg .
    mkdir InstallESD
    cd InstallESD
    sudo xar -x -P -f ../InstallESD.dmg
    < Edits to Internal InstallESD.dmg go here >
    sudo xar -c --compression=none -f ../New\ InstallESD.pkg *
    cd ..
    sudo ./pkg2pkgdmg New\ InstallESD.pkg New\ InstallESD.dmg
    


