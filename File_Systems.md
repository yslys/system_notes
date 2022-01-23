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
    + **Followup question**: What if two snapshots choosing the same disk block to store the copy?


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


### Lecture 4 - background FSCK






























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

+ **Followup question**: now, the pointers within snapshot inode contain pointers to the copied disk blocks. If The user tries to modify again, what will happen? Does it force FS to write to disk block? Or FS creates another new snapshot which treats the newer old file as the old file?

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
