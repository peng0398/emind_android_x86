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
- 7.设备以正常模式重新启动,新更新的boot分区被载入，它挂在好并启动更新后的系统分区,作为正常启动的一部分，系统会检查recovey分区的内容并与之前放入system分区的文件做对比，发现不同后，会对recovery分区进行更新
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

Android编译系统在初始化的过程中，会通过根目录下的Makefile脚本加载build/core/main.mk脚本，接着build/core/main.mk脚本又会加载build/core/Makefile脚本，而Android系统镜像文件就是由build/core/Makefile脚本负责打包生成的。



#### Build Img 过程分析
	
##### 主要使用到 build_image.py文件

> 该Python 脚本接收三个参数：

- input_directory 
- properties_file 
- output_image_file

 该脚本会根据 properties_file 生成对应的参数，并最终调用 mkuserimg.sh脚本来生成需要的img镜像文件

	mkuserimg.sh [-s] SRC_DIR OUTPUT_FILE EXT_VARIANT MOUNT_POINT SIZE [-j <journal_size>]
	             [-T TIMESTAMP] [-C FS_CONFIG] [-B BLOCK_LIST_FILE] [FILE_CONTEXTS]

	mkuserimg.sh out/target/product/x86_64/system out/target/product/x86_64/obj/PACKAGING/systemimage_interm 0 out/target/product/x86_64/root/file_contexts

	make_ext4fs -T -1 -S out/target/product/x86_64/root/file_contexts -l 1262M -a system out/target/product/
> python 脚本位置 ：./build/toos/releasetools/build_image.py


> 具体程序执行流程分析：

- main() 方法

	     def main(argv):
		  if len(argv) != 3:
		    print __doc__
		    sys.exit(1)
		
		  in_dir = argv[0]
		  glob_dict_file = argv[1]
		  out_file = argv[2]
		
		  glob_dict = LoadGlobalDict(glob_dict_file)
		  image_filename = os.path.basename(out_file)
		  mount_point = ""
		  if image_filename == "system.img":
		    mount_point = "system"
		  elif image_filename == "userdata.img":
		    mount_point = "data"
		  elif image_filename == "cache.img":
		    mount_point = "cache"
		  elif image_filename == "vendor.img":
		    mount_point = "vendor"
		  else:
		    print >> sys.stderr, "error: unknown image file name ", image_filename
		    exit(1)
		
		  image_properties = ImagePropFromGlobalDict(glob_dict, mount_point)
		  if not BuildImage(in_dir, image_properties, out_file):
		    print >> sys.stderr, "error: failed to build %s from %s" % (out_file, in_dir)
		    exit(1)

main函数会根据需要生成的镜像名选择合适的挂载点，之后构造镜像文件的属性信息，最后调用 BuildImage方法来生成镜像


- BuildImage() 方法：

		def BuildImage(in_dir, prop_dict, out_file):
		  """Build an image to out_file from in_dir with property prop_dict.
		
		  build_command = []
		  fs_type = prop_dict.get("fs_type", "")
		  run_fsck = False
		  if fs_type.startswith("ext"):
		    build_command = ["mkuserimg.sh"]
		    if "extfs_sparse_flag" in prop_dict:
		      build_command.append(prop_dict["extfs_sparse_flag"])
		      run_fsck = True
		    build_command.extend([in_dir, out_file, fs_type,
		                          prop_dict["mount_point"]])
		    if "partition_size" in prop_dict:
		      build_command.append(prop_dict["partition_size"])
		    if "selinux_fc" in prop_dict:
		      build_command.append(prop_dict["selinux_fc"])
		  else:
		    build_command = ["mkyaffs2image", "-f"]
		    if prop_dict.get("mkyaffs2_extra_flags", None):
		      build_command.extend(prop_dict["mkyaffs2_extra_flags"].split())
		    build_command.append(in_dir)
		    build_command.append(out_file)
		    if "selinux_fc" in prop_dict:
		      build_command.append(prop_dict["selinux_fc"])
		      build_command.append(prop_dict["mount_point"])
		
		  exit_code = RunCommand(build_command)
		  if exit_code != 0:
		    return False
		
		  if run_fsck and prop_dict.get("skip_fsck") != "true":
		    # Inflate the sparse image
		    unsparse_image = os.path.join(
		        os.path.dirname(out_file), "unsparse_" + os.path.basename(out_file))
		    inflate_command = ["simg2img", out_file, unsparse_image]
		    exit_code = RunCommand(inflate_command)
		    if exit_code != 0:
		      os.remove(unsparse_image)
		      return False
		
		    # Run e2fsck on the inflated image file
		    e2fsck_command = ["e2fsck", "-f", "-n", unsparse_image]
		    exit_code = RunCommand(e2fsck_command)
		
		    os.remove(unsparse_image)
		
		  return exit_code == 0

获取文件系统类型，根据不同的文件类型选择不同的命令，并拼接相关的参数，最终通过RunCommand方法将命令交由子进程执行，最终生成img镜像文件

由于我们需要的镜像文件系统类型为 ext4 ,因此此处会调用 mkuserimg.sh脚本来生成镜像，mkuserimg.sh部分脚本内容如下：

	system/extras/ext4_utils/mkuserimg.sh:
	
	MAKE_EXT4FS_CMD="make_ext4fs $ENABLE_SPARSE_IMAGE $FCOPT -l $SIZE -a $MOUNT_POINT $OUTPUT_FILE $SRC_DIR"
	echo $MAKE_EXT4FS_CMD
	$MAKE_EXT4FS_CMD
	if [ $? -ne 0 ]; then
	  exit 4
	fi

可以看到最终调用make_ext4fs生成所需镜像文件
