# 如何分析查看 page cahce 内存中缓存了哪些文件 ( mmap + mincore )?

众所周知, 在linux 下使用 `Buffered I/O` 读写文件是要经过 page cache ( 通常把 buffer cache 也算到 page cache 里). 

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202303/202303171833934.png)

那么大家肯定好奇, 如何查看系统的 page cache 中都缓存了哪些文件, 以及各个文件缓存了多少个 page 页, 那些 page 页被缓存 ?

社区中 [pcstat](https://github.com/tobert/pcstat) 工具提供了查看 page cahce 缓存文件信息的方法, 该项目使用 golang 开发, 其最核心代码也就百行, 内部用到了两个系统调用来计算 page 页. mmap 用来映射文件到进程地址空间, mincore 用来判断文件有哪些 page 被 `page cache` 缓存了.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202303/202303121052113.png)

但 pcstat 有几个问题, 在查询某进程打开文件的 page cache 使用情况时, 只扫描了 `/proc/{pid}/maps`, 而没有去扫描 `/proc/{pid}/fd/*` 目录, 这样影响了最后的统计结果. 另外 pcstat 不支持全局查询分析 page cache 使用状态.

本人参考了 pcstat 的设计, 开发了加强版的 page cache 分析工具 `pgcacher`, 相比 pcstat 来说做了很多调整, 调整了进程打开文件的目录, 还支持全局查询、排序输出、支持多线程并发检索、目录深度递归、忽略小 size 文件、指定和排除目录等等选项.

`pgcacher` - 加强版 page cache 分析调试工具:

[https://github.com/rfyiamcool/pgcacher](https://github.com/rfyiamcool/pgcacher)

## 分析文件在 page cache 缓存占用信息的实现原理

### 如何分析文件在 page cache 缓存使用情况

查看分析某个文件在 page cache 缓存中的统计原理是这样的. 

1. 先打开需要查看的文件, 把文件 mmap 到进程地址空间上. 
2. 获取文件的 size, 然后创建一个长度为 filesize / page 4k 长度的字节数组.
3. 接着使用 `syscall.Mincore` 对 mmap 映射的空间做 page cache 的缓存标记操作. 通过 mincore 系统调用可以得到文件的哪些 `page` 被 page cache 缓存了, 被缓存 page 为 1, 没被缓存则为 0.
4. 最后计算多少个 page cached, 就可以求出该文件在 page cache 内存里缓存占用情况.

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202303/202303131739063.png)

那么文件的 page cache 内存使用情况，只需要使用被缓存的 page 的数量乘以系统的 page size 即可, page size 默认为 4KB.

```sh
# getconf PAGESIZE
4096
```

下面是 `pgcacher` 中计算文件 page cache 缓存状态的 `GetFileMincore` 方法, 其实现原理跟上述是一致的.

```go
import (
	"fmt"
	"os"
	"unsafe"

	"golang.org/x/sys/unix"
)

type Mincore struct {
	Cached int64
	Miss   int64
}

func GetFileMincore(f *os.File, size int64) (*Mincore, error) {
	if int(size) == 0 {
		return nil, nil
	}

	mmap, err := unix.Mmap(int(f.Fd()), 0, int(size), unix.PROT_NONE, unix.MAP_SHARED)
	if err != nil {
		return nil, fmt.Errorf("could not mmap: %v", err)
	}

	vecsz := (size + int64(os.Getpagesize()) - 1) / int64(os.Getpagesize())
	vec := make([]byte, vecsz)

	mmap_ptr := uintptr(unsafe.Pointer(&mmap[0]))
	size_ptr := uintptr(size)
	vec_ptr := uintptr(unsafe.Pointer(&vec[0]))

	ret, _, err := unix.Syscall(unix.SYS_MINCORE, mmap_ptr, size_ptr, vec_ptr)
	if ret != 0 {
		return nil, fmt.Errorf("syscall SYS_MINCORE failed: %v", err)
	}
	defer unix.Munmap(mmap)

	value := new(Mincore)
	for _, b := range vec {
		if b%2 == 1 {
			value.Cached++
		} else {
			value.Miss++
		}
	}

	return value, nil
}
```

### 如何计算进程打开文件的 page cache 占用情况 ?

`pcstat` 对进程打开文件查询时, 只查询了 `/proc/{pid}/maps` 文件, 但该文件没有完整记录进程当前打开的文件列表, 其实还需要到 `/proc/{pid}/fd/*` 来查询. `pgcacher` 里做了这方面的调整, 把这两个目录的文件做了合并,最终 pgcacher 拿到的进程文件列表跟 `lsof -p pid` 一样的. 然后对这些文件进行 `mmap + mincore` 遍历查询, 计算出每个文件的 page cache 使用情况.

需要注意的是, 在使用 pgcacher 对进程和全局做 page cache 缓存信息扫描时, 只能针对已打开文件, 毕竟 pgcacher 是通过 `/proc/{pid}/fd/*` 来扫描的, 如果进程把文件关了, 自然就扫不到了. 比如 rocksDB 进程打开了一些 SSTable 数据文件，在遍历读取完后又关闭了这些文件描述符. 那么 pgcacher 是拿不到已关闭的文件. 

当然倒是有个笨方法, 直接遍历 rocksDB 数据目录的所有文件, 求出该进程的 page cache 使用情况.

### 如何计算当前系统里哪些文件空间占用最多 ?

拿到当前系统的所有进程列表, 然后遍历轮询拿到各个进程的已打开文件列表, 接着使用 mmap + mincore 求出文件在 page cache 的缓存信息. 再拿到所有进程所有文件的 page cache 缓存信息后, 做个排序输出即可.

## pgcacher 使用文档

### 安装

#### source code compilation

```sh
git clone https://github.com/rfyiamcool/pgcacher.git
cd pgcacher
make build
sudo cp pgcacher /usr/local/bin/
pgcacher -h
```

#### github releases

[https://github.com/rfyiamcool/pgcacher/releases](https://github.com/rfyiamcool/pgcacher/releases)

1. download package from github releases url.
2. decompress the package.
3. copy `pgcacher` to `/usr/local/bin`.

#### use binary directly

test pass on ubuntu, centos 7.x and centos 8.x.

```
wget xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/files/pgcacher
chmod 777 pgcacher
\cp pgcacher /usr/local/bin
```

### 使用过程

#### pgcacher 命令帮助文档.

```sh
pgcacher <-json <-pps>|-terse|-default> <-nohdr> <-bname> file file file
    -limit limit the number of files displayed, default: 500
    -depth set the depth of dirs to scan, default: 0
    -worker concurrency workers, default: 2
    -pid show all open maps for the given pid
    -top scan the open files of all processes, show the top few files that occupy the most memory space in the page cache, default: false
    -lease-size ignore files smaller than the lastSize, such as '10MB' and '15GB'
    -exclude-files exclude the specified files by wildcard, such as 'a*c?d' and '*xiaorui*,rfyiamcool'
    -include-files only include the specified files by wildcard, such as 'a*c?d' and '*xiaorui?cc,rfyiamcool'
    -json output will be JSON
    -pps include the per-page information in the output (can be huge!)
    -terse print terse machine-parseable output
    -histo print a histogram using unicode block characters
    -nohdr don't print the column header in terse or default format
    -bname use basename(file) in the output (use for long paths)
    -plain return data with no box characters
    -unicode return data with unicode box characters
```

#### pgcacher 各种场景的使用过程.

查询该进程的 page cahce 使用占比, 附加了一个并发参数, 开启 5 个线程并发扫描.

```
# sudo pgcacher -pid=29260 -worker=5
+-------------------+----------------+-------------+----------------+-------------+---------+
| Name              | Size           │ Pages       │ Cached Size    │ Cached Pages│ Percent │
|-------------------+----------------+-------------+----------------+-------------+---------|
| /root/rui/file4g  | 3.906G         | 1024000     | 3.906G         | 1024000     | 100.000 |
| /root/rui/file3g  | 2.930G         | 768000      | 2.930G         | 768000      | 100.000 |
| /root/rui/file2g  | 1.953G         | 512000      | 1.953G         | 512000      | 100.000 |
| /root/rui/file1g  | 1000.000M      | 256000      | 1000.000M      | 256000      | 100.000 |
| /root/rui/open_re | 1.791M         | 459         | 1.791M         | 459         | 100.000 |
|-------------------+----------------+-------------+----------------+-------------+---------|
│ Sum               │ 9.767G         │ 2560459     │ 9.767G         │ 2560459     │ 100.000 │
+-------------------+----------------+-------------+----------------+-------------+---------+
```

查询多个文件在 page cache 的使用情况.

```
# dd if=/dev/urandom of=file1g bs=1M count=1000
# dd if=/dev/urandom of=file2g bs=1M count=2000
# dd if=/dev/urandom of=file3g bs=1M count=3000
# dd if=/dev/urandom of=file4g bs=1M count=4000
# cat file1g file2g file3g file4g > /dev/null

# sudo pgcacher file1g file2g file3g file4g
+--------+----------------+-------------+----------------+-------------+---------+
| Name   | Size           │ Pages       │ Cached Size    │ Cached Pages│ Percent │
|--------+----------------+-------------+----------------+-------------+---------|
| file4g | 3.906G         | 1024000     | 3.906G         | 1024000     | 100.000 |
| file3g | 2.930G         | 768000      | 2.930G         | 768000      | 100.000 |
| file2g | 1.953G         | 512000      | 1.953G         | 512000      | 100.000 |
| file1g | 1000.000M      | 256000      | 1000.000M      | 256000      | 100.000 |
|--------+----------------+-------------+----------------+-------------+---------|
│ Sum    │ 9.766G         │ 2560000     │ 9.766G         │ 2560000     │ 100.000 │
+--------+----------------+-------------+----------------+-------------+---------+
```

查看该目录下的所有文件在 page cache 缓存里的使用情况.

```
# sudo pgcacher /root/xiaorui.cc/*

+------------+----------------+-------------+----------------+-------------+---------+
| Name       | Size           │ Pages       │ Cached Size    │ Cached Pages│ Percent │
|------------+----------------+-------------+----------------+-------------+---------|
| file4g     | 3.906G         | 1024000     | 3.906G         | 1024000     | 100.000 |
| file3g     | 2.930G         | 768000      | 2.930G         | 768000      | 100.000 |
| file2g     | 1.953G         | 512000      | 1.953G         | 512000      | 100.000 |
| testfile   | 1000.000M      | 256000      | 1000.000M      | 256000      | 100.000 |
| file1g     | 1000.000M      | 256000      | 1000.000M      | 256000      | 100.000 |
| pgcacher   | 2.440M         | 625         | 2.440M         | 625         | 100.000 |
| open_re    | 1.791M         | 459         | 1.791M         | 459         | 100.000 |
| cache.go   | 19.576K        | 5           | 19.576K        | 5           | 100.000 |
| open_re.go | 644B           | 1           | 644B           | 1           | 100.000 |
| nohup.out  | 957B           | 1           | 957B           | 1           | 100.000 |
|------------+----------------+-------------+----------------+-------------+---------|
│ Sum        │ 10.746G        │ 2817091     │ 10.746G        │ 2817091     │ 100.000 │
+------------+----------------+-------------+----------------+-------------+---------+
```

查看当前系统下所有进程的已打开文件在 page cache 缓存里的使用情况.

```
# sudo pgcacher -top -limit 3

+------------------+----------------+-------------+----------------+-------------+---------+
| Name             | Size           │ Pages       │ Cached Size    │ Cached Pages│ Percent │
|------------------+----------------+-------------+----------------+-------------+---------|
| /root/rui/file4g | 3.906G         | 1024000     | 3.906G         | 1024000     | 100.000 |
| /root/rui/file3g | 2.930G         | 768000      | 2.930G         | 768000      | 100.000 |
| /root/rui/file2g | 1.953G         | 512000      | 1.953G         | 512000      | 100.000 |
|------------------+----------------+-------------+----------------+-------------+---------|
│ Sum              │ 8.789G         │ 2304000     │ 8.789G         │ 2304000     │ 100.000 │
+------------------+----------------+-------------+----------------+-------------+---------+
```

递归遍历 4 层查看 aaa 目录所有子文件在 page cache 缓存里的使用情况.

```
# sudo pgcacher -depth=4 aaa/

+---------------------+----------------+-------------+----------------+-------------+---------+
| Name                | Size           │ Pages       │ Cached Size    │ Cached Pages│ Percent │
|---------------------+----------------+-------------+----------------+-------------+---------|
| aaa/a2g             | 1.953G         | 512000      | 1.953G         | 512000      | 100.000 |
| aaa/bbb/ccc/ddd/d2g | 1.953G         | 512000      | 1.940G         | 508531      | 99.322  |
| aaa/bbb/ccc/c1g     | 1000.000M      | 256000      | 1000.000M      | 256000      | 100.000 |
| aaa/bbb/ccc/c2g     | 1.953G         | 512000      | 1000.000M      | 256000      | 50.000  |
| aaa/bbb/ccc/ddd/d1g | 1000.000M      | 256000      | 1000.000M      | 256000      | 100.000 |
| aaa/a1g             | 1000.000M      | 256000      | 1000.000M      | 256000      | 100.000 |
| aaa/bbb/bbb1g       | 1000.000M      | 256000      | 1000.000M      | 256000      | 100.000 |
| aaa/bbb/bbb2g       | 1.953G         | 512000      | 1000.000M      | 256000      | 50.000  |
|---------------------+----------------+-------------+----------------+-------------+---------|
│ Sum                 │ 11.719G        │ 3072000     │ 9.752G         │ 2556531     │ 83.220  │
+---------------------+----------------+-------------+----------------+-------------+---------+
```

## 总结

![](https://xiaorui-cc.oss-cn-hangzhou.aliyuncs.com/images/202303/202303121052113.png)

本文主要讲解了 `如何分析文件在 page cache 缓存中的使用情况`, 其原理其实就两个系统调用, mmap 用来把文件映射到进程的地址空间里, 然后 mincore 通过 mmap 映射的空间计算出哪些 page 页被缓存. 只需要计算出多少个 page 页被 page cahce 缓存, 就可求出当前文件在 page cache 缓存中的使用情况.

`pgcacher` 用是个分析 page cahce 使用情况的调试工具, 它实现了对文件列表 page cache 的分析, 对进程的 page cache 分析, 及当前系统全局的 page cache 分析.

[https://github.com/rfyiamcool/pgcacher](https://github.com/rfyiamcool/pgcacher)