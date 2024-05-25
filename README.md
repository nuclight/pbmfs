# PBMFS - Petabyte of Metadata File System

This is a filesystem designed for the very special case: storing small number of mostly very big files up to 256 Tb in size - that is, SQLite databases. No sparse files, no advanced features - but allow it to be embedded in partition in such a way that most parts are unused by this filesystem but "proxied" to "upper-layer" consumer, e.g. another filesystem.

It is part of a larger project, yet to be burn - a filesystem using SQLite for metadata. But **NOT** a thing you would usually think when you hear "SQLite filesystem", that is, where entire filesystem and all your files are put into a single SQLite database file. No, this is for "traditional" filesystem - that is, placed on a raw block device (disk partition). And for *such* way you can't "just" take SQLite, because SQLite works with files via standard POSIX API, so you immediately get a chicken-and-egg problem - to implement such filesystem, you need some filesystem where to put SQLite files! Okay, for "large production" installations this actually may be a Good Thing - e.g. ZFS allows `special` vdev type for keeping metadata, so putting metadata on SSD mey speed up accessing main data on HDDs. But what is you have just two disks for mirror or even a single disk at home?..

So PBMFS is needed to solve such case - it allocates some chunks of block device for itself to keep SQLite files on it, including journals and temporary files, and all other space is available "pass-through" for a filesystem which metadta is kept in those SQLite files.

Actually, PBMFS could be made much more simpler than described here - like, no hardlinks, no directories, filenames of 15 characters length max... but I thought that someone in the future may want to make `/boot` on PBMFS and root on it's main FS, so now it supports small number of hardlinks and directories of up to 640 Kb, which should be enough for everyone :-)

For now, it (and later main FS) is planned to be implemented via FUSE, due to highly experimental nature of both FSes, but design was made keeping 

## Motivation

I've bought more disks into my home FreeBSD machine and was upset that ZFS can't convert single disks into `raidz` in-place - only `zmirror` was supported. I started to read [ZFS on-disk format](http://www.giis.co.in/Zfs_ondiskformat.pdf) and was very disappointed by some parts, especially hash tables in ZAP - not even a B-tree, not to mention B+trees! Also ZFS, despite having very useful features like snapshots, actually severely limits them: for example, snapshot is only for entire filesystem, and what if you had directory you later need to "convert" to filesytem? Next, speaking in terms of version control systems, snapshots do not have "branches" - clones are very limited, and so on. Given my interest in VCS, current filesystems could be viewed as very poor version control systems (and vice versa, Git architecture is comparable to FAT16), I had an idea to have both worlds combined - but it would be very hard task to implement from scratch and then keep extending.
Luckily, we have SQLite, the most tested and ubiquitous database in the world, which is designed to be able to work in constrained embedded environments - so it is in principle possible to put SQLite into kernel filesystem driver. And doing `ALTER TABLE ...` is much more simple than migrating on-disk data structures from traditional FS version N to version N+1.

But to play with file system metadata in SQLite, first you need a place where to store SQLite files, as described above, so PBMFS has to come first. As SQLite DB is limited to 256 Tb max (2^32 64 Kb pages), it may be possible that future design will split metadata to several SQLIte databases, to allow more space for very large partitions. Thus the name - if there are four 256 Tb files, it's already a petabyte.

# Spec * * * DRAFT * * *

PBMFS is conceptually a hybrid of traditional inode-based Unix file system and NTFS, but simplified. Almost **all** it's metadata - including directories and things like "indirect blocks" - is kept in a single MFT, MetaFile Table, which consists of fixed size numbered records (currently 4 Kb each).

Currently, very raw draft notes ("mind stream", almost not edited) of both PBMFS and future SQLite-based FS/VCS are available in `pb_draft_notes.txt` file (mostly in Russian) with not an open-source license (this will be changed later).

# To be continued
