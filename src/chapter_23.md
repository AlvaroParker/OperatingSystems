# Files and Directories
Persisten storage, stores information permanently, unlike memory whose contents are lost when there is a power loss, persistent-storage device keeps such data intact.
In this chapter, we can see an overview of the filesystem API available with Unix file systems

## Files and Directories
Virtualization of storage has 2 key abstractions: 
1. Files: Linear array of bytes that you can read or write
	- Each file has a low level name (inode number)
	- The filesystem doesn't know about the structure of the file (if it's a pdf, image, document, etc), it simple stores the data persistently on disk and returns the expected data when requested
2. Directories (a.k.a Folders)
	- Each directory has a low level name (inode number)
	- It contains a list of (user-readable name, low-level name) pairs
	- An entry example of this list: ("foo", "10") where "foo" is the user-readable name and "10" is the low-level name

By placing directories on directories we build a directory tree, under these tree, all files and directories are stored. 

## The File System Interface
It provides ways to access, create, delete and modify files. 

## Creating files
We do this using the `open` system call with the `O_CREAT` flag:
```c
int fd = open("foo", O_CREAT | O_WRONLY | O_TRUN, S_IRUSR | S_IWUSR)
```
- We create the file if it doesn't exist (`O_CREATE`)
- And open it in write-only mode (`O_WRONLY`)
- If the file exists, we truncate it to zero (`O_TRUNC`)

The `open` system call returns a file descriptor, which is a number private per process used in UNIX to access files. We will use the file descriptor number to read or write the file. 

File descriptor are managed by the operating system, on a peer process basis, in fact on the `xv6-riscv` source code, we can see that file descriptors are on the `proc` structure on `kernel/proc.c`:
```C
struct proc {
	...
	struct file *ofile[NOFILE]; // Open files
	...
}
```
We keep an array of open files per process

## Reading and Writing Files
We use the system call `read` to read from a file. It takes as an argument a file descriptor `fd`, a buffer `buf` of at least size `count` and the `count` which is the number of bytes we want to read from the file.
```C
ssize_t read(int fd, void buf[.count], size_t count);
```
We can also write to a file by using the system call `write`  which takes a file descriptor `fd` as argument, as well as the content `buf` we want to write and the content length `count`
```C
ssize_t write(int fd, const void buf[.count], size_t count);
```

### Example `cat`
Let's see how the command `cat` handles opening and reading from file by using the `strace` command. We will read a file `foo` which has "hello\n" as content:
```bash
$ strace cat foo
...
open("foo", O_RDONLY | O_LARGEFILE) = 3
read(3, "hello\n", 4096)
write(1, "hello\n", 6)
hello
read(3, "", 4096)
close(3)
...
```
Here we open the file `foo` on read only mode (`O_RDONLY`) and to handle large file that exceed the max number of `off_t` type, we pass the `O_LARGEFILE` option, this call returns a file descriptor `3` which is the used to read from the file using the `read` system call. 

We then write the content to standard output which by convention is file descriptor number `1` by calling `write` with the file descriptor (`1` in this case), the content ("hello\n") and the content length (`6`)

## Read and Writing, But Not Sequentially
Sometime we will want to read and write from a specific offset of a file, we use the `lseek` system call to do so. 
```C
off_t lseek(int fd, off_t offset, int whence);
```
This system call takes a file descriptor `fd`, an `offset` and an int `whence` determines how the seek is performed, from the man page: 
```
If whence is SEEK_SET, the offset is set to offset bytes.
If whence is SEEK_CUR, the offset is set to its current
location plus offset bytes.
If whence is SEEK_END, the offset is set to the size of
the file plus offset bytes.
```
Part of the abstraction of an open file is what's the current offset. When we write to an open file, the offset will be modified accordingly, for example if we write 4 bytes, the offset will increase by 4 bytes, the offset *implicitly* updates.
An open file on `xv6` is defined as the following structure: 
```C
struct fike {
	int ref;
	char readable;
	char writable;
	struct inode *ip;
	uint off;
}
```
`ref` is a refence count, `readable` and `writable` are used to keep track if the current file is readable or writable, `inode` is the low-level name and `off` is the offset. 

## Shared File Table Entries: `fork()` and `dup()`
In many cases, mapping from file descriptor to an entry in the open file table is one-top-one, meaning for each file descriptor there's only one entry in the open file table. If another process reads the same file, it will have a different offsets because his file descriptor is pointing to a different entry on his open file table. 

There are exceptions to these case: 
 1. There are cases where an entry in the open file table is shared, with `fork` this happens. Parent creates a child process, and they both share the same open file table entry, when the process is created the reference count increases on the open file table entry. 
 2. Another case is when using the `dup` system call: Allows a process to create a new file descriptor that refers to the same underlying open file as an existing descriptor.

## Writing Immediately with `fsync`
`write` tells the file system "Please write this content to the desired file when you can", to force the dirty data (data that's on memory and has not yet been written) to disk we can use the `fsync(int fd)` system call, which takes a file descriptor as an argument. 
A simple example: 
```C
int fd = open("foo", O_CREAT | O_WRONLY | O_TRUNC, S_IRUSR | S_IWUSR);
assert(fd > -1)
int rc = write(fd, buffer, size);
assert(rc == size);
rc = fsync(fd);
assert(rc == 0);
```

## Renaming files
To rename a file `foo` to `bar` we can use to `mv` command: 
```bash
$ mv foo bar
```
If we use `strace` we see that `mv` calls the systems call `rename(char *old, char *new)`

## Getting information about files
Information about files is called metadata, too see such metadata we can use the systemcall `stat()` or `fstat()`. These take a pathname (or file descriptor) to a file and fill in a `stat` structure. 

A `stat` structure looks something like this: 
```C
struct stat {
	dev_t st_dev; // ID of device containing file
	ino_t st_ino; // inode number
	mode_t st_mode; // protection
	nlink_t st_nlink; // number of hard links
	uid_t st_uid; // user ID of owner
	gid_t st_gid; // group ID of owner
	dev_t st_rdev; // device ID (if special file)
	off_t st_size; // total size, in bytes
	blksize_t st_blksize; // blocksize for filesystem I/O
	blkcnt_t st_blocks; // number of blocks allocated
	time_t st_atime; // time of last access
	time_t st_mtime; // time of last modification
	time_t st_ctime; // time of last status change
}
```

These kind of information is usually stored by the file system in a structure called the **inode** 

## Removing Files
The `rm` command on UNIX executes the following system call: 
```C
unlink("foo")
```
## Making directories
We can use a set of system calls that allow us to make, read and delete directories. 
You can't never write to a directory directly, the format of the directory is file system metadata which can only be changed indirectly, by creating file, directories or other object types within it. 

To create a directory, we use `mkdir()`: 
```C
mkdir("foo", 0777)
```
Here `0777` are the permissions of the directory, and "foo" is the directory name, the directory created is considered empty. 

## Reading directories
Instead of reading the directory as a file, we use a set of system calls to read the content of a directory. 

- The `opendir()` , `readdir()` and `closedir()` are the most used ones. 
- Example of a simple `ls` like program: 

```C
int main(int argc, char *argv[]) {
	DIR *dp = opendir(".");
	assert(dp != NULL);
	struct dirent *d;
	while ((d = readdir(dp)) != NULL) {
		printf("%lu %s\n", (unsigned long) d->d_ino, d->d_name)	;
	closedir(dp);
	return 0;
	}
}
```

## Deleting Directories
We use `rmdir()` system call to remove a directory. The `rmdir` system calls require the directory to be empty before deleting it, else it will fail.

## Hard Links
We can make new entries on our file system call using the system call `link()`, which takes two arguments, an old path name and a new one, when you link an old path name to a new path name, you create a new way to refer to the same file. The command `ln` is used to do this. 
```bash
$ echo hello > file
$ cat file
hello
$ ln file file2
$ cat file2
hello
```
When we create a file we are dong 2 things, we are making a structure (inode) that will track virtually all relevant information about the file and we are linking a human-readable name to that file, and putting that link into a directory.

When we call `rm` we are calling system call `unlink` to that file, which check a reference count to that number and decreases the count by 1 value. When the reference count reaches zero, the file system also frees the inode and relate data blocks, and thus is truly deleted.

## Symbolic Links
To create this link we can use the `ln` but with the `-s` flag. 
```bash
$ echo hello > file
$ ln -s file file2
$ cat file2
hello
```
The difference between this links and hard-links is that symbolic links are a file itself of a different type. The content of a symbolic link is the path to the file it's pointing to, in the example, the content of `file2` would be `file` path. 
