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
