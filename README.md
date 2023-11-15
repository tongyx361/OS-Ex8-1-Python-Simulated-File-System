# OS-Ex8.1 Python 模拟文件系统 (Python-Simulated File System) Draft

> [课后练习](https://www.yuque.com/xyong-9fuoz/qczol5/uzf18vbnscar3hzi#lUmxq)
>
> “[vsfs.py](https://github.com/remzi-arpacidusseau/ostep-homework/blob/master/file-implementation/vsfs.py)”是用 python 脚本实现的一个简单的文件系统模拟器，可以模拟用户在进行文件操作过程中磁盘上文件系统的存储结构和内容的变化情况；“[README.md](https://github.com/remzi-arpacidusseau/ostep-homework/blob/master/file-implementation/README.md#overview)”是该文件系统模拟器的简要介绍。
> 请通过阅读“[README.md](https://github.com/remzi-arpacidusseau/ostep-homework/blob/master/file-implementation/README.md#overview)”，并分析和执行“[vsfs.py](https://github.com/remzi-arpacidusseau/ostep-homework/blob/master/file-implementation/vsfs.py)”，然后依据自己的兴趣和时间情况，选择完成如下部分或全部任务。
>
> 1. 描述该文件系统的存储结构；
> 2. 基于模拟器的执行过程跟踪分析，描述该文件系统中“读取指定文件最后 10 字节数据”时要访问的磁盘数据和访问顺序；
> 3. 基于模拟器的执行过程跟踪分析，描述该文件系统中“创建指定路径文件，并写入 10 字节数据”时要访问的磁盘数据和访问顺序；
> 4. 请改进该文件系统的存储结构和实现，成为一个支持崩溃一致性的文件系统。

## 1 该文件系统的存储结构

示例如下：

```
inode bitmap 11110000
inodes       [d a:0 r:6] [f a:1 r:1] [f a:-1 r:1] [d a:2 r:2] [] ...
data bitmap  11100000
data         [(.,0) (..,0) (y,1) (z,2) (f,3)] [u] [(.,3) (..,0)] [] ...
```

**Inode Bitmap**：位图，显示每个位置的 inode 是否被分配，1 表示分配，0 表示未分配。`11110000` 表示前 4 个 inode 已经被分配，而后 4 个还没有。
**Inodes**：列表/数组，每个元素是一个字典/结构体，存储内存中的文件信息。每个 inode 包含三个字段/成员：

1. 文件类型（文件或目录）：`d` 表示目录，`f` 表示常规文件；
2. 该文件的数据块地址 **（注意：该文件系统中一个 inode 只对应一个 data block，故只需用一个索引描述）** ：非零数字表示在 `data` 中的索引，`-1` 表示未分配 data block；
3. 引用计数：至少为 `1`，当减少为 `0` 时 inode 会被删除。
   **Data Bitmap**：位图，显示每个位置的磁盘数据块是否被分配，与 **Inode Bitmap** 类似。
   **Data**：列表/数组，每个元素是一个列表，存储实际的文件数据或目录数据。

- 对于目录，每个元素列表包含条目 `(name，inode_index)`。
- 对于文件，每个元素列表包含文件内容，即一个字符序列。

## 2 该文件系统中“读取指定文件最后 10 字节数据”时要访问的磁盘数据和访问顺序

该文件系统不提供 `read()` 操作，故“读取”任务只能通过直接查看 `data` 部分的内容完成。下面描述步骤：

假设要读取文件 `/a/b/c/…/z` 的最后 10 字节数据

1. 从 `data[0]`，即根目录项开始，找到 `(a,i_a)`，其中 `i_a` 为 `inodes` 的索引。
2. 找到 `inodes[i_a]`，其中内容为 `a` 的 inode 信息。此处 `a` 是目录，故内容应该为 `[d a:a_a r:r_a]`，其中
   1. `a_a` 为 `a` 在 `data` 对应的数据块的索引，此处 `a` 必然有对应数据块，故 `a_a` 不可能为 `-1`；
   2. `r_a` 为该 inode 的引用数量，对次数查找文件没有帮助，暂时忽略。
3. 找到 `data[a_a]`，即存储 `a` 数据的数据块，重复 1~3，知道找到最终文件 `z` 对应的数据块 `data[a_z]`。
4. 查看 `data[a_z]` 的内容，如果 `len(data[a_z] < 10`，则报错，否则读取最后 10 字节的数据即可。

## 3 该文件系统中“创建指定路径文件，并写入 10 字节数据”时要访问的磁盘数据和访问顺序

假设要创建文件 `/a/b/c/…/z` 并写入 10 字节数据

1. 创建 `creat("/a/b/c/…/z");`：仿照 3.1~3.3 找到最终文件 `z` 对应的 inode 索引 `i_z`，
   1. 检查 `ibitmap[i_z]` 若对应位 `ibitmap[i_z]` 已经为 1，则已经被创建，报错，否则将 `ibitmap[i_z]` 置为 1；
   2. 创建 `inodes[i_z]`，此时内容应为 `[f a:-1 r:1]`。
2. 写入 `fd=open("/y", O_WRONLY|O_APPEND); write(fd, buf, BLOCKSIZE); close(fd);`，
   1. 检查 `dbitmap`，若已经满了，则报错，否则分配一个数据块索引 `a_z`，并更新 `dbitmap[a_z]` 为 1，此时 `data[a_z]` 为空；
   2. 写入 `data[a_z]`，如果文件数据块已经满了，则报错，否则写入数据即可，此时 `len(data[a_z])` 为 `10`。

### 相关代码

```Python
class bitmap:
    def __init__(self, size):
        self.size = size
        self.bmap = []
        for num in range(size):
            self.bmap.append(0)
        self.numAllocated = 0

    def alloc(self):
        for num in range(len(self.bmap)):
            if self.bmap[num] == 0:
                self.bmap[num] = 1
                self.numAllocated += 1
                return num
        return -1

class fs:
    def inodeAlloc(self):
        return self.ibitmap.alloc()

    def dataAlloc(self):
        return self.dbitmap.alloc()

    def createFile(self, parent, newfile, ftype):
        # find info about parent
        parentInum = self.nameToInum[parent]

        # is there room in the parent directory?
        pblock = self.inodes[parentInum].getAddr()
        if self.data[pblock].getFreeEntries() <= 0:
            dprint('*** createFile failed: no room in parent directory ***')
            return -1

        # have to make sure file name is unique
        block = self.inodes[parentInum].getAddr()
        # print('is %s in directory %d' % (newfile, block))
        if self.data[block].dirEntryExists(newfile):
            dprint('*** createFile failed: not a unique name ***')
            return -1

        # find free inode
        inum = self.inodeAlloc()
        if inum == -1:
            dprint('*** createFile failed: no inodes left ***')
            return -1

        # if a directory, have to allocate directory block for basic (., ..) info
        fblock = -1
        if ftype == 'd':
            refCnt = 2
            fblock = self.dataAlloc()
            if fblock == -1:
                dprint('*** createFile failed: no data blocks left ***')
                self.inodeFree(inum)
                return -1
            else:
                self.data[fblock].setType('d')
                self.data[fblock].addDirEntry('.',  inum)
                self.data[fblock].addDirEntry('..', parentInum)
        else:
            refCnt = 1

        # now ok to init inode properly
        self.inodes[inum].setAll(ftype, fblock, refCnt)

        # inc parent ref count if this is a directory
        if ftype == 'd':
            self.inodes[parentInum].incRefCnt()

        # and add to directory of parent
        self.data[pblock].addDirEntry(newfile, inum)
        return inum

    def writeFile(self, tfile, data):
        inum = self.nameToInum[tfile]
        curSize = self.inodes[inum].getSize()
        dprint('writeFile: inum:%d cursize:%d refcnt:%d' % (inum, curSize, self.inodes[inum].getRefCnt()))
        if curSize == 1:
            dprint('*** writeFile failed: file is full ***')
            return -1
        fblock = self.dataAlloc()
        if fblock == -1:
            dprint('*** writeFile failed: no data blocks left ***')
            return -1
        else:
            self.data[fblock].setType('f')
            self.data[fblock].addData(data)
        self.inodes[inum].setAddr(fblock)
        if printOps:
            print('fd=open("%s", O_WRONLY|O_APPEND); write(fd, buf, BLOCKSIZE); close(fd);' % tfile)
        return 0

    def doCreate(self, ftype):
        dprint('doCreate')
        parent = self.dirs[int(random.random() * len(self.dirs))]
        nfile = self.makeName()
        if ftype == 'd':
            tlist = self.dirs
        else:
            tlist = self.files

        if parent == '/':
            fullName = parent + nfile
        else:
            fullName = parent + '/' + nfile

        dprint('try createFile(%s %s %s)' % (parent, nfile, ftype))
        inum = self.createFile(parent, nfile, ftype)
        if inum >= 0:
            tlist.append(fullName)
            self.nameToInum[fullName] = inum
            if parent == '/':
                parent = ''
            if ftype == 'd':
                if printOps:
                    print('mkdir("%s/%s");' % (parent, nfile))
            else:
                if printOps:
                    print('creat("%s/%s");' % (parent, nfile))
            return 0
        return -1

    def doAppend(self):
        dprint('doAppend')
        if len(self.files) == 0:
            return -1
        afile = self.files[int(random.random() * len(self.files))]
        dprint('try writeFile(%s)' % afile)
        data = chr(ord('a') + int(random.random() * 26))
        rc = self.writeFile(afile, data)
        return rc
```

### 具体示例

创建文件 `/y` 并写入 10 字节数据的示例如下：

```
Initial state

inode bitmap  10000000
inodes        [d a:0 r:2] [] [] [] [] [] [] []
data bitmap   10000000
data          [(.,0) (..,0)] [] [] [] [] [] [] []

creat("/y");

inode bitmap  11000000
inodes        [d a:0 r:2] [f a:-1 r:1] [] [] [] [] [] []
data bitmap   10000000
data          [(.,0) (..,0) (y,1)] [] [] [] [] [] [] []

fd=open("/y", O_WRONLY|O_APPEND); write(fd, buf, BLOCKSIZE); close(fd);

inode bitmap  11000000
inodes        [d a:0 r:2] [f a:1 r:1] [] [] [] [] [] []
data bitmap   11000000
data          [(.,0) (..,0) (y,1)] [abcdefghij] [] [] [] [] [] []
```

### 4 改进该文件系统的存储结构和实现，成为一个支持崩溃一致性的文件系统

要支持崩溃一致性，我们需要实现以下几个关键的机制：

1. **日志系统**：引入一个日志系统来**记录每个操作的预期结果**。这样，在系统崩溃时，可以通过重放日志来恢复到最后一致的状态。
2. **写操作的原子性**：确保**写操作的原子性**，即要么完全执行，要么在崩溃时完全不执行。
3. **检查点**：定期创建**文件系统状态的快照**，以便在崩溃后恢复。
