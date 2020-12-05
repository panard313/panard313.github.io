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