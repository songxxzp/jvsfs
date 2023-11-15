# 课后练习8

### 宋曦轩 2021010728

### 1. 描述该文件系统的存储结构

```
[BitMap of inode][BitMap of data][inode][data]

[BitMap of inode] : 00101000 like
[BitMap of data]  : 00100110 like
[inode]           : [type(file or dir)，address, reference count]
[data] of dir     : [(filename, inode id), ...]
[data] of file    : [data]
```

### 2. 基于模拟器的执行过程跟踪分析，描述该文件系统中“读取指定文件最后10字节数据”时要访问的磁盘数据和访问顺序

模拟器中没有实现读操作，按照个人理解，读取指定文件最后10字节数据时：
0. 初始 `inode_id = 0`
1. 从 `inode[inode_id]` 获取 `address`
2. 从 `data[address]` 获取 `data`
3. 如果 `inode` 是 `dir`, 则从 `data` 中根据路径获取下一个 `inode_id`（不存在则报错）；否则读取 `data` 中最后 `10` 字节数据（如果 `data` 为空则报错）。

### 3. 基于模拟器的执行过程跟踪分析，描述该文件系统中“创建指定路径文件，并写入10字节数据”时要访问的磁盘数据和访问顺序

模拟器中没有创建并写入操作，合并创建和写操作，新操作流程如下：
0. 获取 parent 的 inode id，即 p_inode_id
1. 根据 parent 的 inode[p_inode_id] 中的 address ，获取 parent 的 data；如果 parent 目录已满，报错；如果存在同名文件，报错
2. 申请空闲 inode：访问inode bitmap，mark新分配的位置，返回 f_inode_id（若无则报错）
3. 申请空闲 data：访问data bitmap，mark新分配的位置，返回 f_data_id
4. 设置新data块为文件类型，并写入 10 字节数据。
5. inode[f_inode_id] = [f, f_data_id, 1]
6. 将 (新文件名, f_inode_id) 写入 parent 的 data

如果拆分为创建和写入两个操作的话：
0. 获取 parent 的 inode id，即 p_inode_id
1. 根据 parent 的 inode[p_inode_id] 中的 address ，获取 parent 的 data；如果 parent 目录已满，报错；如果存在同名文件，报错
2. 申请空闲 inode：访问inode bitmap，mark新分配的位置，返回 f_inode_id（若无则报错）
3. inode[f_inode_id] = [f, -1, 1]
4. 将 (新文件名, f_inode_id) 写入 parent 的 data
5. 获取 文件 的 f_inode_id
6. 申请空闲 data：访问data bitmap，mark新分配的位置，返回 f_data_id
7. 设置新data块为文件类型，并写入 10 字节数据。
8. inode[f_inode_id] = [f, f_data_id, 1]


### 4. 请改进该文件系统的存储结构和实现，成为一个支持崩溃一致性的文件系统。

设计日志系统

```python
class journal:
    def __init__(self):
        self.queue = []

    def TxB(self):
        if not self.check():
            raise AssertionError("File system damaged.")
        self.queue.append("TxB")
    
    def TxE(self):
        self.queue.append("TxE")

    def metadata(self, metadata):
        self.queue.append(metadata)

    def check(self):
        return len(self.queue) == 0 or self.queue[-1] == "TxE"

    def finish(self):
        if self.check():
            for op in self.queue:
                if isinstance(op, dict):
                    op["func"](*op["args"])
            self.queue = []
        else:
            raise AssertionError("File system damaged.")
    
    def recoverable(self):
        return len(self.queue) > 0

    def recover(self):
        self.finish()
```

对 `fs.deleteFile`，`fs.createLink`，`fs.createFile`，`fs.writeFile` 进行了修改。

以 `fs.writeFile` 函数为例：

```python
class fs:
    def writeFile(self, tfile, data):
        self.journal.TxB()  # 设置TxB
        inum = self.nameToInum[tfile]
        curSize = self.inodes[inum].getSize()
        dprint('writeFile: inum:%d cursize:%d refcnt:%d' % (inum, curSize, self.inodes[inum].getRefCnt()))
        if curSize == 1:
            dprint('*** writeFile failed: file is full ***')
            self.journal.TxE()
            self.journal.finish()  # excute ops in journal
            return -1
        fblock = self.dataAlloc()
        if fblock == -1:
            dprint('*** writeFile failed: no data blocks left ***')
            self.journal.TxE()
            self.journal.finish()  # excute ops in journal
            return -1
        else:
            self.journal.metadata(
                {
                    "fs": self,
                    "func": self.data[fblock].setType,
                    "args": ('f', )
                }
            )  # log op to journal instead of excute
            # self.data[fblock].setType('f')
            self.journal.metadata(
                {
                    "fs": self,
                    "func": self.data[fblock].addData,
                    "args": (data, )
                }
            )  # log op to journal instead of excute
            # self.data[fblock].addData(data)
        self.journal.metadata(
            {
                "fs": self,
                "func": self.inodes[inum].setAddr,
                "args": (fblock, )
            }
        )  # log op to journal instead of excute
        # self.inodes[inum].setAddr(fblock)
        if printOps:
            print('fd=open("%s", O_WRONLY|O_APPEND); write(fd, buf, BLOCKSIZE); close(fd);' % tfile)
        self.journal.TxE()
        self.journal.finish()  # excute ops in journal
        return 0
```

将所有写入命令记录到日志中。在全部写入完成后，调用 `journal.finish()` ，将日志中的命令全部执行并清空日志。
如果在任意完成原子指令之后，发现日志系统队列不为空 `journal.recoverable()` ，则尝试恢复 `journal.recover()`：
1. 存在 `TxE` ，重新执行日志内容，若成功则清空日志。
2. 不存在 `TxE` ，说明日志系统队列中存在未完成原子指令，抛弃改指令并发出异常。
