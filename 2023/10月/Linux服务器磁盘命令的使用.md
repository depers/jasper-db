> Linux

> 最近在做项目的时候，我需要将本地打包好的jar放置到服务器上去运行，但是在通过FTP上传的时候报错，最后发现是因为磁盘空间不够了导致的。所以这里写一篇文章记录下磁盘相关命令的使用

# df

* 英文全称：disk free。
* 命令格式：`df [-ahikHTm] [目录或文件名]`
* 功能：统计磁盘空间或文件系统使用情况，显示磁盘分区上的可使用的磁盘空间，默认显示单位为KB。

## 使用案例

1. `df`：显示所有挂载的文件系统的磁盘使用情况。

    ```shell
    depers@depersdeMacBook-Pro / % df
    Filesystem     512-blocks      Used Available Capacity iused      ifree %iused  Mounted on
    /dev/disk3s1s1  965595304  17697928 644317080     3%  355382 3221585400    0%   /
    devfs                 394       394         0   100%     682          0  100%   /dev
    /dev/disk3s6    965595304   6291496 644317080     1%       3 3221585400    0%   /System/Volumes/VM
    /dev/disk3s2    965595304   9068848 644317080     2%     784 3221585400    0%   /System/Volumes/Preboot
    /dev/disk3s4    965595304     19792 644317080     1%      43 3221585400    0%   /System/Volumes/Update
    /dev/disk1s2      1024000     12328    985176     2%       1    4925880    0%   /System/Volumes/xarts
    /dev/disk1s1      1024000     12656    985176     2%      30    4925880    0%   /System/Volumes/iSCPreboot
    /dev/disk1s3      1024000      4120    985176     1%      48    4925880    0%   /System/Volumes/Hardware
    /dev/disk3s5    965595304 286326208 644317080    31%  817544 3221585400    0%   /System/Volumes/Data
    map auto_home           0         0         0   100%       0          0  100%   /System/Volumes/Data/home
    ```

2. `df -h`：将容量结果以易读的容量格式显示出来。

    ```shell
    depers@depersdeMacBook-Pro / % df -h
    Filesystem       Size   Used  Avail Capacity iused      ifree %iused  Mounted on
    /dev/disk3s1s1  460Gi  8.4Gi  307Gi     3%  355382 3221585640    0%   /
    devfs           197Ki  197Ki    0Bi   100%     682          0  100%   /dev
    /dev/disk3s6    460Gi  3.0Gi  307Gi     1%       3 3221585640    0%   /System/Volumes/VM
    /dev/disk3s2    460Gi  4.3Gi  307Gi     2%     784 3221585640    0%   /System/Volumes/Preboot
    /dev/disk3s4    460Gi  9.7Mi  307Gi     1%      43 3221585640    0%   /System/Volumes/Update
    /dev/disk1s2    500Mi  6.0Mi  481Mi     2%       1    4925880    0%   /System/Volumes/xarts
    /dev/disk1s1    500Mi  6.2Mi  481Mi     2%      30    4925880    0%   /System/Volumes/iSCPreboot
    /dev/disk1s3    500Mi  2.0Mi  481Mi     1%      48    4925880    0%   /System/Volumes/Hardware
    /dev/disk3s5    460Gi  137Gi  307Gi    31%  817544 3221585640    0%   /System/Volumes/Data
    map auto_home     0Bi    0Bi    0Bi   100%       0          0  100%   /System/Volumes/Data/home
    ```

3. `df -h [目录或文件名]`：将该文件或是目录所在的磁盘容量已易读的格式展示出来。

    ```shell
    depers@depersdeMacBook-Pro ~ % df -h Desktop 
    Filesystem     Size   Used  Avail Capacity iused      ifree %iused  Mounted on
    /dev/disk3s5  460Gi  137Gi  307Gi    31%  817624 3221444480    0%   /System/Volumes/Data
    ```

# du

* 英文全称：disk used。
* 命令格式：`du [-ahskm] [文件或目录名称]`
* 功能：用于估算文件或目录的磁盘空间使用量。

## 使用案例

1. `du`：显示当前目录下每个子目录和文件的磁盘空间使用量。

    ```shell
    depers@depersdeMacBook-Pro depers % du
    0	./logseq/.recycle
    32	./logseq
    248	./journals
    480	./assets
    40	./pages
    800	.
    ```

2. `du -a`：将文件的容量也列出来。

    ```shell
    depers@depersdeMacBook-Pro depers % du -a
    32	./logseq/config.edn
    0	./logseq/.recycle
    0	./logseq/custom.css
    32	./logseq
    8	./journals/2023_05_13.md
    8	./journals/2023_07_07.md
    8	./journals/2023_06_09.md
    ...省略
    ```

3. `du -h`：以人类可读的格式显示磁盘空间使用量。

    ```shell
    depers@depersdeMacBook-Pro depers % du -h
      0B	./logseq/.recycle
     16K	./logseq
    124K	./journals
    240K	./assets
     20K	./pages
    400K	.
    ```

4. `du -sh`：统计当前目录的容量大小。

    ```shell
    depers@depersdeMacBook-Pro depers % du -sh
    400K	.
    ```

# 参考文章

* [Linux 磁盘管理](https://www.runoob.com/linux/linux-filesystem.html)
* [【Linux】与磁盘相关的常用命令（自用）](https://zhuanlan.zhihu.com/p/641397199)