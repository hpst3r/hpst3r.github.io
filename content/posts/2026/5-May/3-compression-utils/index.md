---
title: "Archival/compression utilities on Linux"
date: 2026-05-09T20:10:00-00:00
draft: false
---

There are five main archival utilities available on most Linux systems:

- tar
- gzip
- xz
- bzip2
- zip

tar creates uncompressed archives; it was originally intended for use creating tape backups, but now finds usage anywhere you'd like to turn a bunch of files into one file. Files are stored linearly in the tarball, so it's very cheap (computationally) to work with tar archives.

tar archives are often combined with a compression utility, which is where the `tar.xz`, `tar.gz`, `tar.bz2` files come from. On \*nix platforms we tend to have separate archival utilities (like tar, used for turning a set of files or directory into a single file) and compression utilities (e.g., gzip, bzip2, xz, used to reduce the amount of disk space a file uses), in contrast to the Windows approach of a single tool for both (e.g., `zip`, `rar`, `7z`).

The four "zip" utilities use different algorithms, offering different ratios of speed to compression:

- gzip is the fastest, and uses the least memory, but offers the (relative) lowest compression ratio.
- xz at moderate compression levels is faster than bzip2 and offers better compression. Modern hardware is so fast that you should probably default to this.
- bzip2 is an older compression format that offers good compression at the cost of slow computation. No real reason to use it over xz.
- zip is the format traditionally used on Windows.. it offers decent compression, and is a compression utility as well as an archival utility. This means that you can "zip" a directory. In addition, if you do zip a directory, each file is separately compressed (so you can extract just one file out of a directory) - though this means compression of redundant data in multiple files in the archive suffers.

To create a tar archive:

```sh
tar cf dest.tar source1 source2 source3
```

To gzip a file:

```sh
gzip file.txt
```

This will replace the source with a compressed `file.txt.gz`. To keep the file, pass the `-k` (`--keep`) argument.

To decompress a file, run `gzip -d` or `gunzip` the compressed file.

To bzip a file:

```sh
bzip2 file.txt
```

This will replace the source with a compressed `file.txt.bz2`. To keep the file, pass the `-k` (`--keep`) argument.

To decompress a file, pass the `-d` argument or `bunzip2` the file.

To xz a file:

```sh
xz file.txt
```

Same deal with `--keep`, `--decompress`, and `unxz`.

To zip a file or directory:

```sh
zip files.zip file1.txt file2.txt
```

To unzip, simply specify `unzip`.

To assemble a directory or set of files into a tarball and compress it with gzip, with one command:

```sh
tar -czvf files.tar.gz file1.txt file2.txt
```

The arguments are:

- `-c` (create archive)
- `-z` (gzip)
- `-v` (verbose)
- `-f` (outfile)

If you wanted to `xz` the tarball instead, use the `-J` parameter in place of `-z`.

To extract (and decompress) a tarball:

```sh
tar -xvf files.tar.gz
```

To extract to a specific directory, specify the `-C` parameter and a target directory:

```sh
tar -xvf files.tar.gz -C /tmp/example
```

To list the contents of a tar archive, use the `-t` (`--list`) parameter:

```sh
$ tar -tvf files.tar.gz
-rw-r--r-- wporter/wporter   0 2026-04-21 21:01 file1.txt
-rw-r--r-- wporter/wporter   0 2026-04-21 21:01 file2.txt
```

With GNU tar, compression will be automatically handled - you don't need to specify compression type.

If needed (not GNU tar, or automatic detection doesn't work), specify the "compression" parameter (`-z`, `-J`, etc) in addition to the other arguments.
