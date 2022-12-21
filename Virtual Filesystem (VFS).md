# Virtual Filesystem (VFS)

## The Role of the VFS

VFS's main strength is providing a common interface to several kinds of filesystems.

### The Common File Model

The key idea behind the VFS consists of introducing a common file model capable of representing all supported filesystems.

The common file model consists of the following object types:

* **The inode (index node) object**

  This object usually corresponds to a `file control block` stored on disk. Each inode object is associated with an inode number, which uniquely identifies the file within the filesystem.

* **The file object**

  Stores information about the interaction between an open file and a process. This information exists only in kernel memory during the period when a process has the file open.

* **The dentry object**

  Stores information about the linking of a directory entry with the corresponding file. Each disk-based filesystem stores this information in its own particular way on disk.

#### Inodes object

The elements of an inode can be grouped into two classes:

1. Metadata to describe the file status; like access permissions or date of last change.
2. A pointer to data in which the actual file contents are held.

How the kernel goes directory hierarchy and find the inode of `/usr/bin/emacs`?

First it lookup the root inode, which represents the root directory `/`. The inode only contains data which includes the root root directory entries. Each entry consists of two elements.

1. The number of the inode, so the data of the next entry are located.
2. The name of the file or directory.

Then it scans all the entry to find the inode of the subdirectory `usr`. 