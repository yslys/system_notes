# File Systems

### Lecture 1 - Intro
Every time you create a program on a 32-bit system, OS is going to assign 2^32
address space to such program.

+ What is a file?
    + A file is a sequence of bytes (with order).
    + A file is persistent on disk (even if power goes off).

+ How do files being stored on a disk?
    + A disk is divided into blocks.
    + A file is located on >= 1 disk blocks, not necessarily consecutive blocks.
    + E.g. a file with block number: 4 - 7 - 2 - 10 - 12.

+ How to retrieve a file located on separate disk blocks?
    + Option 1: linked list
        + For each block, we save a pointer to the next block, by which the order will persist.
        + Cons: 
            + Disk access will be sequential: slow for random access.
            + We want disk block size to be a power of 2, and we want the size of the content stored on such disk block to be a power of 2. Since we need to store a pointer in this case, it is hard to satisfy both conditions.

+ How to organize a file?
    + ```inode``` - index node.
        + A data structure to hook all the disk block together for a file.
        + It supports random access, and makes sure everything is a power of 2.

### Lecture 2 - inode & UFS
+ What does inode do?
    + inode maps the logical file into the physical disk block.

+ How large is an inode?
    + 256 Bytes.

+ What is the core data structure within an inode structure?
    + (For FreeBSD OS) 15 pointers.
        + 12 direct pointers pointing to disk blocks, representing the first 12 disk blocks of the file represented by inode.
        + 1 single indirection pointer, pointing to a pointer block - a disk block that stores only pointers to disk blocks.
        + 1 double indirection pointer, pointing to a block of single indirection pointers.
        + 1 triple indirection pointer, pointing to a block of double indirection pointers.
    + The order of the pointers determines the order of the file - starting from the first 12 pointers, then single indirection pointer, then double, then triple.

+ Direct access to any block of a file:
    + If you want to access to a particular position of a file, you would use a syscall called ```fseek()```, which requires a param called ```offset``` - the offset from the beginning of the file you want to access.
    + Such ```offset``` is divided by block size to determine which block will be accessed.

+ What happens when you ```open()``` a file?
    + 1st disk access:
        + The inode of the corresponding file will be cached from the hard disk to memory.
    + What happens when you ```fseek()``` with an offset?
        + If the offset lays in the first 12 direct pointers, then simply do another disk access to get the disk block.
        + If the offset lays in the single indirection pointer, then there will be 1 disk access to get the pointer block, and then another disk access for the data we are interested in.
        + If the offset lays in the double indirection pointer, then there will be two disk accesses to get the pointer block that contains the pointer to the disk block we want, and then another disk access.
        + If the offset lays in the triple indirection pointer, then (3 + 1) disk accesses.


+ Examples: Suppose each disk block is 4KB
    + If a file is less than 48KB, then using direct pointers would suffice.
    + If it is 64-bit architecture, then each pointer will be 8 bytes; then for each disk block, there will be 4KB/8 = 512 pointers. 

+ Extra info:
    + UFS2 (Unix FileSystem 2): 64-bit architecture, each inode is of size 256 bytes.
    + UFS1 (Unix FileSystem 1): 32-bit architecture, each inode is of size 128 bytes.

+ Something important to know about ```inode``` and file system:
    + inode is pretty efficient when handling small files.
    + inode also stores the order of the sequence of the bytes of the file.
    + For a file system, there are 5 different kinds of blocks (all stored on hard disk):
        + Super block
        + inode block
        + indirection (pointer) block
        + directory block
        + data block

Below is the structure of the ```ufs2_dinode```:
```
125 struct ufs2_dinode {
126     u_int16_t di_mode; /* 0: IFMT, permissions; see below. */
127     int16_t di_nlink; /* 2: File link count. */
128     u_int32_t di_uid; /* 4: File owner. */ 
129     u_int32_t di_gid; /* 8: File group. */ 
130     u_int32_t di_blksize; /* 12: Inode blocksize. */ 
131     u_int64_t di_size; /* 16: File byte count. */ 
132     u_int64_t di_blocks; /* 24: Bytes actually held. */ 
133     ufs_time_t di_atime; /* 32: Last access time. */ 
134     ufs_time_t di_mtime; /* 40: Last modified time. */ 
135     ufs_time_t di_ctime; /* 48: Last inode change time. */ 
136     ufs_time_t di_birthtime; /* 56: Inode creation time. */ 
137     int32_t di_mtimensec; /* 64: Last modified time. */ 
138     int32_t di_atimensec; /* 68: Last access time. */ 
139     int32_t di_ctimensec; /* 72: Last inode change time. */ 
140     int32_t di_birthnsec; /* 76: Inode creation time. */ 
141     int32_t di_gen; /* 80: Generation number. */ 
142     u_int32_t di_kernflags; /* 84: Kernel flags. */ 
143     u_int32_t di_flags; /* 88: Status flags (chflags). */ 
144     int32_t di_extsize; /* 92: External attributes block. */ 
145     ufs2_daddr_t di_extb[NXADDR];/* 96: External attributes block. */ 
146     ufs2_daddr_t di_db[NDADDR]; /* 112: Direct disk blocks. */ 
147     ufs2_daddr_t di_ib[NIADDR]; /* 208: Indirect disk blocks. */ 
148     int64_t di_spare[3]; /* 232: Reserved; currently unused */ 
149 }; 
```
+ The numbers in the comments: the offsets in bytes of the fields from the starting of the ```ufs2_dinode``` structure.
+ The **size of this inode structure is 256 bytes**.
+ How many inodes are there in one disk block (4KB)?
    + 4KB / 256bytes = 16 (inodes/disk block)
+ Which lines correspond to the direct blocks and indirect blocks?
    + Line 146 ```ufs2_daddr_t di_db[NDADDR];```: direct blocks.
        + ```NDADDR``` = (208-112)/8 = 12.
    + Line 147 ```ufs2_daddr_t di_ib[NIADDR];```: indirect blocks.
        + ```NIADDR``` = (232-208)/8 = 3.
+ Line 131 ```u_int64_t di_size;``` vs Line 132 ```u_int64_t di_blocks;```? 
    + This corresponds to **logical file size** and **physical file size**.
        + Logical file size: the size of the file.
        + Physical file size: the number of disk blocks to store this file. It not only contains the data block, but also the inode block, the indirection block.
        + See this program that seeks to a random position, and writes random bytes for 1000 times:
        ```
        #include <stdio.h>
        #include <stdlib.h>

        int main(void) {
            FILE *f1 = fopen("./sss.txt", "w");
            int i;

            for (i = 0; i < 1000; i++)
                {
                fseek(f1, rand(), SEEK_SET);
                fprintf(f1, "%d%d%d%d", rand(), rand(),
                        rand(), rand());
                if (i % 100 == 0) sleep(1);
                }
            fflush(f1);
        }
        ```
        + Logically, this file would be large, since for each seek & write, there will be a block associated to it. But the file will have lots of holes in it.
        + After compiling the code, if we execute ```$ du``` (disk usage), the output on my side is 28, meaning that there are 28 disk blocks for the current directory.
        + After running the program, execute ```$ du``` again, it shows 4076, meaning that 4076 disk blocks for this directory - subtracting the original 28, we get 4048 (~4KB) disk blocks for the newly written file.
        + How to determine the size of each **disk block**?
            + By searching on the internet, there are two ways:
                + w/ sudo: ```$ sudo tune2fs -l /dev/sda1 | grep -i 'block size'```
                + w/o sudo: ```$ stat -fc %s .```
            + In my case, I got 4096 bytes per disk block.
        + Therefore, the total file size on disk is ```~4KB disk blocks * 4KB/disk block```, which is 16MB.
        + The logical file size of the newly written file is - (by executing ```$ ls -al```) 2146220982 bytes (~2.1GB).
        + **Physical file size: ~16MB, and logical file size: ~2.1GB.**
    + Hence, in the above case, ```di_blocks``` is 4048 (bytes of blocks), ```di_blocks``` is 2146220982 (bytes).
    + Interesting applications:
        + When user opens a file (say, it's a movie with partially downloaded using bittorrent) and seek to a particular offset, OS will handle the rest. If for that offset, the file has not been downloaded, what OS will do is - instead of returning an error to the user, it will redirect that offset to the bittorrent client, and then bittorrent client will receive such request, and put such request at a higher priority to grab that piece, and send it back to OS. However, there will be some delay for the user to be able to watch that offset of the movie.
+ ```u_int64_t di_size;``` compared with ```u_int64_t di_blocks;```?
    + Can be equal, or greater than, or less than.
    + di_size is mostly less than di_block*blocksize

+ A nice property of the inode:
    + It is possible for us to use an inode represent a logical file.


+ **Slide 45**: A typical hard disk configuration, a.k.a file system. 
    + On top: a hard disk consists of several partitions.
    + In the middle: the file system associated with each partition.
        + The first block: boot block.
        + The second block: super block.
        + 3rd piece: the list of inode blocks.
        + 4th piece: directory blocks, data blocks and indirection blocks(within inodes).
    + On the bottom: the list of inode blocks.


+ How to recover a file?
    + **inode block**: the disk block that contains only the inodes. (4096 bytes) / (256 bytes/inode) = 16 inodes in each inode block.
    + What happens when removing a file:
        + Remove the entry to the inode from the directory.
        + The inode structure itself will not be replaced until you create a new file that requires an inode, which means, if your disk space is not full, then possibly your old inode is still there. Hence, as long as the old file has not been overwritten by new files, and as long as I can find the older version of the directory block that still has the inode number corresponding to the deleted file, then I will be able to access to every data block or indirect block of the old file.





### Lecture 3 - inode & directory & snapshot
File systems have to be:
+ Hierarchical (directory)
+ Efficient in time and space (inode)
+ Robust against all failures (soft update, fsck, snapshots - the three must work together)
+ Extensibility to new functionalities (vnode)

Below is the structure of the ```ufs2_dinode```:
```
125 struct ufs2_dinode {
126     u_int16_t di_mode; /* 0: IFMT, permissions; see below. */
127     int16_t di_nlink; /* 2: File link count. */
128     u_int32_t di_uid; /* 4: File owner. */ 
129     u_int32_t di_gid; /* 8: File group. */ 
130     u_int32_t di_blksize; /* 12: Inode blocksize. */ 
131     u_int64_t di_size; /* 16: File byte count. */ 
132     u_int64_t di_blocks; /* 24: Bytes actually held. */ 
133     ufs_time_t di_atime; /* 32: Last access time. */ 
134     ufs_time_t di_mtime; /* 40: Last modified time. */ 
135     ufs_time_t di_ctime; /* 48: Last inode change time. */ 
136     ufs_time_t di_birthtime; /* 56: Inode creation time. */ 
137     int32_t di_mtimensec; /* 64: Last modified time. */ 
138     int32_t di_atimensec; /* 68: Last access time. */ 
139     int32_t di_ctimensec; /* 72: Last inode change time. */ 
140     int32_t di_birthnsec; /* 76: Inode creation time. */ 
141     int32_t di_gen; /* 80: Generation number. */ 
142     u_int32_t di_kernflags; /* 84: Kernel flags. */ 
143     u_int32_t di_flags; /* 88: Status flags (chflags). */ 
144     int32_t di_extsize; /* 92: External attributes block. */ 
145     ufs2_daddr_t di_extb[NXADDR];/* 96: External attributes block. */ 
146     ufs2_daddr_t di_db[NDADDR]; /* 112: Direct disk blocks. */ 
147     ufs2_daddr_t di_ib[NIADDR]; /* 208: Indirect disk blocks. */ 
148     int64_t di_spare[3]; /* 232: Reserved; currently unused */ 
149 }; 
```

+ Are the file names stored in ```inode```?
    + As we can see from the above inode structure, it does not contain the file name. Hence, the **filename is not the part of the file itself**. This is because a file is just a sequence of bytes stored (sequentailly or discretely) on the hard disk, and inode only points to those bytes, with some extra information shown below:
        + ```u_int16_t di_mode; /* 0: IFMT, permissions; see below. */```: access mode of this file.
        + ```u_int32_t di_uid; /* 4: File owner. */```: who created this file.
        + ```u_int32_t di_gid; /* 8: File group. */```: who can access this file.
        + Time stamps.
        + ```u_int32_t di_kernflags; /* 84: Kernel flags. */```: flags associated with this file.

+ Where are the file names stored?
    + On **Slide 45**, it shows the typical hard disk configuration, a.k.a file system. In the middle layer, we can notice directory blocks. Those blocks contain a lot of pointers to the file names and inodes.

+ Therefore, there must be a data structure that stores the file names, and that is called **Directory entries```.
```
struct dirent {
	ino_t d_ino; /* inode number */
	char  d_name[NAME_MAX+1]; /* the name of the file */
};
```

+ How are ```dirent``` structured organized and where are they stored?
    + All directory entries are stored in directory blocks.
    + In each directory block, it has many ```dirent``` structures.
    + In each ```dirent``` structure, there is an inode number, and the file name.
    + Then file system can use the ```d_ino``` **inode number to retrieve the actual inode** of the file. 

+ What is the order of execution when we search for a file, e.g. ```/dir/filename```?
    + We need to first go to inode #2 to get the inode of the root. 
        + (inode #2 is always corresponding to the root, so whenever we want to search for a file in a file system, we always start with inode #2. inode is the first block of the inode block list, right after the boot block and the super block).
    + 1.1) Use the inode to fetch the corresponding directory block from disk.
    + 1.2) Inside the directory block, find the corresponding ```dirent``` structure with ```dirent->d_name``` being "dir".
    + 1.3) Inside that ```dirent``` structure, it has ```dirent->d_ino``` - the inode number. Use that inode number to retrieve the corresponding inode structure.
    + 2.1) Use such inode to fetch the corresponding directory block (with name being "dir").
    + 2.2) Inside the directory block, find the corresponding ```dirent``` structure with ```dirent->d_name``` being "filename".
    + 2.3) Inside that ```dirent``` structure, it has ```dirent->d_ino``` - the inode number. Use that inode number to retrieve the corresponding inode structure.
    + 3) Now, we have the inode for the file ```/dir/filename```. Then we can access the actual content depending on the size of the file and the offset user passed in.


+ Example on slide 47 and 48:
    + Slide 47: a graph that represents a directory hiererachy starting from root.
    + Slide 48: 
        + left - the inode structures
        + right - directory blocks
            + 1st column: the name of the file in each dirent structure
            + 2nd column: the inode of the file in each dirent structure

+ **Hard link: two directory entries with different names, but the same inode number.**
    + This allows us to have multiple names of the same file.
    + This corresponds to ```ex``` and ```vi``` in the example.
+ ```int16_t di_nlink; /* 2: File link count. */``` - the **hard link**.
    + Now it is time to revise this field in the inode - it stores the # of directory entries associated with this inode number.

+ What happens if the hard link becomes 0?
    + It means of all the directory entries in all the directory blocks, none of them is associated with this inode anymore.
    + It means we can remove that file.

+ What does file system do when we remove that file?
    + inode block needs to be modified
    + directory block needs to be modified
    + free block bitmap needs to be modified
    + free inode bitmap needs to be modified

+ What if system crashes in the middle of removing a file (as mentioned above)?
    + UFS can guarantee that the file system is consistent regardless of how the system crashes.
    + UFS achieves it by three things -
        + soft update
        + snapshot
        + background FSCK


#### Snapshot of the FS
+ What is a snapshot? (Slide 67, Slide 84)
    + A snapshot is actually a **file** located on disk. There is an **inode** associated to it.
    + A snapshot stores - for each disk block, whether such block is 
        + 1) not used (1), or 
        + 2) not yet copied (0) or
        + 3) the pointer to the copied blocks.
    + A snapshot stores what the FS (disk blocks) was like when the last time a snapshot was took.

+ What will happen if user modified something in the FS (Slide 63)?
    + After you modify something in your FS, there will be inconsistency between what FS currently is (figure on the left), and the previous snapshot FS took (figure on the right). Such inconsistency is the red part on the figure.
    + FS will not take a snapshot as soon as user has modified a single file; instead, FS will take a snapshot in a **Copy-on-Write** manner - a snapshot will not be taken until the modified file will be overwritten.
    + With an active file system, if 99% of your files are not modified, then FS will not do any copy on those 99% of the files.

+ What happens when FS is taking a snapshot (from a higher level perspective)?
    + Freeze all activities related to FS.
    + Copy everything to "somewhere else".
    + Resume the activities.

+ How long will it take to take a snapshot for a FS with 2TB?
    + A few seconds.

+ Why could a snapshot can be taken in such short period of time?
    + **Copy-on-Write**: try to avoid writing to disk until the data is about to be overwritten. It is a lazy way of writing to disk (doing the copy).
    + FS will take a snapshot in the **Copy-on-Write** manner. In other words, with an active file system, if 99% of your files are not modified, then FS will not do any copy on those 99% of the files.
    + While taking a snapshot, FS only need to store simple info: whether a disk block is not used or not yet copied, or the pointer to the copied block. If a disk block has been copied already, then no need to take a snapshot on that disk block. If a disk block has not been copied since last time FS took a snapshot, then at the time when the disk block is to be overwritten, a snapshot must be taken before being overwritten.
    + That is why not all the disk block need to be "copied", which makes taking a snapshot much faster.

+ **How does FS take a snapshot?** (slide 84)
    + Go through **free blocks bitmap** (0: not free, 1: free), and for each bit in the bitmap, if it is free, then in the corresponding direct disk blocks (```di_db[NDADDR]```) or indirect disk blocks (```di_ib[NIADDR]```) in the (snapshot) inode structure, we will set it to be ```not used```. If the bit in the bitmap is not free, then we need to set the corresponding part in the (snapshot) inode structure to be ```not yet copied```.

+ **How will snapshot be changed when user wants to modify n disk blocks?** (slides 85, 86)
    + Note: a snapshot already exists. When user modifies a file in user space, it is not necessarily the disk block will be modified. Only when the FS actually tries to write to the disk, snapshot occurs. Hence, the following steps are done before actually modifying the disk blocks.
    + **Only the very first snapshot (i.e. no other older snapshots) is done by going through free block bitmap, and mark those to be "not used" or "not yet copied" one by one. For later snapshots, we need both the free block bitmap and the older snapshot file - and the snapshot file will update.**
    + 1) For the to-be-modified blocks that are "not used" in free blocks bitmap, then simply update the corresponding blocks within the snapshot inode to be "not copied". Done.
    + 2) For the disk blocks (say, n) that are "not copied", go through free blocks bitmap to find n free blocks, and make a copy of the n disk blocks to be modified.
    + 3) After making a copy, the direct disk blocks (```di_db[NDADDR]```) or indirect disk blocks (```di_ib[NIADDR]```) in the (snapshot) inode structure will NO longer store "not used" or "not yet copied", but store the pointer to the newly copied disk block.
    + **Followup question**: now, the pointers within snapshot inode contain pointers to the copied disk blocks. If The user tries to modify again, what will happen? Does it force FS to write to disk block? Or FS creates another new snapshot which treats the newer old file as the old file?
        + It will notice that the snapshot inode has the corresponding pointer being not yet copied, and will not write to that inode pointer. So everytime we want to recover to a snapshot inode, we can only recover to what it was when we first took the snapshot.
    + **Followup question**: What if two snapshots choosing the same disk block to store the copy?
        + That is absolutely possible, and in that way, we can save a lot of space.


+ Since we are using exactly 1 inode to store the snapshot, will the number of disk blocks exceed the max pointers an inode could handle?
    + No. When configuring the FS, it needs to make sure that max # pointers an inode can handle > # disk blocks.
    + Hence, for a very large partition, the disk block size might be 4KB or 16KB, etc.

+ Where is the snapshot kept?
    + Stored in the free blocks.

+ In a typical FreeBSD file system, there are 20 snapshot files allowed, which means, every time user tries to make a modification to the disk blocks that are "not yet copied", FS needs to check all those 20 snapshots to see if any of the snapshot file needs to do a copy (i.e. write to disk). Why?
    + In order to answer this question, we first need to understand what snapshots are used for. Snapshots stores various versions of what the disk partition was like at the time such snapshot was taken. Given a snapshot, we are able to recover the disk partition to be exactly what is described by the snapshot.
    + Hence, no matter what application tries to modify the disk block, such modification will be stored by every snapshot, so that for each snapshot, it stores this particular modification so that FS of this disk partition will be able to recover to any of the snapshots given this single modification.

So we save some time everytime when we create a new snapshot, but we pay a bit more when we actively modify the disk block.

If we take a snapshot, the next snapshot will remember the previous snapshot.


Example of creating a snapshot:
```
$ mkdir /backups/usr/noon

## -o snapshot: the option passed to mount command indicating creating a snapshot
## create a snapshot for the file /sur, and the snapshot is /usr/snap.noon
$ mount –u –o snapshot /usr/snap.noon /usr

## we can then use mdconfig to look at the snapshot
$ mdconfig –a –t vnode –u 0 –f /usr/snap.noon
$ mount –r /dev/md0 /backups/usr/noon

/* do whatever you want to test it */

$ umount /backups/usr/noon
$ mdconfig –d –u 0
$ rm –f /usr/snap.noon

```
In the example on Slide 73 - 79: when we create a snapshot /usr/snap.noon, the logical file size of the snapshot is 16GB and the number of physical disk blocks are 9136 bytes (by executing ```$ du```). Note that for every snapshot file, the # of physical disk blocks are roughly the same. 

After that, we do a make clean, to remove lots of files. Then, we check the size of the snapshot again, the logical size does not change, but the # of physical disk blocks are much larger - 97920 bytes. This is because when we removing those files by make clean, lots of the corresponding disk blocks to be removed are marked "not yet copied" in the snapshot inode. Hence, we need to make lots of copies so that we could recover to what it was before make clean.


+ Where is the snapshot source code?
    + ```/usr/src/sys/ufs/ffs/ffs_snapshot.c```
    + It is a kernel service.
    + Under the ```usr/src/sys/ufs``` directory, there are two directories: ffs and ufs. ffs contains files regarding to **inode**, and services (like snapshot) related to inode will be placed under ffs directory. ufs contains the files for **directory (dirent)** and other related services.

### Lecture 4 - background FSCK (FS consistency check), and soft update
When a FS crashed, we can recover it by taking the snapshot. But if the total time to recover (by merely using snapshot) takes quite a long time, we want to find a way to reduce it. That is why **background FSCK** is introduced so that while FS is recovering, users are allowed to use the FS immediately. Hence, **background FSCK** is to shorten the time for the FS to be available after a crash.

When FS crashed, and you want to use the machine again, after you turn the machine on, before user is able to use the FS, a snapshot is required to be taken. Then, background FSCK starts, while user is able to use FS at the same time.


#### FSCK (Note: NOT background FSCK)
+ Inconsistency problem:
    + In order to achieve higher performance in disk accesses, OS maintains a buffer cache so that disk blocks can be cached to memory, or even be cached to cache (one level higher than memory). In other words, there might be several copies of the same disk block: 
        + 1) on disk itself
        + 2) in memory
        + 3) in cache
    + So when user modifies a disk block, it first reveals in the disk block in cache, then the disk block in memory, but there must be inconsistency between the disk block in memory and on disk.
    + Such inconsistency problem will persist until we flush such disk block to disk.
    + Not only the disk block that contains data will cause inconsistency problems, but also those corresponding data structures like inode block, directory block:
        + inode contains a timestamp to be modified.
        + Some other correlated data structures needs to be modified as well.
    + Hence, whenever user tries to modify a single file, all the corresponding disk blocks (e.g. data block, inode block, directory block) in memory will be placed in the buffer cache, and all those blocks will be different from the disk blocks on disk.

+ How could we maintain FS consistency?
    + The core idea: **the ordering of updates from buffer cache to disk is critical**.
        + e.g. If the directory block is written back before inode and the system crashes, the directory structure will be inconsistent.
    + 1) Less efficient approach, and it still does not 100% guarantee consistency after crash: 
        + Use unbuffered I/O when writing inodes or pointer blocks.
        + Use buffered I/O for other writes and force sync every 30 seconds.
    + 2) Detect and Fix - what most FS does:
        + Detect the inconsistency.
        + Fix them according to the rules.
        + Running FSCK.

Below describes how **FSCK detects the inconsistency** in blocks and directory entries:

+ Two ways of consistency checks: block consistency and file consistency
    + Block consistency check (what FSCK does - really expensive, but **NOT background FSCK**)
        + There are two core data structures:
            + **Block-in-use table**: an array of integers
                + This table is only constructed during FSCK - a large amount of time.
                + 1:block is in-use, 0:block is not in-use.

            + **Free block bitmap**: FS needs to know: on hard disk which block is free to use
                + This bitmap is maintained all the time.
                + 1:free, 0:not free.
        + How to construct the **Block-in-use table**?
            + For a disk block, if there is an inode, in which there is a pointer block that stores the pointer to that particular disk block, then it means such disk block is in use.
            + Hence, we need to examine every pointer block in every single inode. 
                + If for a single disk block, if there is one a pointer block pointing to it, then such disk block will be incremented by 1. 
                + If none of the pointer blocks in all inodes points to a disk block, then such disk block should be marked as not in-use, or free (0).
            + **Ideally, if the FS is consistent, then the values in Block-in-use array should be 0 or 1, since there should be at most 1 pointer block in an inode pointing to an in-used block. There should not be another pointer pointing to the same block.** 
        + If FS is consistent, then the Block-in-use table should be equal to the reverse of the Free block bitmap (0->1, 1->0), for instance:
            + 0 1 1 1 0 0 0 1 0 (Block-in-use table)
            + 1 0 0 0 1 1 1 0 1 (Free block bitmap)
        + If FS is inconsistent, for instance, the last three bits:
            + 0 1 1 1 0 0 0 1 0 **0** **1** **2** (Block-in-use table)
            + 1 0 0 0 1 1 1 0 1 **0** **1** **0**(Free block bitmap)
        + Let's consider the above three pairs of bits ```(block-in-use table, free block bitmap)``` for inconsistency one by one:
            + (0, 0)
                + There is no pointer in inode pointing to that disk block but that disk block is not free.
                + This is not that danger since no inode is pointing to that disk block.
            + (1, 1): 
                + there is a pointer in inode pointing to that disk block but that disk block is free.
                + This is very danger, since that means there is an inode using that disk block, but from the OS's perspective, it will regard such disk block as free. Hence, that disk block might be assigned to other files later which invalidates the previous inode.
                + **Soft update**: prevents this case from happening but allows (0, 0) to happen.
            + (2, 0): there are two pointers in inode pointing to that disk block and that disk block is not free.

    + File consistency check (directory structure consistency check)
        + In each inode, it has ```di_nlink``` field to indicate how many hard links there are for this inode, or how many directory entries having the same inode number.
        + In this case, we are going to build another table similar to Block-in-use table, which is also an integer array, with each element of the array being the **#of directory entries pointing to this inode**.
            + Starting from the root of the FS (inode #2), go through every ```dirent``` structure to find each corresponding inode number, and increment by 1. So if there are n directory entries having the same inode number x, then for the corresponding position x in the array, it will have a value of n.
            + We refer to each element in this array as **D**.
        + We also need another table that stores **```di_nlink``` field of each inode**.
            + Traverse all the inodes, and put ```di_nlink``` field in the array.
            + We refer to each element in this array as **L**.
        + Comparison between **D** and **L**:
            + D == L: files are consistent.
            + D < L: inconsistent.
                + This is less dangerous - just wasting some disk space.
                + Consider the following simpler case:
                    + di_nlink == 2, but only 1 dirent pointing to this inode.
                    + When the dirent removes the file, di_nlink field in the inode will be 1. So all the disk blocks corresponding to this inode will still be there, and will not be marked as free in Free block bitmap until "Garbage Collection" is executed. 
            + D > L: inconsistent.
                + This is very dangerous.
                + Consider the following simpler case:
                    + di_nlink == 1, but 2 dirents pointing to this inode.
                    + If one of the two dirents tries to remove the file corresponding to this inode, then the di_nlink field will be 0, indicating no dirent is associated with this inode number, hence this inode is marked as free, which might be overwritten by other applications. But the dirent that did not remove the file will still consider that the file is still there, so when it tries to access the file, it may not be able to access the correct file as it was.

+ Metadata operations
    + What is metadata?
        + The data blocks are not the metadata. The inode, Free block bitmap, Free inode bitmap, directory entry, etc. are all metadata.
    + Metadata operations modify the structure of the FS.
        + Creating, deleting or renaming files, directories or special files.
    + Metadata is actually maintaining the integrity of the FS.
        + **If you lost all the metadata, you might lose the whole FS.**
        + It is ok to lose some of the data blocks since we can use metadata to recover the FS to what it was when taking a snapshot.

Hence, metadata is extremely important and we do not want to lose it. We need a way to guarantee the integrity of the metadata.

+ How to guarantee the integrity of the metadata?
    + FFS uses synchronous writes for metadata, i.e. every time metadata is modified, it must be written back to the disk immediately. 
    + These writes will be blocking, i.e. nothing else can be done while the metadata is being written to the disk.
    + But using synchronous writes will largely impact the performance. The cost of metadata updates will be high.
    + For data blocks, the writes might be cached and flushed later.

From the above, we know that blocking writes of the metadata will strongly impact the performance, can we find out a way that we use less blocking writes (synchronous writes) but still maintains metadata integrity? Yes, by using **Soft Updates**.

#### Snapshot, Soft update, and background FSCK
Before diving into Soft update, we first need to figure out the relationship between the three.

+ Snapshot: gets a consistent global view of the whole FS
+ Background FSCK: garbage collect the lost ones
    + Background FSCK is - take a snapshot of FS, then run background FSCK against the snapshot whenever OS schedules it to run.
    + Foreground FSCK is slow.
+ Soft update: prevents Block inconsisty with (1, 1), and D > L
    + This makes sure that FS is safe to use while background FSCK is running.
    + Although this does not prevent garbage generated, garbage collection will be handled by background FSCK.

#### Soft Updates
Soft updates allows us to perform "dedalyed write" on metadata to the disk. But for the cached metadata, we need to make sure to **follow a particular order of writing those cached metadata back to disk**.

On deleting a file:

+ When deleting a file, we are deleting two things in order to let deletion complete.
    + The copy of disk block in memory
    + The disk block in disk

+ When deleting a file, what metadata needs to be modified (i.e. write to disk)?
    + ```dirent``` - the directory entry corresponding to the file removed should be deleted.
    + ```inode``` - decrement di_nlink.
    + Free block bitmap - for the free disk block.
    + Free inode bitmap - for the free inode.
+ Suppose the above 4 data structures are located on 4 different disk blocks,
    + In memory, there will be 4 disk blocks (pages) that are dirty.
    + Then needs to write the dirty blocks to disk.

+ What is the order of the 4 writes so that in between the 4 writes, no matter when a crash happens we can guarantee that 1) no block inconsisty with (1, 1) and no D > L?
    + If I write the inode block to disk before writing the Free inode bitmap, and system crashes, then the corresponding part of the Free inode bitmap will be still 0 (not free), this is acceptable since garbage collection in background FSCK will be able to handle it.
    + If I write the inode block to disk before writing the directory block, and system crashes, then in the directory block, it will consider that the file still exists. However, the inode block has already been modified with ```di_nlink``` being 0. Although free inode bitmap has not updated, background FSCK will be able to notice it and mark such inode to be free. Then it will be very danger since if later such inode has been assigned to another file, the directory entry will never be able to get the old file.
    + The correct order is as follows:
        + 1) Directory block
        + 2) inode block
        + 3) 4) free block bit map, and free inode bitmap

On creating a file, the order is going to be reversed. The goal is still to prevent 1) Block inconsisty with (1, 1), and 2) D > L
+ 1) Write the inode bitmap
+ 2) Write the inode block
+ 3) Write the directory block

On renaming a file (deleting and creating), both the directory block and inode block will be modified. The order is:
+ 1) increment the inode di_nlink by 1 (+1)
+ 2) add a new name in dirent
+ 3) remove the old name in dirent
+ 4) decrement the inode di_nlink by 1 (-1)

If the file is in the same directory, then only modifying the dirent name would suffice, but if in a different directory, then needs to follow the above 4 steps.

**Rules for soft update**:
+ For creating a file: never point to a structure before it has been initialized. So the data blocks must be written to disk first, before we can start writing the inode to disk. Similarly, we must first write the inode to disk, then we can write the corresponding directory entry to disk.
+ For deleting a file: never reuse a resource before nullifying all previous pointers to it. So when we delete a file, we first need to write the directory entry to disk first, then update the inode structure in disk.
+ For renaming (deleting and creating) a file: never reset the old pointer to a live resource before the new pointer has been set.




### Lecture 5 - Circular Dependency in Soft Updates & Journaling
https://web.stanford.edu/~ouster/cgi-bin/cs140-winter13/lecture.php?topic=recovery

Consistency conditions:
+ L >= D
    + To avoid D > L
    + Deletion: first dirent then inode (order of writing to disk)
    + Creation: first inode then dirent
+ (block_in_use_table\[B\], free_block_bitmap\[B\]) cannot be (1, 1)
    + (1, 0), (0, 1) are normal
    + (0, 0) is acceptable - waste some space, needs garbage collection
    + Deletion: (1, 0) -> (0, 1) - i.e. (1, 0) ~ (0, 0) ~ (0, 1)
    + Creation: (0, 1) -> (1, 0) - i.e. (0, 1) ~ (0, 0) ~ (1, 0)

Dependency relationships:
+ Problem setup: two blocks dirty blocks (dir block and inode block), we delete file "foo" and create a new file "bar".
    + Inside dir block: foo, and "NEW bar"
    + Inside inode block: inode-foo, and inode-NEW-bar
    + The two blocks are now in memory, with modified values (already deleted foo and inode-foo, created NEW bar and inode-NEW-bar), but not yet written to disk.

+ What is the order to write to disk?
    + For deleting foo, the order should be: dirent block then inode block.
    + For creating bar, the order should be: inode block then dirent block.
    + So there will be a conflict since there is a circular dependency.

+ **How to solve the circular dependency issue**?
    + Whenever there is a loop, we need to break that loop. That is called "roll back".
    + Consider the example on Slide 144:
    + 1st row: where circular dependency occurs
        + inode #4 and dirent A are the newly created in memory.
        + inode #5 and dirent B have been deleted in memory.
    + In order to solve the circular dependency, we need to waste one more disk block write. What we need to do is to roll back one of the two operations (creation and deletion) that causes the conflict. Suppose we choose to roll back the creation.
    + 2nd row: roll back creation (only deletion is done)
        + Since we are doing deletion only, we first need to write directory block to disk. But before that, we need to make a copy of the directory block to store the dirent A.
        + We then write the directory block to disk, with dirent A being empty.
        + On the right of the 2nd row, the original dirent B has been removed, while dirent A is not written there.
    + Now, the circular dependency has been solved, and we need to do the creation. For creation, we need to first write the inode block, then the directory block, that is why in row 3, we are writing the inode block. Besides, for the deletion that we done previously, we have already written the directory block, so it is also the time to write the inode block. Hence, in this single write (inode block), we write both the inode we created and the inode we deleted.
    + Finally, on the 4th row, we need to copy what is was in dirent A to the directory block to be written, and then we write the directory block to disk, so that dirent A is written to disk.

+ In the above example, we roll back creation. Can we roll back deletion?
    + We always roll back creation. We do not roll back deletion.
    + Reason?
        + Before we roll back, we need to consider what we have in memory. Roll back creation (row 2) means we roll back the creation of the dirent A in disk. Since before creation, the corresponding dirent in disk is NULL, we can safely overwrite that part with an empty dirent.
        + However, if instead, we roll back deletion and do creation first, we first need to make a copy of inode #5, zero inode #5 in inode block, and then write only inode #4 to disk (overwrite the inode block in disk). However, that will also overwrite the inode #5 in disk. So we must go to disk first, make a copy of what inode #5 was, and then fill that old value to the inode block, then we can safely overwrite the inode block in disk. That requires another disk read.


#### Journaling
Core idea: **write-ahead logging**

+ Write-ahead logging ensures that the log is written to disk before any blocks containing data modified by the corresponding operations.
+ Either all or nothing: either all the dirty blocks are written to the hard disk, or none of them are written to the hard disk.
+ No need to worry about circular dependency, since all the corresponding blocks will be written at once.
+ In journaling, first write the dirty blocks to the log. Such log is a specific area in disk. Then, after those dirty blocks have been written to the log, the file system will then write the log (i.e. the dirty blocks) to where they should be written to.
+ When writing to the log, system crashes, then after rebooting the system, FS will notice that the log is half-completed, then nullify those blocks in log.
+ After completing the log, system crashes, then after reboothing, FS will notice that the log is completed, and write to disk from the beginning of the log.
+ Every block needs to be written twice.



+ Comparison between journaling and soft update in various application scenarios?
    + Journaling has a simpler design than soft updates, easier to maintain and develop.
    + If the application you are working on is important, and you do not want to lose all the data, then soft update would be better.
    + One advantage of soft update is "when you can start using the FS after a crash".
        + This is critical to real-time system, critical mission system.
        + Soft update: after the snapshot has been taken can user starts to use the FS. You can start using FS even without taking a snapshot (if we do not care about the wasted space caused by crash).
        + Journaling: if system crashes after the log has been completed, but not yet written to the corresponding parts of disk, then we need to wait until log has been fully written.


### Lecture 6 - vnode & virtual file system
+ The layers of file system in Linux (ordered from top to bottom)
    + VFS: vnode
    + UFS: dirent
    + FFS: inode

+ vnode is a layer of abstraction in file system, right above the UFS layer (dirent).
+ vnode provides flexibility that we can introduce new services. Hence, a lot of OS services today are implemented in the vnode layer.
+ Examples: 
    + virus scanning service
        + Using vnode
        + Adding a virtualization layer for checking whether there is some kind of malicious signature in the files.
    + crypto file system
        + Want to encrypt the files and store to hard disk.
        + Encryption and decryption are implemented in vnode.

Before diving into vnode, we first need to know where such structure is defined. The first structure we will see is ```struct proc```, the structure of a process.

+ ```sys/proc.h```
    + This header file contains the data structure to capture all the processes or threads in kernel.
    + We will look into ```struct proc``` - a process
    + 
    ```
        struct proc {
            LIST_ENTRY(proc) p_list;        /* (d) List of all processes. */
            TAILQ_HEAD(, ksegrp) p_ksegrps; /* (c)(kg_ksegrp) All KSEGs. */
            TAILQ_HEAD(, thread) p_threads; /* (j)(td_plist) Threads. (shortcut) */
            TAILQ_HEAD(, thread) p_suspended; /* (td_runq) Suspended threads. */
            struct ucred    *p_ucred;       /* (c) Process owner's identity. */
            struct filedesc *p_fd;          /* (b) Ptr to open files structure. */
            struct filedesc_to_leader *p_fdtol; /* (b) Ptr to tracking node */
                                            /* Accumulated stats for all threads? */
            struct pstats   *p_stats;       /* (b) Accounting/statistics (CPU). */
            struct plimit   *p_limit;       /* (c) Process limits. */
            struct vm_object *p_unused1;    /* (a) Former upages object */
            struct sigacts  *p_sigacts;     /* (x) Signal actions, state (CPU). */
    ```
    + ```TAILQ_HEAD(, ksegrp) p_ksegrps```: kse group - kernel schedulable entity group
        + A process can contain more than 1 threads.
        + When a thread is mapped to a hypervisor or hardware, it is assigned a KSE - kernel schedulable entity.
    + ```struct filedesc *p_fd```: pointer to open files structure
        + Represents all the files that current process has access to.

Now, let's take a look at the ```struct filedesc``` - the structure for file descriptor.

+ ```sys/filedesc.h```
    + 
    ```
        50 struct filedesc {
        51     struct  file **fd_ofiles; /* file structures for open files */
        52     char    *fd_ofileflags;         /* per-process open file flags */
        53     struct  vnode *fd_cdir;         /* current directory */
        54     struct  vnode *fd_rdir;         /* root directory */
        55     struct  vnode *fd_jdir;         /* jail root directory */
        56     int     fd_nfiles;              /* number of open files allocated */
        57     NDSLOTTYPE *fd_map;             /* bitmap of free fds */
        ...
    ```
    + ```struct file **fd_ofiles;```: this is the list of pointers to open files owned by this file descriptor structure.

Next, we need to go to check ```struct file```, which represents an open file.

+ ```sys/file.h```
    + 
    ```
        106 struct file {
        107     LIST_ENTRY(file) f_list;/* (fl) list of active files */
        108     short   f_type;         /* descriptor type */
        109     void    *f_data;        /* file descriptor specific data */
        110     u_int   f_flag;         /* see fcntl.h */
        111     struct mtx      *f_mtxp;        /* mutex to protect data */
        112     struct fileops *f_ops;  /* File operations */
        113     struct  ucred *f_cred;  /* credentials associated with descriptor */
        114     int     f_count;        /* (f) reference count */
        115     struct vnode *f_vnode;   /* NULL or applicable vnode */
        116 
        117     /* DFLAG_SEEKABLE specific fields */
        118     off_t   f_offset;
        ...
    ```
    + ```off_t f_offset;```: the offset of the start of the current file (```fseek()```).
    + ```struct vnode *f_vnode;```: this is the vnode we will be talking about.


```struct proc``` -> ```struct filedesc``` -> ```struct file``` -> ```struct vnode```.
+ ```sys/vnode.h```
    + 
    ```
        107 struct vnode {
        108     struct  mtx v_interlock;                /* lock for "i" things */
        109     u_long  v_iflag;                        /* i vnode flags (see below) */
        110     int     v_usecount;                     /* i ref count of users */
        111     long    v_numoutput;                    /* i writes in progress */
        112     struct thread *v_vxthread;              /* i thread owning VXLOCK */
        113     int     v_holdcnt;                      /* i page & buffer references */
        114     struct  buflists v_cleanblkhd;
        115     struct  buf      *v_cleanblkroot;
        116     int              v_cleanbufcnt;
        117     struct  buflists v_dirtyblkhd;
        118     struct  buf      *v_dirtyblkroot;
        119     int     v_dirtybufcnt;
        120     u_long  v_vflag;                        /* v vnode flags */
        121     int     v_writecount;                   /* v ref count of writers */
        122     struct  vm_object *v_object;
        123     daddr_t v_lastw;                        /* v last write (write cluster) */
        124     daddr_t v_cstart;                       /* v start block of cluster */
        125     daddr_t v_lasta;                        /* v last allocation (cluster) */
        126     int     v_clen;                         /* v length of current cluster */
        127     union {
        128         struct mount    *vu_mountedhere;
        129         struct socket   *vu_socket;
        130         struct {
        131             struct cdev     *vu_cdev;
        132             SLIST_ENTRY(vnode) vu_specnext;
        133         } vu_spec;
        134         struct fifoinfo *vu_fifoinfo;
        135     } v_un;
        136     TAILQ_ENTRY(vnode) v_freelist;          /* f vnode freelist */
        137     TAILQ_ENTRY(vnode) v_nmntvnodes;        /* m vnodes for mount point */
        138     LIST_ENTRY(vnode) v_synclist;           /* S dirty vnode list */
        139     enum    vtype v_type;
        140     const char *v_tag;                      /* u type of underlying data */
        141     void    *v_data;                        /* u private data for fs */
        142     struct  lock v_lock;                    /* u used if fs don't have one */
        143     struct  lock *v_vnlock;                 /* u pointer to vnode lock */
        144     vop_t   **v_op;                         /* u vnode operations vector */
        ...
    ```
    + Clean buffer & Dirty buffer: line 114 - 119
        + Clean buffer: the ```buf``` structure that reads in but not modified - the list of blocks accessed by ```read()```
        + Dirty buffer: the list of blocks modified.
    + line 128: ```struct mount *vu_mountedhere;```
        + ```$ df``` - 1st column: Filesystem, last column: Mounted on.
        + ```$ mount``` similar. 
        + The output entries represent the file systems (can be different disks, or partitions within a disk)
        + Hard link: you cannot have a hard link across different file systems - must be on the same FS.
        + Symbolic link: you can have symbolic links across different file systems.
    + line 141: ```void *v_data;```
        + This corresponds to vnode, which allows you to access the potentian low level of FS
        + This is used with two macros: converting between in-memory inode and in-memory vnode. (They have a one-to-one correspondence)
            + ```#define VTOI(vp) ((struct inode *)(vp)->v_data)```
                + ```vp``` is a pointer to an in-memory vnode structure, ```(vp)->v_data``` is of type ```void*```. Hence, we are obtaining an in-mem inode pointer from this macro.
            + ```#define ITOV(ip) ((ip)->i_vnode)```
                + ```ip``` is a pointer to an in-memory inode structure, ```(ip)->i_vnode``` is of type ```struct vnode*``` in ```ufs/ufs/inode.h```([source](https://unix.superglobalmegacorp.com/BSD4.4/newsrc/ufs/ufs/inode.h.html#:~:text=struct%09vnode%20*i_vnode)). Hence, we are obtaining an in-mem vnode pointer from this macro.



+ Figure on slide 180 (Kernel level detail of vnode)
    + For each process, there is a structure ```vm_map``` (virtual memory map) associated to it, which represents every virtual memory piece to allocate for this current process.
    + For ```vm_map```, it contains a linked list of ```struct vm_map_entry```.
    + ```struct vm_map_entry```: contains info related to kernel, heap, stack, etc. Need to look at the entries to decide where the memory is.
        + Each ```struct vm_map_entry``` points to a ```shadow object```, ```shadow object``` points to ```vm_page```.
    + Both ```struct vm_map_entry``` and ```shadow object``` can point to a ```vnode```.
        + There might be multiple ```struct vm_map_entry```'s for storing (e.g.) the kernel files, or heap, or stack, but all of the vm_map_entry's of the same type point to one vnode.
        
+ Different between vnode and inode
    + For each ```mount```, it has its own inode list, and the most important data structure for each ```mount``` is vnode.
    + Suppose 1M files in a mount, but only open 3 files, then there would be 3 vnodes; however, the number of inodes is 1M.
    + inode is persistent (w/ or w/o opening files), but # of vnodes is number of open files.
    + When you want to use a FS, you use a vnode to represent that FS as a directory. Then, in the subsequent accesses (like opening the files within a FS), more vnodes would be created and linked to the opened files.
    + vnode contains a set of member functions that allow user to use to access the file.

Vnode table entry:
```
                ------------------------
 ------        |      File Type         |
|      |        ------------------------
| File |       | Pointers to functions  |
| Table|  -->  |     for this file      |  -->  Disk Drive
| Entry|        ------------------------
|      |       |   inode information    |
 ------        |(socket info for server)|         
                ------------------------
                   vnode table entry
```

+ Why for each file we need the pointers to the functions for the file? In other words, why can't we have a more generic set of functions that handle all files?
    + Because different files have different ways of access. vnode leverages the idea of polymorphism in OOP, which allows you to call the same function name, but does different things.

File System:
```
 -----------        ---------      
|  Process  |      |Open File|    
| Descriptor|  ->  |  Entry  |  ->   vnode   ->     inode
 -----------        ---------                   ufs/ufs/dinode.h (inode on disk)     
sys/filedesc.h   sys/file.h       sys/vnode.h   ufs/ufs/inode.h  (inode in memory)
```

+ What is the difference between inode on disk and inode in memory?
    + When accessing a file, the first thing to do is to bring the inode from disk to memory.
    + The inode brought into memory itself is just one field in the in-mem inode. So there is some extra info stored in the in-mem inode, including the pointer to vnode, and pointer to dirty buffer, etc.


#### **VFS (virtual FS): the FS Switch**
Given multiple FS on a host machine, before VFS was introduced, it was hard to integrate various FS together, i.e. need to write different code for accessing different FS. Hence, when you want to install another FS, you need to modify Linux kernel first before you could start using the new FS.

```
                          User space
--------------------------------------------------------------
                syscall layer (file, uio, etc.)
--------------------------------------------------------------
network  |                   Virtual
protocal |                 File System
 stack   |----------------------------------------------------
(TCP/IP) |  NFS  |  UFS  |  LFS  |  *FS  |  etc.  |  ...
--------------------------------------------------------------
                       device drivers
```

With VFS, it serves as an interface between syscall layer and the actual file systems. VFS consists of a bunch of vnode, with each vnode containing a bunch of function pointers corresponding to different accessing methods for different FS.

Recall that in ```struct vnode``` it has a field - ```void *v_data;```.
- If such vnode corresponds to UFS, then ```v_data``` will point to the inode.
- If such vnode corresponds to NFS, then ```v_data``` will point to a socket.
- etc.

+ vnode operations and attributes (slide 197)
    + ```ufs/ufs/ufs_vnops.c

+ Network File System (NFS) (slide 201)
    + Client side and Server side.
    + Client side:
        + Below the VFS layer, there is an NFS client (serves as a proxy) that connects to another server that contains a hard disk (inode).
        + When user wants to mount a disk on another server, the user does not need to worry about whether that disk is local or remote.
    + Server side: where the real disk locates
        + Receives the request from the Client side's NFS client (the proxy)
        + Then pass the request through the VFS layer, and then to the disk (inode).

+ Can we have multiple layers of vnode? Yes.
    + Between VFS and the actual FS, can have another layer - FiST-produced file system.
    + FiST: stackable File system
    + No need to modify the kernel
    + Only need to load another layer of VFS module into the kernel.

+ Example: anti-virus FS
    + Adding a layer of VFS that scans the file/request before accessing the UFS layer on the bottom.
    + ```read()``` -> ```antiVirusFS_read()``` (then send to another virus checking program. If no virus, then) -> ```ext2_read()```.



### Lecture 7 - Distributed FS
When the host wants to mount a remote FS on another server, the remote FS must export the portion of the directory it allows the host to mount to host's FS. So the host can execute the following:
```
$ mount -t nfs server.yahoo.com:/export/usr /usr
```
Host mounts the remote FS to ```/usr``` (local) directory.

+ This provides the features - **Transparency and Location independence**
    + Network **transparency**, in its most general sense, refers to the ability of a protocol to transmit data over the network in a manner which is transparent (invisible) to those using the applications that are using the protocol. In this way, users of a particular application may access remote resources in the same manner in which they would access their own local resources.
    + **Transparency and Location independence** is similar.

+ Some other issues of Distributed FS:
    + Reliability and Crash recovery
    + Scalability and Efficiency
        + The number of servers or clients needed to deal with
        + There might be a large number of users trying to access the same file at the same time - bottleneck, and may cause the response time for a distributed FS to degrade.
    + Correctness and Consistency
        + Correctness: on-copy unix semantics - every modification to every byte of a file has to be immediately and permanently visible to every client. **But this is very hard in a distributed FS**
    + Security and Safety


#### NFS (SUN, 1985) - stateless
+ Brief intro:
    + NFS is an application layer protocol
        + NFS client can be implemented one layer below the virtual file system
        + NFS client can also be implemented in user mode (above the kernel layer)
    + Based on RPC (Remote Procedure Call) and XDR (Extended Data Representation, e.g. json)
        + Difference between RPC and json RPC:
            + RPC is using XDR to communicate with server (a more compact representation of the data, but less readable than json)
    + Stateless - **Server maintains no state of the client, i.e. server does not need to maintain the info of what the user has done, and what the offset of the file that user used to read/write**.
        + a READ on the server opens, seeks, reads, and closes
        + a WRITE is similar, but the buffer is flushed to disk before closing
    + Server crash: client continues to try until server reboots – no loss
    + Client crashes: client must rebuild its own state – no effect on server

+ **Stateless vs Stateful** (Slide 19 20)
    + Stateless (bottom figure): discussed above. The server is light-weight, and rely on the clinet to maintain the state of a file.
        + When client opens a file, or seek to an offset, it will not be sent to the server. When client actually reads the file, such request will be sent to the server.
    + Stateful (top figure): we want the server to maintain some info about a session corresponds to a particular client.
        + When client opens the file, the request is sent to server; if the file is not too big, then a chunk of the file will be returned to the client. Then when client wants to read, or stat, then client can process locally.
    + Scalability issue in Stateless design
        + For a stateless design (as early NFS), it suffers from when tons of users trying to read the files, since each read is a request, and the server needs to handle all those requests, which challenged the CPU capacity and network capacity.

+ NFS operations
    + Every operation is independent: server opens file for every operation
    + File identified by handle -- no state information retained by server
    + client maintains mount table, v-node, offset in file table etc.
    + List of operations in version 2:
        + NULL, GETATTR, SETATTR
        + LOOKUP, READLINK, READ
        + CREATE, WRITE, REMOVE, RENAME
        + LINK, SYMLINK
        + READIR, MKDIR, RMDIR
        + STATFS (get file system attributes)

+ Why there's no open/close operations?
    + The purpose of ```open``` is to keep a pointer (offset in the file) so that the following reads and writes know where to read/write. 
    + Hence, the offset is stored in client side and the offset is sent with each subsequent reads/writes to the server.

+ File handle - an unique identifier for a file on the server side
    + Why not use the absolute path?
        + Because the files on the server side might change its location to another directory. Then the full path has to be changed. With fhandle, regardless where the file is, client is still able to access the file.
    + How are fhandle assigned?
        + fhandle is the inode number.

+ Mounting a remote NFS on the client side
    + A vnode associated with the root of the mounted NFS would be created.
    + A file handle associated with the root of the mounted NFS would be provided to the client.
    + Subsequent vnodes would be created as client calls ```LOOKUP```. The vnodes are like a tree structure, representing the directory structure in the remote NFS.
    + Client calls ```LOOKUP(dirfhandle, filename)``` -> Server returns a file handle -> Client calls ```read(fhandle, offset, count)``` -> Server returns the data.


### Lecture 8 - Andrew File System & Atomic commit protocols

#### AFS (Andrew FS) - stateful
When opening the file on server, cache the whole file (when the file is not that large) on the client side, then the subsequent operations are all performed locally, which does not involve network. When closing the file, the whole file is transfered back to the server.

+ **Callback - a notification service** in version 2 and 3
    + 1) Client opens a file and gets a copy of the file.
    + 2) Client modifies the file and closes the file to send the modified copy to server.
    + 3) During the above time period (open, modify, writeback), if another client has modified the file and written back, then the server will inform the client that file has been modified and sent the newer copy to the client.
    + 4) It is the client's responsibility to check the diff of the file, and either merge the file or do something else, and then write back.

+ optimistic concurrency control (in Callback in AFS)
    + Lock: pessimistic - no two clients can access one same file at the same time.
    + Optimistic: assume the probability that the race condition will happen rarely. Hence, no lock mechanism.
    + This is based on the observation that at any time, both users trying to write to the same file is unlikely to happen.
    + Validation phase: check if the modified file sent from a client has a conflict with the current file. No matter how many clinets opened the file, there will always be 1 callback corresponding to the file located on AFS. Whoever sends the modified copy back will have to check if the callback is still there - 
        + if the callback is there, then no one has modified the file yet, then AFS will take that callback away, and safely update the file.
        + if the callback is gone, then someone else has modified the file, so AFS will notify the user, and AFS will create a new callback (no need if someone else has already created one since last modification time), get the newer version of the file, make modifications, and write to AFS again.
        + Question??
            + if time0, user0 creates a callback0, time1, user1 opens the file again, will the callback0 be overwritten?


+ Fault Tolerance in AFS
    + Client crashes
        + check for callback tokens first, if there's a callback associated with the client, then the client knows that the modified file was not written to AFS.
    + Server crashes (callback is gone)
        + Client can request an open of the file, create a callback, then compare the file (returned by the server) and the file locally.

+ Atomic Commit protocols (1 client w/ multiple servers)
    + One-phase commit protocol
        + Atomic commit protocols are widely used in database and distributed systems where there are a bunch of transactions.
        + In database scenario: 
            + A transaction may consist of a number of individual transactions which carry out processes such as reading and writing to a database. When the transaction is completed all the individual transactions must have successfully carried out their tasks. 
            + A one-phase commit protocol involves a coordinator periodically communicating with the servers that are carrying out each individual transaction in order to inform them whether to commit the transaction or abort it.
        + In distributed system scenario: ([link](https://www.tutorialspoint.com/distributed_dbms/distributed_dbms_commit_protocols.htm))
            + The client acts as a coordinator.
            + A client may breaks a transaction into several small transactions over multiple servers.
            + After each server has locally completed its transaction, it sends a "DONE" message to the client.
            + The servers wait for "Commit" or "Abort" message from the client. This waiting time is called **window of vulnerability**.
            + When the client receives "DONE" message from each server, it makes a decision to commit or abort. This is called the commit point. Then, the client sends this message to all the servers.
            + On receiving this message, a server either commits or aborts (depending on what the client says) and then sends an acknowledgement message to the client.
            + If one server notices the transaction fails (a deadlock or a crash on the server), such server does not have the right to abort the transaction - only the client can allow the server to abort the transaction.
    + Two-phase commit protocol:
        + In distributed system scenario:
            + Core: commit if all servers say commit. Abort if any of the servers says abort.
            + Client sends transactions to all servers, and let servers execute the transactions.
            + Two-phase commit only starts when we are ready to commit the transaction. The client sends the commit request to the transaction coordinator.
            + Phase 1: prepare phase
                + The transaction coordinator sends a "Prepare" message to the servers.
                + The servers (mostly databases) will have to write all the transactions to disk and checks any constraints to make sure the consistency is maintained. The servers also need to write an entry to their undo log and an entry to their redo log (in case they want to abort the transaction).
                + The servers vote on whether they still want to commit or not. If a server wants to commit, it sends a "Ready" message. If not, then sends a "Not Ready" message. This may happen when the server has conflicting concurrent transactions or there is a timeout.
            + Phase 2: commit/abort phase
                + If the coordinator has received "Ready" message from all the servers −
                    + The coordinator sends a "Global Commit" message to the servers.
                    + The servers apply the transaction and send a "Commit ACK" message to the coordinator, release all the locks.
                    + When the coordinator receives "Commit ACK" message from all the servers, it considers the transaction as committed.
                + If the coordinator has received the first "Not Ready" message from any server −
                    + The coordinator sends a "Global Abort" message to the servers.
                    + The servers abort the transaction and send a "Abort ACK" message to the coordinator, release all the locks.
                    + When the coordinator receives "Abort ACK" message from all the servers, it considers the transaction as aborted.
    + Crash: 
        + If after coordinator has sent "Prepare" to the servers, some servers crash, then the coordinator will timeout and abort all transactions.
        + What if the coordinator crashes?
            + Solution 1: coordinator writes its decision to disk, so after it recovers, can check what decision was made.
        + If coordinator crashes after sending the "Prepare", but before broadcasting the final decision, i.e. other nodes do not know how coordinator has decided, then all other nodes will get blocked until the coordinator reboots.
            + Case 1: some servers have not received "Prepare" message, so cannot send "Ready" or "Not ready" to the coordinator, so the coordinator will never receive from all servers, so coordinator can just send Global abort.
                + Those servers just abort.
            + Case 2: not all servers have received Global commit/abort
                + Those servers received can do either commit or abort; those servers have not received are in the WAIT stage, and after the timeout, can query other servers about whether they have done commit or abort - if they have committed, then it means the coordinator had sent a Global commit message to them, so the "not yet committed" server can do commit. Similarly for Global abort. 
            + Case 3: all servers have received Global commit/abort
                + No influence. The servers can just do commit/abort.
            + **Case 4 (the weak spot for two-phase commit):** if the coordinator receives all "Ready" from all servers, and only sent Global commit to one of the servers, and failed to send it to the rest two servers. So the one success server will do commit and release the lock, while the two failed servers will be in WAIT state (they have not received whether to commit or abort from the coordinator). If right after the success server does commit, such server went down, then the two WAITing server remains in WAIT state since it does not know what the other server's state is (maybe that server was in WAIT state before crash, maybe that server has done commit (in this case) before crash), until either the coordinator or the crashed server come back. So need to wait for undetermined time.

    + Three-phase commit protocol:
        + Phase 1: prepare phase
            + The transaction coordinator sends a "Prepare" message to the servers.
            + The servers (mostly databases) will have to write all the transactions to disk and checks any constraints to make sure the consistency is maintained. The servers also need to write an entry to their undo log and an entry to their redo log (in case they want to abort the transaction).
            + The servers vote on whether they still want to commit or not. If a server wants to commit, it sends a "Ready" message. If not, then sends a "Not Ready" message. This may happen when the server has conflicting concurrent transactions or there is a timeout.
        + Phase 2: prepare to commit
            + The coordinator broadcasts a "Pre prepare" message.
            + The servers vote OK in response.
        + Phase 3: commit/abort
    
    + Three-phase commit solves Case 4 of crash.
        + If all servers are in WAIT state (all willing to commit), and have all received the Pre-Commit message from the coordinator, then they will response with an ACK to the coordinator. After coordinator receives all ACKs from all servers, coordinator will broadcast "doCommit" to all servers. If there are three servers in total:
            + 2 of them receive "doCommit", then commit, and then crash.
            + 1 of them doesn't receive "doCommit", then it remains in Pre-Commit stage.

The three-phase commit protocol eliminates this problem by introducing the Prepared to commit state. If the coordinator fails before broadcasting preCommit messages, the cohort (group of servers) will unanimously agree that the operation was aborted. The coordinator will not send out a doCommit message until all cohort members have ACKed that they are Prepared to commit. This eliminates the possibility that any cohort member actually completed the transaction before all cohort members were aware of the decision to do so.

The pre-commit phase introduced above helps the system to recover when a participant or both the coordinator and a participant failed during the commit phase. When the recovery coordinator takes over after the coordinator failed during a commit phase of two-phase commit, the new pre-commit comes handy as follows: On querying participants, if it learns that some nodes are in commit phase then it assumes that the previous coordinator before crashing has made the decision to commit. Hence it can shepherd the protocol to commit. Similarly, if a participant says that it had not received a PrepareToCommit message, then the new coordinator can assume that the previous coordinator failed even before it completed the PrepareToCommit phase. Hence it can safely assume that no participant has committed the changes, and hence safely abort the transaction.


### Lecture 9 - Google FS & Optimistic Concurrency Control

#### Google FS
+ First generation (slide 51)
    + Master: stores inode info, directory into
    + multiple chunk servers: serves as disks

+ Write control and Data flow (slide 67)
    + thin arrow: control flow, thick arrow: data flow.
    + If writing to a chunk, needs also write two copies.
    + **Diverge the network traffic by splitting control flow and data flow**:
        + Control flow: the client communicates with Primary Replica
        + Data flow: the client communicates with Secondary Replica A
    + **Another important property:**
        + If there are multiple updates to the same file, those updates should be consistently applied. In other words, the secondary replicas should be the same as the primary replica.
    + Steps for write:
        + Client asks master for all replicas.
            + Client provides directory path and offset of the file to master.
        + Master replies. Client caches.
            + Master replies the corresponding file handle
        + Client pre-pushes data to all replicas.
            + This is based on "whichever is closer goes first" policy, regardless of whether the data is first pushed to the primary replica or not. Hence, if secondary replica B is closest, then it will receive the data first.
            + In this way, it will make sure that all replicas receive the data as quickly as possible - that is the goal.
        + After all replicas acknowledge, client sends write request to primary.
            + Here is where control flow continues.
            + Since the data pushed in consists of (very likely) > 1 mutations to the file, the primary needs to assign consecutive serial numbers to the mutations it receives (possibly from multiple clients so serialization is a must), then applies the mutation to its own local state in serial number order. 
        + Primary forwards write request to all replicas.
            + Each secondary replica applies mutations in the same serial number order assigned by the primary.
        + Secondaries signal completion to the primary.
        + Primary replies to client. Errors handled by retrying.
            + Any errors encountered at any of the replicas are reported to the client. In case of errors, the write may have succeeded at the primary and an arbitrary subset of the secondary replicas. (If it had failed at the primary, it would not have been assigned a serial number and forwarded.) The client request is considered to have failed, and the modified region is left in an inconsistent state. Our client code handles such errors by retrying the failed mutation. It will make a few attempts at steps (3) through (7) before falling backto a retry from the beginning of the write.
    + Two types of writes:
        + Write
            + Requires an offset within the file.
        + Record Append
            + The client appends a new record to the end of the chunk.
            + If multiple clients append at the same time, the order is not guaranteed, which means the file is not deterministic.
            + But applications are able to serialize the record appendings
                + If the appendings are timestamps, then application can read it, and change the order manually then write back.
                + So for those applications, append can be very efficient as it does not require locks.
    + File Region State after mutation
        + **defined** and **consistent**
            + consistent: all replica have the same data, but the mutations might not be applied in serial order.
            + defined: the mutations are applied to the file in the serial order.
            + Defined is stronger than consistent.
            + For banking systems, we want data to be defined since we care about the order of each transaction (mutation). For online photo storing, we want data to be consistent since we do not care about the order.
        + Serial + Write + Success: defined
        + Concurrent + Write + Success: consistent but undefined
        + Failure: inconsistent
        + Record Append (no matter serial or concurrent access): defined interspersed with inconsistent



### Comparison between NFS, AFS and GFS

+ On reading the newest copy
    + NFS: as long as we don’t cache...
    + AFS: the read callback might be broken...
    + GFS: we can check the master... 

+ On writing to the master copy
    + NFS: write through
    + AFS: you have to have the callback... (delay a bit)
    + GFS: we will update all three replicas, but what happen if a read occurs during the process we are updating the copies. (name space and file to chunks mapping)



### Pessimistic vs Optimistic
+ Pessimistic: using a lock
    + Simple locking:
        + Say three files a, b, c. Three clients trying to write to all three files.
        + Whoever is trying to access file x needs to acquire lock(x) first before writing to the file.
    + Two-phase locking:
        + Locking phase: acquire the locks.
        + Unlocking phase: release the locks.
        + Hence, typically - acquire locks at the beginning, then do transactions, then release the locks one by one.
        + In simple locking, you can acquire the lock for the same file multiple times, but in two-phase locking, if you have acquired the lock for one file, then after you have released the lock, you should not acquire the same lock again.
        + Two-phase locking guarantees consistency (not defined), but does not guarantee no deadlock (might cause deadlock).

+ Optimistic: used by both AFS and GFS

+ When to use optimistic approach?
    + The chance for certain bad things to occur is very small.
    + It is very expensive to pessimistically prevent the probability.
        + e.g. using lock will strongly impact performance
    + "Let it happen, and we try to detect and recover from that"

+ Optimistic - details
    + When multiple clients write to a GFS block, GFS will maintain a recording mechanism s.t. if the writes have no conflict, then all good; if it detects a conflict, then needs to resolve the conflict based on the recording.
        + e.g. AFS: the callback is a simple form of recording.
    + In other words, Optimistic allows inconsistency, and deal with inconsistency later.
    + But the inconsistency will NOT be noticed by the clients. So only after GFS has handled such inconsistency can the consistent file be read by the clients.


### Optimistic concurrency control (OCC)
Do a logging on every transaction program executes. After a period of time, enter the validation phase. If during such period of time, no conflict occurred, then enter the commit state (push to the disk and make it available to the clients). If there was conflict, then enter the abort state and invalidate the log. (This is similar to what AFS does)

+ Three phases in OCC:
    + Working phase
        + The coordinator records the "readset" and "writeset" of each transaction (logging)
        + The mutations (writeset) in the logging can be regarded as a shadow (temporary) copy which have not yet committed.
    + Validation phase
        + On closing the transaction, the coordinator validates the transaction (looks for conflicts)
        + If validate succeeded, the transaction can commit.
        + If validate failed, either the whole transaction or the conflict ones are aborted.
    + Update phase
        + If validated, the changes(mutations) are made persistent.
        + Read-only transactions can commit immediately after passing the validation phase.

+ How to validate?
    + When a transaction enters the validation phase, it is assigned a transaction number.
    + Recall that there are three phases in OCC. Two transactions T0, T1 (T0 starts before T1) may overlap.
        + (Draw a picture would help.)
        + If no overlap, then no conflict.
        + If T0 starts and then T1, and **there is some overlap in between**:
            + It means when T1 starts, T0 is in Working phase, or Validation phase, or Update phase.
            + If T0 consists of reads only, then no conflict.
            + If T1 reads something that has been written by earlier committed transactions, then there is a conflict.

Hence, if a later transaction overlaps with earlier transactions, and the later transaction reads something that has been written by earlier transactions, then the later transaction needs to abort.

+ Is there a way not to abort?
    + If the only intersection is ReadSet(T1) and WriteSet(T0).
    + Assume WriteSet(T1) does not intersect with ReadSet(T0).
    + Then it is safe to execute T1 first then T0 although T1 has a larger transaction number. Hence, no need to abort T1.

+ Distributed OCC
    + The above considers only on a single machine, here we consider how OCC works in distributed FS.
    + Setup: multiple coordinators trying to apply transactions on multiple servers.
    + Question: How does the transaction order conflict in each server can be solved?


### Lecture 10 - Priority Ceiling Protocol
+ Priority Inversion
    + Concept: an interference between two mechanisms using any kind of system that will handle concurrent activities.
    + 1st mechanism: Real-time priority scheduling
        + Given limited CPU resources, we assign priority to each of the tasks, and the tasks would be executed in an order based on their priority.
        + In freeBSD kernel 256 priority levels, 64 scheduling classes(queues) with each matching 4 priority levels, i.e. the highest 4 priority levels will be treated as having the same priority.
    + 2nd mechanism: Mutual exclusion
        + If a higher priority thread wants to enter a critical section held by a lower priority thread, the higher priority thread has to wait until the lower priority thread leaves the critical section.
        + This has a strong impact when parallel execution.
        + Need to make it as small as possible.
    + Priority Inversion
        + If thread1 has the highest priority, then thread2 then thread3.
        + If thread3 comes first, and enters the critical section, then thread1 comes.
        + Then thread1 will keep executing until it tries to enter the critical section since thread3 is holding the lock.
        + If at the time thread1 gets blocked, thread2 comes, then thread3 will keep holding the lock and cannot execute until thread2 finished.
        + In this case, thread1 is blocked by thread2, while thread2 has lower priority than thread1.
        + Things get worse if more threads with lower priority than thread1 but higher priority than thread3.
        + Thread1 - with the highest priority, cannot execute!!!
        + This is called priority inversion - a lower priority task is blocking a higher priority task due to mutual exclusion.

Is there a way to solve priority inversion problem?

+ Priority Inheritance
    + We can bump up the lower priority tasks to a higher priority level.
    + In previous case, since thread3 is blocking thread1 due to mutual exclusion, after thread1 gets blocked, even though thread2 comes, we execute thread3 instead of thread2. Hence, thread2 will not start executing until thread1 has finished.
    + Implementation:
        + For each semaphore, a list of blocked threads (willing to acquire such lock) are stored in the corresponding priority queue (waiting queue).
        + A thread T uses its assigned priority, unless it is in its critical section and is currently blocking some higher priority threads. In such case, thread T will inherit the highest dynamic priority of all the threads it blocks.
            + e.g. Ti is blocking T20, and Ti has higher priority than T20, then priority level for Ti is still i. If T9 comes into the priority queue, and Ti has lower priority than T9, then Ti will have priority level 9. 
        + If a thread Ti is holding two semaphores m1 and m2, (priority level: Tk > Tj > Ti), if Tj is blocked in m1's priority queue, and Tk is blocked in m2's priority queue, then Ti will have the highest priority level k.
        + Priority inheritance is transitive; that is, if thread Ti blocks Tj, and Tj blocks Tk (priority level: Tk > Tj > Ti), then Ti can inherit the priority level of Tk. Hence, there might be a chain of changes in priority levels of multiple threads.
    + Two potential problems:
        + Deadlock:
            + Two threads need to access a pair of shared resources (A, B) simultaneously. If (A, B) are accessed in opposite orders by each thread, then deadlock may occur.
            + Example:
                + Two threads, T1 > T2
                + T1: P(S1); P(S2); V(S2); V(S1);
                + T2: P(S2); P(S1); V(S1); V(S2);
                + If T2 starts first, and then T1 preempts before T2 does P(S1), then deadlock will occur.
        + Blocking chain:
            + The blocking duration is bounded by at most the sum of critical section time lengths, but that can be substantial.
            + If a lot of lower priority threads get bumped up to higher priority levels, it is still, to some extent, blocking other threads.
            + Example: (L - normal tasks without acquiring mutex, R - tasks needed in critical section, double R: acquire, then release)
                + T1: L R2 L R3 L R4 L ... L Rn L;
                + T2: L R2 R2;
                + T3: L R3 R3;
                + ...
                + Tn: L Rn Rn;
            + In the above example, it is possible that T1 (highest priority) got blocked for each thread with lower priority.

How to solve the deadlock and blocking chain problems?

+ Priority Ceiling Protocol (PCP)
    + The most common feature in fully-preemptive kernel like Linux.
    + Detail:
        + A higher priority thread can be blocked **at most once in its life time** by a lower priority thread.
        + Deadlocks can be avoided (not prevented). But for Priority Ceiling Emulation (PCE), it is deadlock prevention.
        + Transitive inheritance is prevented (no more complexity brought by transitive inheritance).
    + Deadlock:
        + Detection
        + Prevention
            + Follow a procedure and no deadlock will be possible.
            + e.g. everyone follows a specific semaphore acquiring order
        + Avoidance
            + Before acquiring the lock, need to first watch what others are doing, and then proceed.
        + In Priority Ceiling Protocol, if a mutex M is available, and thread T needs it, T needs to check before locking it. However, we do not know whether a higher priority thread will occur in the next few seconds.
    + What to check before acquiring a lock?
        + Priority ceiling - a value predefined for each mutex while the program is compiled. Question: in user mode, we initialize several mutices and threads, and while compiling, how priority ceiling values are determined?
        + Explainations:
            + The value of priority ceiling is the highest priority level of all threads waiting on this mutex.
            + For any newly coming thread with priority level x willing to acquire lock(i),
                + If lock(i) is already been held by another thread, either higher priority or lower priority than the newly coming thread, then the new thread just appends to the priority queue of lock(i).
                + **If lock(i) is not held by any other thread, then the new thread needs to make a decision: should I acquire lock(i) or not? In order to make the correct decision, needs to check every mutex that is locked, and check their priority ceiling values. If x <= the highest priority ceiling value, then the new thread CANNOT acquire the lock.**
            + What if current thread has priority level 2, willing to acquire an unlocked mutex, and for all the locked mutex, there is a mutex with priority ceiling value being 2, and such mutex is currently acquired by thread9?
                + Current thread cannot acquire such unlocked mutex, since it might cause deadlock.
                    + e.g. T2: P1 P2 V2 V1. P1 is the mutex currently acquired by thread9, with priority ceiling being 2. P2 is the mutex that has not been acquired by any thread. Since T2 checks all locked mutex and noticed that T2 is not higher than all priority ceilings, T2 cannot lock P2. Because now P1 is acquired by thread9, if T2 acquires P2 first, then for P1, it is T2 that being the next to acquire the lock. However, there will be a deadlock. 
                

PCP: can a thread acquires an unlocked mutex? Why can it only acquire a locked mutex?
PCP: how does this solve the example of blocking chain? If T1 has not acquired the lock, then in the priority queue of the corresponding mutex, there will not be T1 waiting there. So if another lower priority thread comes, it needs to check before acquiring a lock, however, T1 is not waiting there.
PCP: priority ceiling has two values right? One is the original value determined in compile time, the other is determined by the threads waiting on such mutex? So the value determined in compile time is the upperbound? If ceiling is 2, unlocked, then thread9 can acquire that lock, then priority ceiling will change to 9 or remains as 2? If then thread 3 comes, it will acquire the lock as well, since it could do that because thread3 is lower than 2, greater than 9, right? If thread2 comes, it cannot acquire the lock, why it may cause deadlock?

So thread3 can grab the mutex with ceiling being 2, but highest priority of thread being 9 because it makes sure that thread3 will be able to preempt after thread9 releases the lock?

What if thread3 grabs another mutex not acquired by thread9?



How is rollback performed? nullify the value in memory and write to disk? or it is done in disk only?
If nullifying the value in memory and write to disk, then for creation, roll back will also cause the data in memory to be lost and we cannot recover if the data is not in cache. The only way is to make a copy somewhere else.


Questions:
+ Is the following question described correctly? Is it what was discussed in lecture?
+ **How will snapshot be changed when user wants to modify n disk blocks?** (slides 85, 86)
    + Note: a snapshot already exists. When user modifies a file in user space, it is not necessarily the disk block will be modified. Only when the FS actually tries to write to the disk, snapshot occurs. Hence, the following steps are done before actually modifying the disk blocks.
    + **Only the very first snapshot (i.e. no other older snapshots) is done by going through free block bitmap, and mark those to be "not used" or "not yet copied" one by one. For later snapshots, we need both the free block bitmap and the older snapshot file - and the snapshot file will update.**
    + 1) For the to-be-modified blocks that are "not used" in free blocks bitmap, then simply update the corresponding blocks within the snapshot inode to be "not copied". Done.
    + 2) For the disk blocks (say, n) that are "not copied", go through free blocks bitmap to find n free blocks, and make a copy of the n disk blocks to be modified.
    + 3) After making a copy, the direct disk blocks (```di_db[NDADDR]```) or indirect disk blocks (```di_ib[NIADDR]```) in the (snapshot) inode structure will NO longer store "not used" or "not yet copied", but store the pointer to the newly copied disk block.
    + **Followup question**: now, the pointers within snapshot inode contain pointers to the copied disk blocks. If The user tries to modify again, what will happen? Does it force FS to write to disk block? Or FS creates another new snapshot which treats the newer old file as the old file?
    + **Followup question**: What if two snapshots choosing the same disk block to store the copy?

+ Is the following saying correct?
> Note: a snapshot already exists. When user modifies a file in user space, it is not necessarily the disk block will be modified. Only when the FS actually tries to write to the disk, snapshot occurs. Hence, the following steps are done before actually modifying the disk blocks.


What will happen if no extra disk blocks are available?
+ **Followup question**: What if two snapshots choosing the same disk block to store the copy? I remember in the example where we do make clean, all the snapshots should copy the removed files. Hence, I think the answer is probably - every snapshot should be updated sequentially, or when accessing the free block bitmap, needs to acquire a mutex. What if while updating the snapshots, some of the snapshots cannot find free disk blocks? That means those particular snapshots will no longer be able to recover to what it was when snapshots were taken.

+ Is the following answer to the question correct?
    + In a typical FreeBSD file system, there are 20 snapshot files allowed, which means, every time user tries to make a modification to the disk blocks that are "not yet copied", FS needs to check all those 20 snapshots to see if any of the snapshot file needs to do a copy (i.e. write to disk). Why?
        + In order to answer this question, we first need to understand what snapshots are used for. Snapshots stores various versions of what the disk partition was like at the time such snapshot was taken. Given a snapshot, we are able to recover the disk partition to be exactly what is described by the snapshot.
        + Hence, no matter what application tries to modify the disk block, such modification will be stored by every snapshot, so that for each snapshot, it stores this particular modification so that FS of this disk partition will be able to recover to any of the snapshots given this single modification.

+ Whenever a file is removed, the snapshot code inspects each of the blocks being freed and claims any that were in use at the time of the snapshot. Those blocks marked "not used" are returned to the free list.





What is the difference?
ln –s /usr/src/sys/sys/proc.h ppp.h
ln /usr/src/sys/sys/proc.h ppp.h



+ Hard link(di_nlink) && Symbolic link (shortcut).



What is the largest file an inode can handle?

Examples of the file systems: FAT32 (windows) - limitation of the file size is 4GB, NTFS, exFAT. So if you format your USB disk with FAT32, then the largest single file that you can store is 4GB regardless of how large your USB is. If instead, you format your USB disk with exFAT or NTFS, then you can break that file size limit.

hard link vs symbolic link?

How many disk blocks can a FS have?
264 or 232: Pointer (to blocks) size is 8/4 bytes.
How many levels of i-node indirection will be necessary to store a file of 2G (231) bytes? (I.e., 0, 1, 2 or 3)
12*210 + 28 * 210 + 28 *28 *2 10 + 28 * 28 *28 *2 10 >? 231 
What is the largest possible file size in i-node?
12*210 + 28 * 210 + 28 *28 *2 10 + 28 * 28 *28 *2 10
264 –1
232 * 210

What is the size of the i-node itself for a file of 10GB with only 512 MB downloaded?
