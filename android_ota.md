## Android 系统OTA过程分析
### Android 系统目录结构

#### Boot
   包含了一个Linux kernel 和一个最小的root系统，这个分区可以挂载系统分区和其他分区并且能够启动系统分区中的Runtime。
#### System
   包含系统应用和相关的库文件（这些库文件的源码都存在于android系统的源码工程上），在正常操作情况下，这个分区是以只读的方式挂载的，只有在做OTA升级的时候这个分区的内容才会被改变
#### vendor
   包含系统应用和相关的库文件（这些库文件的源码不存在于android系统的源码工程上），在正常操作情况下，这个分区是以只读的方式挂载的，只有在做OTA升级的时候这个分区的内容才会被改变
#### userdata
   存储用户安装的应用程序所产生的数据或者其他数据，这个分区在OTA过程中不会受到影响。
#### cache
   临时的存储空间，被一些应用使用或者用来存储下载好的OTA升级包，这个分区的文件可能会随时丢失，部分OTA升级过程可能会导致该分区下的所有数据被擦除。
#### recovery
   包含一个完整的Linux系统，拥有一个kernel和一个特殊的程序可以用来读取升级包，更新其他分区。
#### misc
   由于recovery过程过程中设备可能会重启，这个小分区会用来暂存recovery过程中产生的一些数据。

### OTA升级过程如下：

- 1.设备检查是否有新的更新包，如果存在新包的话，OTA服务器会将OTA升级包的下载地址以及本次升级的描述返回给用户。
- 2.下载更新包到cache 分区或者数据分区，在通过本地/system/etc/security/otacerts.zip 证书验证后，用户被通知安装更新。
- 3.系统重启并进入recovery模式，这种模式下手机将启动recovery分区的系统，而不是启动boot分区的系统。
- 4.Recovery分区的更新程序被init启动，通过/cache/recovery/command的命令行参数找到指定的更新包。
- 5.验证更新包是否有问题。
- 6.更新包中的数据被用来更新boot ,system 以及vendor分区， One of the new files left on the system partition contains the contents of the new recovery partition.
- 7.设备以正常模式重新启动
    
   	

	> a.新更新的boot分区被载入，它挂在好并启动更新后的系统分区

	> b.作为正常启动的一部分，系统会检查recovey分区的内容并与之前放入system分区的文件做对比，发现不同后，会对recovery分区进行更新
	
- 8.至此，整个系统更新完成！！！

### OTA升级方式：
#### 1. 完整升级
   系统的最终状态（包括system,boot,以及recovery分区）都被打包到一个安装包中，只要系统能够接收更新包并且可以启动到recovery模式，系统就能安装上指定的版本。
#### 2. 增量升级
   增量更新是指更新包包含一整套的补丁，这些补丁会被打到设备中从而对系统进行更新，这种方式可以使得我们的更新包变小。

### OTA升级包生成过程
#### 1. build the target-files .zip
        % . build/envsetup.sh && lunch tardis-eng
        % mkdir dist_output
        % make dist DIST_DIR=dist_output
#### 2. Make OTA.zip  
        %./build/tools/releasetools/ota_from_target_files dist_output/tardis-target_files.zip ota_update.zip
        unzipping target target-files...
        done.
### make dist 过程分析
   make dist : 将所有的程序和相关的文档打包为一个压缩文件以供发布


