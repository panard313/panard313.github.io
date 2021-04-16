# Erofs

ROM4.0 ：
Wiki ： http://10.92.35.99/wiki/index.php/TCTROM_V4.0_R
下载： repo init -u git@huizhou.gitweb.com:qualcomm/manifest.git -m r6150_dev.xml --reference=/home/android/mirror && repo sync -j8 --no-tag

AP 编译:
source build/envsetup.sh
choosecombo 1 t1 userdebug false
./build.sh dist -j32 
机器：T1


非ROM架构：
http://cd.gerrit.tclcom.com:8081/#/c/mtk6761/platform/frameworks/base/+/152499/
http://cd.gerrit.tclcom.com:8081/#/c/mtk6761/platform/frameworks/base/+/154124/
http://cd.gerrit.tclcom.com:8081/#/c/mtk6761/platform/frameworks/native/+/158450/
http://cd.gerrit.tclcom.com:8081/#/c/mtk6761/platform/frameworks/base/+/158448/


ROM4.0上编译已成功，刷机存在问题，目前无法开机
ROM4.0刷机并测试erofs效果


mkerofsimage.sh 1012547584 /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/IMAGES/vendor.img /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/VENDOR -C /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/vendor_filesystem_config.txt -s /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/file_contexts.bin -t vendor -z lz4hc -r 100

./code/ROM_4.0/out/soong/host/linux-x86/bin/mkfs.erofs -C/root/99srv/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/vendor_filesystem_config.txt -s /root/99srv/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/file_contexts.bin -t /vendor -zlz4hc -r100 /root/99srv/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/IMAGES/vendor.img /root/99srv/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/VENDOR

./out/soong/host/linux-x86/bin/mkfs.erofs -C/root/99srv/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/vendor_filesystem_config.txt -s /root/99srv/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/file_contexts.bin -t /vendor -zlz4hc -r100 /root/99srv/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/IMAGES/vendor.img /root/99srv/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/VENDOR

./out/soong/host/linux-x86/bin/mkfs.erofs -C/home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/vendor_filesystem_config.txt -s /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/file_contexts.bin -t /vendor -zlz4hc -r100 /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/IMAGES/vendor.img /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/VENDOR

./out/host/linux-x86/bin/mkerofsimage.sh 1012572160 /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/IMAGES/vendor.img /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/VENDOR -C /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/vendor_filesystem_config.txt -s /home/zhangku.guo/code/ROM_4.0/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.zhangku.guo/META/file_contexts.bin -t vendor -z lz4hc -r 100





ROM-4.0:


MAKE_EROFS_CMD="mkfs.erofs $EROFS_OPTS $OUTPUT_FILE $SRC_DIR"
echo $MAKE_EROFS_CMD
$MAKE_EROFS_CMD
if [ $? -ne 0 ]; then
    exit 4
fi
SPARSE_SUFFIX=".sparse"
if [ "$SPARSE" = true ]; then
    img2simg $OUTPUT_FILE $OUTPUT_FILE$SPARSE_SUFFIX
    if [ $? -ne 0 ]; then
    ▏   exit 5
    fi
    mv $OUTPUT_FILE$SPARSE_SUFFIX $OUTPUT_FILE
fi
       

build_image.py - INFO    : The tree size of /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/VENDOR is 933 MB.
common.py - INFO    :   Running: "avbtool add_hashtree_footer --partition_size 996069376 --calc_max_image_size --prop com.android.build.vendor.fingerprint:TCL/t1/sm6150:11/RKQ1.200730.002/xiong.chen09101126:userdebug/release-keys --prop com.android.build.vendor.os_version:11 --prop com.android.build.vendor.security_patch:2020-09-05 --prop com.android.build.vendor.security_patch:2020-09-05"
common.py - INFO    :   Running: "avbtool add_hashtree_footer --partition_size 1012121600 --calc_max_image_size --prop com.android.build.vendor.fingerprint:TCL/t1/sm6150:11/RKQ1.200730.002/xiong.chen09101126:userdebug/release-keys --prop com.android.build.vendor.os_version:11 --prop com.android.build.vendor.security_patch:2020-09-05 --prop com.android.build.vendor.security_patch:2020-09-05"
common.py - INFO    :   Running: "avbtool add_hashtree_footer --partition_size 1012125696 --calc_max_image_size --prop com.android.build.vendor.fingerprint:TCL/t1/sm6150:11/RKQ1.200730.002/xiong.chen09101126:userdebug/release-keys --prop com.android.build.vendor.os_version:11 --prop com.android.build.vendor.security_patch:2020-09-05 --prop com.android.build.vendor.security_patch:2020-09-05"
common.py - INFO    :   Running: "avbtool add_hashtree_footer --partition_size 1012121600 --calc_max_image_size --prop com.android.build.vendor.fingerprint:TCL/t1/sm6150:11/RKQ1.200730.002/xiong.chen09101126:userdebug/release-keys --prop com.android.build.vendor.os_version:11 --prop com.android.build.vendor.security_patch:2020-09-05 --prop com.android.build.vendor.security_patch:2020-09-05"
verity_utils.py - INFO    : CalculateMinPartitionSize(996069376): partition_size 1012125696.
build_image.py - INFO    : Allocating 965 MB for /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/IMAGES/vendor.img.
common.py - INFO    :   Running: "avbtool add_hashtree_footer --partition_size 1012125696 --calc_max_image_size --prop com.android.build.vendor.fingerprint:TCL/t1/sm6150:11/RKQ1.200730.002/xiong.chen09101126:userdebug/release-keys --prop com.android.build.vendor.os_version:11 --prop com.android.build.vendor.security_patch:2020-09-05 --prop com.android.build.vendor.security_patch:2020-09-05"
common.py - INFO    :   Running: "mkerofsimage.sh 1012125696 /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/IMAGES/vendor.img /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/VENDOR -C /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/vendor_filesystem_config.txt -s /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/file_contexts.bin -t vendor -z lz4hc -r 100"
common.py - INFO    : in mkerofsimage.sh PATH=out/host/linux-x86/bin:out/host/linux-x86/bin/:system/extras/ext4_utils/:/home/xiong.chen/work/ROM-4.0-T1/prebuilts/build-tools/path/linux-x86:/home/xiong.chen/work/ROM-4.0-T1/out/.path
988404+0 records in
988404+0 records out
1012125696 bytes (1.0 GB, 965 MiB) copied, 2.66091 s, 380 MB/s
mkfs.erofs -C/home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/vendor_filesystem_config.txt -s /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/file_contexts.bin -t /vendor -zlz4hc -r100 /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/IMAGES/vendor.img /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/VENDOR



mkerofsimage.sh 1386901504 /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/IMAGES/vendor.img /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/VENDOR -C /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/vendor_filesystem_config.txt -s /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/file_contexts.bin -t vendor -z lz4hc -r 100


mkerofsimage.sh 1387524096 /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/IMAGES/vendor.img /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/VENDOR -C /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/vendor_filesystem_config.txt -s /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/file_contexts.bin -t vendor -z lz4hc -r 100"
mkerofsimage.sh: PATH=out/host/linux-x86/bin:out/host/linux-x86/bin/:system/extras/ext4_utils/:/home/xiong.chen/work/ROM-4.0-T1/prebuilts/build-tools/path/linux-x86:/home/xiong.chen/work/ROM-4.0-T1/out/.path
1355004+0 records in
1355004+0 records out
1387524096 bytes (1.4 GB, 1.3 GiB) copied, 4.36759 s, 318 MB/s
mkfs.erofs -C/home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/vendor_filesystem_config.txt -s /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/file_contexts.bin -t /vendor -zlz4hc -r100 /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/IMAGES/vendor.img /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/VENDOR


2020-09-12 16:12:38 - build_image.py - INFO    : build_image.py main: in_dir=out/target/product/qssi/product, glob_dict_file=out/target/product/qssi/obj/PACKAGING/product_intermediates/
product_image_info.txt, out_file=out/target/product/qssi/product.img, target_out=out/target/product/qssi/system, current_path=/home/xiong.chen/work/ROM-4.0-T1



cx:
PATH=out/host/linux-x86/bin:out/host/linux-x86/bin/:system/extras/ext4_utils/:/home/xiong.chen/work/ROM-4.0-T1/prebuilts/build-tools/path/linux-x86:/home/xiong.chen/work/ROM-4.0-T1/out/.path mkerofsimage.sh 1387524096 /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/IMAGES/vendor.img /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/VENDOR -D out/target/product/qssi/system -s /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/file_contexts.bin -t vendor -z lz4hc -r 100

PATH=out/host/linux-x86/bin:out/host/linux-x86/bin/:system/extras/ext4_utils/:/home/xiong.chen/work/ROM-4.0-T1/prebuilts/build-tools/path/linux-x86:/home/xiong.chen/work/ROM-4.0-T1/out/.path mkerofsimage.sh 1387524096 /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/IMAGES/vendor.img /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/VENDOR -s /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/file_contexts.bin -t vendor -z lz4hc -r 100


PATH=out/host/linux-x86/bin:out/host/linux-x86/bin/:system/extras/ext4_utils/:/home/xiong.chen/work/ROM-4.0-T1/prebuilts/build-tools/path/linux-x86:/home/xiong.chen/work/ROM-4.0-T1/out/.path mkerofsimage.sh 1386901504 /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/IMAGES/vendor.img /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/VENDOR -D out/target/product/qssi/system -C /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/vendor_filesystem_config.txt -s /home/xiong.chen/work/ROM-4.0-T1/out/target/product/sm6150/obj/PACKAGING/target_files_intermediates/t1-target_files-eng.xiong.chen/META/file_contexts.bin -t vendor -z lz4hc -r 4


SeattleTmo:

Running: "avbtool add_hashtree_footer --partition_size 1055993856 --calc_max_image_size --prop com.android.build.vendor.os_version:10 --prop com.android.build.vendor.security_patch:2020-08-05 --prop com.android.build.vendor.security_patch:2020-08-05"

mkerofsimage.sh 1055993856 out/target/product/seattletmo/vendor.img out/target/product/seattletmo/vendor -D out/target/product/seattletmo/system -s out/target/product/seattletmo/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin -t vendor -z lz4hc -r 100

mkfs.erofs -Dout/target/product/seattletmo/system -s out/target/product/seattletmo/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin -t /vendor -zlz4hc -r100 out/target/product/seattletmo/vendor.img out/target/product/seattletmo/vendor


PATH=out/host/linux-x86/bin/:/home/xiong.chen/work/SeattleTmo-LA1.1/prebuilts/build-tools/path/linux-x86:/home/xiong.chen/work/SeattleTmo-LA1.1/out/.path mkerofsimage.sh 2684354560 out/target/product/seattletmo/product.img out/target/product/seattletmo/product -D out/target/product/seattletmo/system -s out/target/product/seattletmo/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin -t product -z lz4hc -r 4


https://git.kernel.org/pub/scm/linux/kernel/git/chao/linux.git/tree/drivers/staging/erofs?h=erofs




mkfs.erofs --target-out-path=out/target/product/seattletmo/system --file-contexts=out/target/product/seattletmo/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin --mount-point=/vendor -zlz4,4 out/target/product/seattletmo/vendor.img out/target/product/seattletmo/vendor
mkfs.erofs 1.1
        c_version:           [     1.1]
        c_dbg_lvl:           [       0]
        c_dry_run:           [       0]
        c_mount_point:       [ /vendor]
        c_target_out_path:   [out/target/product/seattletmo/system]
        c_fs_config_file:    [  (null)]
avbtool add_hashtree_footer --partition_size 1428713472 --partition_name vendor --image out/target/product/seattletmo/vendor.img --prop com.android.build.vendor.os_version:10 --prop com.android.build.vendor.security_patch:2020-08-05 --prop com.android.build.vendor.security_patch:2020-08-05




mkerofsimage.sh 2684354560 out/target/product/seattletmo/product.img out/target/product/seattletmo/product -D out/target/product/seattletmo/system -s out/target/product/seattletmo/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin -t product -z lz4hc -r 4




Test cmd:
顺序读
iozone -i 1 -s 300m -r 1024k -+E -w -f /vendor/enwik9-350m
随机读
iozone -i 2 -s 350m -r 4k -+E -w -f /vendor/enwik9-350m


fio --readonly -rw=randread -size=350m -bs=4k -name=/vendor/enwik9



echo 3 > /proc/sys/vm/drop_caches




SeattleTmo R upgrade:

FR:

9961347    10039119
kernel/msm-4.19/arch/arm64/configs/vendor

9961554    10039083
device/tct/seattletmo/seattletmo.mk

9961544    10039087
frameworks/base/core/java/android/view

9961538    10039109   注意冲突， 使用diff
frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

9961568    10039077
frameworks/base/packages/SystemUI/res/values/config.xml

9961530    10039118    8932722
device/tct/seattletmo


10039123
core/java/android/database/sqlite/SQLiteConnection.java



1, q3季度没有新的文档输出，仅完善了部分io/cgroup v2的文档。

1, 移植erofs文件系统到seattletmo，顺利开机并对比测试了不同文件系统的速度。
2, 针对erofs在seattletmo上存在相比其它文件系统读速度无提升的问题，移植最新的erofs-utils和erofs文件系统驱动到seattletmo，代码移植和编译完成。
3, 上条基础上存在无法开机问题，debug发现从5.4内核移植到4.19内核的erofs文件系统存在读取到一定size就会出现解压缩失败的问题，在seattletmo和虚拟机表现一致，仍在继续debug.
4, 9月末已经切到项目线， erofs抽空持续进行。

1, seattletmo项目顺利TA, 重要patch/solution均已合入TA软件。
2, seattletmo项目TA软件后重新检查了全部FR/Solution，并将剩余的patch全部导入MR分支，完成收尾。
3, TA软件点检，mr软件点检完成。

1, 分析并定位到偶现卡顿和val自动化测试出现anr的原因为zram耗尽导致内存不足，并在协助下调整了lmkd参数，解决了问题，并合入了TA软件，未将问题注入用户。
2, TA软件未遗留大的性能问题。
3, 处理完了seattletmo全部的defect， 没有遗留。
4, 及时处理了vts测试fail项目， 未造成软件delay.


### seattletmo R SystemUI

http://cd.gerrit.tclcom.com:8081/#/c/quicl/platform/frameworks/base/+/121583/

7161436， 看描述中的 2020/05/22

Task 9341452: [Lock Screen] Handling of probability exception in face key unlocking process





### tokyo lite tmo r

adb shell dumpsys pinner生效，
 
/system/lib/libhidltransport.so 4096 /system/framework/arm64/boot-framework.oat 12062720 /system/framework/framework.jar 30199808 /system/framework/framework-res.apk 17358848 /system/framework/oat/arm64/services.odex 30121984 /system/framework/services.jar 15106048 /system/framework/arm/boot-framework.oat 10018816 /system/framework/telephony-common.jar 3506176 /system/lib/libhwui.so 7401472 /system/lib64/libhwui.so 9736192 /system/framework/arm/boot-framework.art 3584000


PRODUCT_PRODUCT_PROPERTIES += \
	ro.lmk.swap_free_low_percentage=25 \
	ro.lmk.psi_partial_stall_ms=150 \
	ro.lmk.psi_complete_stall_ms=500 \
	ro.lmk.thrashing_limit=20 \
	ro.lmk.kill_timeout_ms=300


空进程FR: 9889898  re-check


no-need FRs:
9889743
9889759
9889765
9889769
已经在bankok项目上提交,无需再次提交
9889771
9889777
9889798
9889806
9889812
10107884
10107886
10107890




10107869 to fwk


1118:
10193060
9889790
10107894
9889842
9889737
10107875
10107877
10107896
10107898
10107902
10107906
10107918
10107910


1120:
9889834
10107912
10107922
//9889892




re-check:
10107879


Seattletmo R uiturbo:
art/runtime/base

r-seattletmo-r: android.hardware.gatekeeper@1.0-service-qti.rc  check cpuset top-app/ui

r-seattletmo-r: cpu频率低, ui线程未绑大核





PortoTmo-R:
sleep 3600;mkdir PortoTmo-R;cd PortoTmo-R;repo init -u git@shenzhen.gitweb.com:gcs_sz/manifest -m sm6125-r0-portotmo-dint.xml --reference=/home/android/mirror && repo sync -j8 --no-tags;repo start --all sm6125-r0-portotmo-dint;source build/envsetup.sh;choosecombo 1 portotmo userdebug portotmo 1 false false;TCT_EFUSE=false ANTI_ROLLBACK=2 SIGN_SECIMAGE_USEKEY=portotmo ./build.sh dist -j16  2>&1 | tee build.log

sleep 9600;cd PortoTmo-R;cd amss_nicobar_la2.0.1;TCT_EFUSE=false ANTI_ROLLBACK=2 bash linux_build.sh -s -a portotmo tmo


1.userdebug
是否有复现？ ----> 有复现.
2.driver only 版本是否复现？ ----> driver only版本未复现
3.确认最新版本的codebase和最新版本的apk是否有复现  ----> 目前已经是最新版本,并且命令行测试即可复现, 与apk版本看起来并没有关系. 
4.确认这是否是P升Q再升R的版本，还是直接Q升R的版本，还是非OTA版本。
----> Q升R版本

[1 of 1][10304155][performance] Fuse file-system performance optimize 
###%%%comment:[performance] Fuse file-system performance optimize 
###%%%bug number:10304155 ###%%%product name:sm7250-r0-seattletmo-dint 
###%%%Root Cause:Unknown_Today 
###%%%Root Cause Detail:performance 
###%%%Test_Suggestion:none for yet 
###%%%Test_Report:none for yet 
###%%%Bug category:Platform 
###%%%Module_Impact:NO 
###%%%Solution:NO 
###%%%Merge to Global:NO 
###%%%Pre_build:YES 
Change-Id: I27e2080ca6c6ea47e618eae9c83e1f03835572bd


bc360:
我们查看了bc360几个进程的内存占用情况:
27,275K: com.bc360.control (pid 26346 / activities)
13,641K: com.bc360.android.service:remote (pid 26479)
 8,131K: com.bc360.android.service (pid 3743)

三个进程总共占用内存48M左右；(PS: 当前并未能确认音效是否已生效)
去除共享库部分, 实际的内存开销应该在40M左右.

未完装三方应用且未登陆gms的机器开机剩余内存约1.2G, 我们认为bc360对机器的内存压力不大. 
若需要避免bc360的service被系统kill, 可以在系统中做一些保活处理, 避免非极端状态下kill掉相应进程.


We checked the memory usage of several processes in bc360

The total memory occupied by the three processes is about 48m; (PS: we can't confirm whether the sound effect is effective at present.)

Excluding the shared library, the actual memory cost should be about 40m.



We think that the memory pressure of bc360 is not too big.

If it is necessary to prevent bc360 service from being killed by the system, we can do some live keeping processing in the system to avoid killing the corresponding process in non critical state.


绩效管理系统陈雄   注销
系统首页
我的CTS2
返回     /表单详情
[陈雄]2020年第四季度绩效考核 - 员工自评
目标设定
主管审核
员工自评
主管绩效评估
绩效结果确认
绩效面谈完成
目标类型	指标/目标	衡量标准/关键结果	权重
 KO	
KO项：
1, 继续erofs的移植/研究/debug， 尽可能达到稳定。
2, 继续cgroup v2 io部分的研究，完成cgroup v2在android上的启用并测试实际效果。
自评

KO项措施有效性，最高分100分
１. 优化明显且普适　100-120分
 2. 对比数据有提升且能够适用大部分场景　85-99分
 3. 没有明显优化，但能够输出详细文档并指明原因　60-84分
４. 无法按时完成且无有效输出　0-59分
 
自评内容(最长3500个字符)
%
50
 KO	
"文档输出和分享：
1. 模块深入研究并形成文档并分享。
2. 对问题的解决过程总结形成文档并组内分享。
3. 业界性能功耗稳定性有关的技术文档分享"
自评

1. 最高分100分
2. 1篇模块研究文档+40分，分享培训+20分
3. 1篇总结文档及分享+20分，周会上分享+5分
4. 1篇业界技术文档并组内分享+10分"
 
自评内容(最长3500个字符)
%
5
 KPI	
Performance问题解决
1.及时解决SeattleTMO/PortoTMO/TokyoLiteTMO/Thor84GVZW/Apollo84GTMO等项目的defect
2.项目TA的时候进行TA点检
自评

任务(PR或者其他任务)在要求时间内完成的比例：
1.前10%：100-120分
2.前50%：85-99分
3.前95%：60-84分
4.后4%：1-59分
说明：解决重要疑难问题不受解决数量限制，重要问题以DM/TM/XPM认可为准作为参考
 
自评内容(最长3500个字符)
%
20
 KPI	
TOP FR/Solution导入
1. 继续完成SeattleTMO项目TOP FR/Solution导入
2. 每个项目FC前点检，发布report给val进行测试
3. 如有问题支持val测试
自评

任务(PR或者其他任务)在要求时间内完成的比例：
1.前10%：100-120分
2.前50%：85-99分
3.前95%：60-84分
4.后4%：1-59分
说明：解决重要疑难问题不受解决数量限制，重要问题以DM/TM/XPM认可为准作为参考
 
自评内容(最长3500个字符)
%
25


MTK defect:
10436522

2020-0245368
D44131194560
8200921626009


10017150,10104877


wfc:

从bugreport看, 未见内存状态异常, 系统负载较高:Load: 4.98 / 4.61 / 4.05, 可能是引起anr的主要原因.
anr直接原因为inputdispatcher没有焦点窗口超时, 当前porttom项目有遇到几个类似问题, 正在查找是否存在共性原因.

从log可以看到一些可能导致负载高的点, 可以针对性优化一下试试:
1, ActivityManager: Sending non-protected broadcast action_wfc_summary_change from system 3382:com.android.phone/1001 pkg com.android.phone
这个广播可以加入白名单, 减小系统开销
2, 还是这个广播, 发送太过频繁, 能否通过一些机制去减缓发送?例如一秒内有多个状态改变消息, 仅发送最后一个, 其它丢弃?
3, WfcSummary对wifi状态检测太过频繁, 间隔有的短到了几ms, 是否可以加大一下间隔或者注册广播?

以上, 应当可以减小phone和wfc产生的负载, 减小同情景下的anr概率

input事件时间点:
12-29 14:34:26.450左右

中间发生了一些错误日志, 不清楚是否会影响phone或者wfc:
12-29 14:34:26.583  1286  1286 E Diag_Lib:  Diag_LSM_Init: Failed to open handle to diag driver, error = 13

超时丢弃事件:
12-29 14:34:31.441  1878  2402 W InputDispatcher: Still no focused window. Will drop the event in 4999ms

产生anr:
12-29 14:34:31.485  1878 19470 I am_anr  : [0,3330,com.tct.wfcmanager,814267981,Input dispatching timed out (ActivityRecord{15ffbff u0 com.tct.wfcmanager/.sprint.settings.              TctWifiCallingSettingsActivity t100} does not have a focused window)]



我并不了解你说的the issue “frequent failures of launched software”发生在什么时候.
在调试保活机制时发现,  进程"com.bc360.android.service:remote" 会被系统系统杀掉以释放内存(这是正常机制, 此进程的优先级是可以被杀掉的). 当此进程再次启动时, 进程状态会有数秒处于阻塞状态, 这段时间内通过coltroller去设置音效时, 会导致崩溃, 不知道你所说的是不是这个问题.
而我所做的修改就是保证"com.bc360.android.service"和"com.bc360.android.service:remote"这两个进程不会被系统杀掉. 实测重载压力测试36小时, 它们的进程号并未改变, log显示未被杀掉. 且随机去设置音效, 均可顺利设置成功.




R:

Total RAM: 3,676,904K (status normal)
 Free RAM: 1,601,314K (  580,122K cached pss +   910,928K cached kernel +   110,264K free)
      ION:    85,912K (    5,336K mapped +    80,576K unmapped +         0K pools)
 Used RAM: 2,069,542K (1,554,118K used pss +   515,424K kernel)
 Lost RAM:   108,346K
     ZRAM:    44,920K physical used for   170,388K in swap (2,097,148K total swap)
   Tuning: 256 (large 512), oom   322,560K, restore limit    80,640K (high-end-gfx)

unknown:/ # sysctl -a 2>/dev/null |grep vm.dirty
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 5
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 200
vm.dirty_ratio = 20
vm.dirty_writeback_centisecs = 500
vm.dirtytime_expire_seconds = 43200


Q:

Total RAM: 3,719,952K (status normal)
 Free RAM: 1,345,262K (  351,742K cached pss +   916,020K cached kernel +    77,500K free)
 Used RAM: 2,329,325K (1,449,889K used pss +   879,436K kernel)
 Lost RAM:   198,848K
     ZRAM:    51,752K physical used for   215,156K in swap (2,097,148K total swap)
   Tuning: 256 (large 512), oom   322,560K, restore limit    80,640K (high-end-gfx)

adb shell sysctl -a 2>/dev/null |grep vm.dirty
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 5
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 200
vm.dirty_ratio = 20
vm.dirty_writeback_centisecs = 500
vm.dirtytime_expire_seconds = 43200



[07bfa4415] fat: work around race with userspace's read via blockdev while mounting <hirofumi@mail.parknet.co.jp> OGAWA Hirofumi (2019-09-23 22:32:53)
- dir.c fatent.c
[bd8309de0d60838eef6fb575b0c4c7e95841cf73] fs/fat/file.c: issue flush after the writeback of FAT
- file.c
[c0d0e3512] fat: fix using uninitialized fields of fat_inode/fsinfo_inode <hirofumi@mail.parknet.co.jp> OGAWA Hirofumi (2017-03-10 00:17:37)
inode.c
[265b81a52] fat: fix uninit-memory access for partial initialized inode <hirofumi@mail.parknet.co.jp> OGAWA Hirofumi




Defect 10445294: [SD Card IOT]The transfer rate is slowly when from PC to Memory card

问题发生在<=32GB容量的tf卡, 通过mtp拷贝时速度只有9.5MB/s, 而同机型Q版本上约15MB/s.

排查过程:
1, 在ubuntu上通过gvfs往挂载的手机外置sd卡上使用time cp拷贝200M左右的文件, 因文件系统缓存的问题导致出现误判, 将原因归在了fuse上面.
后来测试时一律使用sync;echo 3 > /proc/sys/vm/drop_caches来清空缓存, 然后再拷贝大一些的文件, 使测试写入速度较为准确.

2, 每次测试前先执行sync;echo 3 > /proc/sys/vm/drop_caches以排除page cache影响, 测得结果稳定且排除了fuse的影响
tips: 在开发者选项---> feature flags ----> settings_fuse 可以直接开关fuse, 重启生效.

把其它项目上用的iozone直接拿来测试, 结果与cp命令测接近, 进一步证明测试方法ok.

    做了以下测试:
        测试条件:
        1,所有测试圴先执行sync;echo 3 > /proc/sys/vm/drop_caches以排除page cache影响
        2, 同一张tf卡, 未在手机上格式化(借的, 不方便格式化)
        测试数据:
        1, time cp -tvf /sdcard/500m /storage/0403-0201/
            Q: 35.97s
            R: fuse 54.25s
            no-fuse 53.45s
        2, iozone -i 0 -s 500m -w -f /storage/0403-0201/500m
            Q:  15577 KB/s
                15306 KB/s
            R: fuse 11809 KB/s
                    11336 KB/s
            no-fuse  10573 KB/s
                        10574 KB/s
    其中测试1的结果与mtp文件复制的时间基本一致.
    ps:
        iozone directio模式在Q上运行异常, 并未统计directio的数据
    
    从以上结果来看, R版本上外置tf卡的写速度似乎就比Q慢一些, 并且与fuse无关. 已经对比过R与Q的mmc1 clock, 均为202M.

3, 海望建议测试一下直接写块设备节点, 看一下在文件系统层之上的写入速度, 以排除文件系统影响
测试命令:
    iozone -i 0 -s 500m -w -f /dev/block/vold/165:79

    对比下来R上约22MB/s, Q上约15MB/s, 也就是说sdmmc总线的写入速度方面R并不弱于Q, 可以排除总线/tf卡兼容性问题, 区别就只能在文件系统.

4, 对比了Q和R的kernel/fs/fat, 只有四个文件有差异, 而且看提交信息都是安全/bugfix, 看起来没有什么和性能相关的. 直接用Q的fs/fat覆盖了R的, 编译内核测试, 写入能到22MB/s, 与直接写block设备速度相符, 是使用尽了写入带宽的表现, 因此可以确定就是这几个提交的问题.

5, 逐个恢复提交, 编译内核测, 定位到了相应的提交.
[07bfa4415] fat: work around race with userspace's read via blockdev while mounting <hirofumi@mail.parknet.co.jp> OGAWA Hirofumi (2019-09-23 22:32:53)

6, 评估回退此提交可能会小概率导致坏卡, 交由spm权衡后决定保留.


思路:

排除pagecache, 加大测试文件size, 保证测试方法准确 ----> 对比Q版本, 开关fuse测试, 排除了fuse影响 ----> 直接测试写块设备, 排除文件系统影响(直接写块设备会破坏文件系统) ----> 对比文件系统差异, 定位提交记录

PS:
1, 32G及以下tf卡与64G及以上使用的文件系统不一致, 需要区分
2, 在开发者选项---> feature flags ----> settings_fuse 可以直接开关fuse, 重启生效.


Go
com.google.android.setupwizard/.carrier.SimMissingActivity------686]
com.google.android.setupwizard/.network.NetworkActivity------612]
com.google.android.gms/.setupservices.GoogleServicesActivity------3050]
com.android.settings/.password.SetupChooseLockPassword------3027]
com.android.settings/.faceunlock.SetupWizardFaceUnlockActivity------295]
com.google.android.setupwizard/.user.LoadLauncherLayout------241]
com.tcl.fota.system/.SetUpWizardScheduleAutoUpdateActivity------1010]
com.tct.setupwizard/.SetupWizardEndActivity------296]
com.tmobile.pr.mytmobile/.oobe.OOBEActivity------2665]
com.android.launcher3/.uioverrides.QuickstepLauncher------3400]
com.android.settings/.Settings------1605]
com.android.settings/.SubSettings------464]
com.android.settings/.SubSettings------425]
com.android.settings/.SubSettings------434]
com.android.dialer/.main.impl.MainActivity------1800]
com.google.android.permissioncontroller/com.android.permissioncontroller.permission.ui.GrantPermissionsActivity------1102]
com.google.android.permissioncontroller/com.android.permissioncontroller.permission.ui.GrantPermissionsActivity------1312]
com.debug.loggerui/.settings.SettingsActivity------249]
com.android.settings/.Settings$WifiSettings2Activity------1048]
com.android.settings/.SubSettings------441]
com.tmobile.rsuapp/.MainActivity------1882]
com.google.android.setupwizard/.carrier.SimMissingActivity------686]
com.google.android.setupwizard/.network.NetworkActivity------612]
com.google.android.gms/.setupservices.GoogleServicesActivity------3050]
com.android.settings/.password.SetupChooseLockPassword------3027]
com.android.settings/.faceunlock.SetupWizardFaceUnlockActivity------295]
com.google.android.setupwizard/.user.LoadLauncherLayout------241]
com.tcl.fota.system/.SetUpWizardScheduleAutoUpdateActivity------1010]
com.tct.setupwizard/.SetupWizardEndActivity------296]
com.tmobile.pr.mytmobile/.oobe.OOBEActivity------2665]
com.android.launcher3/.uioverrides.QuickstepLauncher------3400]
com.android.settings/.Settings------1605]
com.android.settings/.SubSettings------464]
com.android.settings/.SubSettings------425]
com.android.settings/.SubSettings------434]
com.android.dialer/.main.impl.MainActivity------1800]
com.google.android.permissioncontroller/com.android.permissioncontroller.permission.ui.GrantPermissionsActivity------1102]
com.google.android.permissioncontroller/com.android.permissioncontroller.permission.ui.GrantPermissionsActivity------1312]
com.debug.loggerui/.settings.SettingsActivity------249]
com.android.settings/.Settings$WifiSettings2Activity------1048]
com.android.settings/.SubSettings------441]
com.tmobile.rsuapp/.MainActivity------1882]

jiuyu.cui gapp,fota


at android.graphics.HardwareRenderer.nSyncAndDrawFrame(Native method)
at android.graphics.HardwareRenderer.syncAndDrawFrame(HardwareRenderer.java:433)
at android.view.ThreadedRenderer.draw(ThreadedRenderer.java:658)
at android.view.ViewRootImpl.draw(ViewRootImpl.java:4172)
at android.view.ViewRootImpl.performDraw(ViewRootImpl.java:3899)
at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3156)


02-20 22:27:58.823 radio  2678  2992 W TelephonyPermissions: reportAccessDeniedToReadIdentifiers:com.tencent.mm:getSubscriberId:1
02-20 22:27:58.846 10235  4737  5743 I chatty  : uid=10235(com.tencent.mm) expire 3 lines
02-20 22:27:58.856 10235  4737  7849 I chatty  : uid=10235(com.tencent.mm) expire 2 lines
02-20 22:27:58.857 10235  4737  6084 I chatty  : uid=10235(com.tencent.mm) expire 2 lines
02-20 22:27:58.907  1000   886  1100 D SurfaceFlinger: assign a empty target
02-20 22:27:58.938  1000  1921  2168 D ConnectivityService: releasing NetworkRequest [ TRACK_DEFAULT id=637, [ Capabilities: INTERNET&NOT_RESTRICTED&TRUSTED Uid: 10235                  AdministratorUids: [] RequestorUid: 10235 RequestorPackageName: com.tencent.mm] ] (release request)
02-20 22:27:59.642 10202  2266  2266 D ViewRootImpl[NavigationBar0]: Caller's package : com.android.systemui
02-20 22:28:00.004 10202  2266  2266 D KeyguardUpdateMonitor: received broadcast android.intent.action.TIME_TICK
02-20 22:28:00.004 10202  2266  2266 D KeyguardUpdateMonitor: handleTimeUpdate
02-20 22:28:00.341 10202  2266  2266 D ViewRootImpl[NavigationBar0]: Caller's package : com.android.systemui
02-20 22:28:00.795 10235  5454 13261 I chatty  : uid=10235 com.tencent.mm:push expire 2 lines
02-20 22:28:00.891 10202  2266  2266 D ViewRootImpl[NavigationBar0]: Caller's package : com.android.systemui
02-20 22:28:03.114 10202  2266  2266 D ViewRootImpl[NavigationBar0]: Caller's package : com.android.systemui
02-20 22:28:03.509 10202  2266  2266 D CarrierTextController: isCDMA:false cdmaRoamingName:null mCdmaRoamingName:null



http://sz.gerrit.tclcom.com:8080/#/c/TCTROM/apps_R/+/317654/
http://sz.gerrit.tclcom.com:8080/#/c/qualcomm/platform/frameworks/base/+/317653/
http://sz.gerrit.tclcom.com:8080/#/c/TCTROM/tcl/frameworks/+/317652/
http://sz.gerrit.tclcom.com:8080/#/c/TCTROM/tcl/device/+/317651/
http://sz.gerrit.tclcom.com:8080/#/c/TCTROM/tcl/build/+/317650/


- [ ] 拿val的机器跑auto-perf, 尽量挂5v5
- [ ] 用5j版本退掉那几个提交, 编个版本, 跑auto perf
- [ ] Transformor vzw feature确认
- [ ] Transformor vzw 可移除的进程确认
- [ ] portotmo/seattletmo 应用被杀的问题跟进
- [ ] update crosscheck list