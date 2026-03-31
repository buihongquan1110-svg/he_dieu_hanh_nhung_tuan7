# TUẦN 7: XÂY DỰNG DRIVER GIAO TIẾP PHẦN CỨNG CƠ BẢN
## Bài 1: Hoàn thiện Driver có đủ các hàm cơ bản
### Cấu trúc thư mục Driver
```bash
package/my_cdev/
├── Config.in
├── my_cdev.mk
└── src/
    ├── my_cdev.c
    └── Makefile
```
### Tạo cấu trúc thư mục trong Buildroot
```bash
cd ~/buildroot
mkdir -p package/my_cdev/src
```
### Tạo và viết 4 file quan trọng
#### 1. File `"package/my_cdev/Config.in"`
```bash
config BR2_PACKAGE_MY_CDEV
	bool "my_cdev"
	depends on BR2_LINUX_KERNEL
	help
	  Driver thiet bi ky tu cho BeagleBone Black.
```
#### 2. File `"package/my_cdev/my_cdev.mk"`
```bash
MY_CDEV_VERSION = 1.0
MY_CDEV_SITE = $(TOPDIR)/package/my_cdev/src
MY_CDEV_SITE_METHOD = local

$(eval $(kernel-module))
$(eval $(generic-package))
```
#### 3. File `"package/my_cdev/src/Makefile"`
```bash
obj-m += my_cdev.o

all:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(LINUX_DIR) M=$(PWD) clean
```
#### 4. File `"package/my_cdev/src/my_cdev.c"`
```bash
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/device.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>
#include <linux/slab.h>

#define DEVICE_NAME "my_char_dev"
#define CLASS_NAME  "my_dev_class"

static int majorNumber;
static struct class* myClass  = NULL;
static struct device* myDevice = NULL;
static char kernel_buffer[1024];

static int dev_open(struct inode *inodep, struct file *filep) {
    printk(KERN_INFO "MyCharDev: Thiet bi da duoc mo\n");
    return 0;
}

static ssize_t dev_read(struct file *filep, char *buffer, size_t len, loff_t *offset) {
    size_t datalen = strlen(kernel_buffer);
    if (*offset >= datalen) return 0;
    if (copy_to_user(buffer, kernel_buffer, datalen)) return -EFAULT;
    *offset += datalen;
    return datalen;
}

static ssize_t dev_write(struct file *filep, const char *buffer, size_t len, loff_t *offset) {
    memset(kernel_buffer, 0, sizeof(kernel_buffer));
    if (len > 1023) len = 1023;
    if (copy_from_user(kernel_buffer, buffer, len)) return -EFAULT;
    printk(KERN_INFO "MyCharDev: Nhan tu User: %s\n", kernel_buffer);
    return len;
}

static int dev_release(struct inode *inodep, struct file *filep) {
    printk(KERN_INFO "MyCharDev: Thiet bi da dong\n");
    return 0;
}

static struct file_operations fops = {
    .open = dev_open,
    .read = dev_read,
    .write = dev_write,
    .release = dev_release,
};

static int __init mycdev_init(void) {
    // 1. Đăng ký Driver và lấy số Major động từ Kernel
    majorNumber = register_chrdev(0, DEVICE_NAME, &fops);

    // 2. TẠO CLASS: Tạo một mục trong /sys/class/
    myClass = class_create(THIS_MODULE, CLASS_NAME);

    // 3. TẠO DEVICE: Tự động tạo file /dev/my_char_dev
    myDevice = device_create(myClass, NULL, MKDEV(majorNumber, 0), NULL, DEVICE_NAME);

    printk(KERN_INFO "MyCharDev: Nap thanh cong Major %d\n", majorNumber);
    return 0;
}
static void __exit mycdev_exit(void) {
    device_destroy(myClass, MKDEV(majorNumber, 0));
    class_unregister(myClass);
    class_destroy(myClass);
    unregister_chrdev(majorNumber, DEVICE_NAME);
}

module_init(mycdev_init);
module_exit(mycdev_exit);
MODULE_LICENSE("GPL");
```

### Thêm vào Config.in chính
```bash
source "package/my_cdev/Config.in
```
### Cập nhật cấu hình
```bash
make menuconfig
```
tìm Target packages --> my_cdev

### Biên dịch Driver
```bash
make my_cdev
make
```
### Nạp img vào thẻ
```bash
sudo dd if=output/images/sdcard.img of=/dev/sdb bs=4M status=progress
sync
```
### Kiểm tra trên BBB
```bash

# Nap driver
modprobe my_cdev

# Kiem tra file thiet bi
ls -l /dev/my_char_dev

# Test ghi du lieu vao Driver
echo "Hello Driver" > /dev/my_char_dev

# Xem Driver da nhan duoc chua
dmesg | tail

# Test doc du lieu tu Driver
cat /dev/my_char_dev
```
### Kết quả
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/7cb39206-74bb-4da0-a366-0ff96d8a342f" />

