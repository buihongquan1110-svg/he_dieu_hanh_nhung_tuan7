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

## BÀI 2: Mở rộng driver trên cho phép tương tác ngoại vi GPIO và Viết chương trình C ở lớp User Space
### Cấu trúc thư viện
```bash
my_gpio_driver/
├── Config.in           # Cấu hình menu cho Buildroot
├── my_gpio_driver.mk   # Kịch bản build (Kernel module + Generic package)
└── src/
    ├── my_gpio_driver.c # Mã nguồn Kernel Driver (ioremap)
    ├── my_gpio_app.c    # Mã nguồn App User Space (Blink LED)
    └── Makefile         # Makefile biên dịch chéo (Cross-compile)
```
### Nội dung các file chi tiết
#### 1. File `"Config.in"`
```bash
nano package/my_gpio_driver/Config.in
```
```bash
config BR2_PACKAGE_MY_GPIO_DRIVER
    bool "my_gpio_driver"
    depends on BR2_LINUX_KERNEL
    help
      Driver dieu khien LED qua GPIO 60 (P9_12) su dung ioremap.
      Bao gom:
      - Kernel Module: my_gpio_driver.ko
      - User-space App: my_gpio_app

      Truy cap tai /dev/my_gpio_dev sau khi load module.
```
#### 2. File `"my_gpio_driver.mk"`
```bash
nano package/my_gpio_driver/my_gpio_driver.mk
```
```bash
MY_GPIO_DRIVER_VERSION = 1.0
MY_GPIO_DRIVER_SITE = $(TOPDIR)/package/my_gpio_driver/src
MY_GPIO_DRIVER_SITE_METHOD = local

define MY_GPIO_DRIVER_BUILD_CMDS
        $(MAKE) $(LINUX_MAKE_FLAGS) -C $(@D) \
                LINUX_DIR=$(LINUX_DIR) \
                PWD=$(@D) \
                CC="$(TARGET_CC)" \
                all
endef

define MY_GPIO_DRIVER_INSTALL_TARGET_CMDS
        $(INSTALL) -D -m 0755 $(@D)/my_gpio_app $(TARGET_DIR)/usr/bin/my_gpio_app
endef

$(eval $(kernel-module))
$(eval $(generic-package))
```

#### 3. File `"my_gpio_driver.c"`
```bash
nano package/my_gpio_driver/src/my_gpio_driver.c
```
```bash
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/io.h>
#include <linux/device.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "my_gpio_dev"
#define CLASS_NAME  "my_gpio_class"

// GPIO1 Base Address cho AM335x
#define GPIO1_BASE         0x4804C000
#define GPIO1_SIZE         0x2000
#define GPIO_OE            0x134
#define GPIO_SETDATAOUT    0x194
#define GPIO_CLEARDATAOUT  0x190
#define GPIO_DATAIN        0x138

// GPIO 60 là GPIO1_28 -> Bit 28
#define GPIO_60_BIT        (1 << 28)

static int majorNumber;
static struct class* gpioClass  = NULL;
static struct device* gpioDevice = NULL;
static void __iomem *gpio1_base_addr;

static int dev_open(struct inode *inodep, struct file *filep) {
    return 0;
}

// Đọc trạng thái chân GPIO 60
static ssize_t dev_read(struct file *filep, char *buffer, size_t len, loff_t *offset) {
    uint32_t reg_val;
    char state;
    if (*offset > 0) return 0;

    reg_val = ioread32(gpio1_base_addr + GPIO_DATAIN);
    state = (reg_val & GPIO_60_BIT) ? '1' : '0';

    if (copy_to_user(buffer, &state, 1) != 0) return -EFAULT;
    *offset = 1;
    return 1;
}

// Ghi trạng thái (Bật/Tắt) LED
static ssize_t dev_write(struct file *filep, const char *buffer, size_t len, loff_t *offset) {
    char k_buf;
    if (copy_from_user(&k_buf, buffer, 1) != 0) return -EFAULT;

    if (k_buf == '1') {
        iowrite32(GPIO_60_BIT, gpio1_base_addr + GPIO_SETDATAOUT);
    } else if (k_buf == '0') {
        iowrite32(GPIO_60_BIT, gpio1_base_addr + GPIO_CLEARDATAOUT);
    }
    return len;
}

static struct file_operations fops = {
    .open = dev_open,
    .read = dev_read,
    .write = dev_write,
};

static int __init gpio_driver_init(void) {
    majorNumber = register_chrdev(0, DEVICE_NAME, &fops);
    gpioClass = class_create(THIS_MODULE, CLASS_NAME);
    gpioDevice = device_create(gpioClass, NULL, MKDEV(majorNumber, 0), NULL, DEVICE_NAME);

    // Ánh xạ bộ nhớ
    gpio1_base_addr = ioremap(GPIO1_BASE, GPIO1_SIZE);

    // Cấu hình Output: Clear bit 28 trong thanh ghi OE (Output Enable)
    uint32_t oe_val = ioread32(gpio1_base_addr + GPIO_OE);
    oe_val &= ~GPIO_60_BIT;
    iowrite32(oe_val, gpio1_base_addr + GPIO_OE);

    printk(KERN_INFO "GPIO Driver: Loaded. GPIO 60 ready.\n");
    return 0;
}

static void __exit gpio_driver_exit(void) {
    iounmap(gpio1_base_addr);
    device_destroy(gpioClass, MKDEV(majorNumber, 0));
    class_unregister(gpioClass);
    class_destroy(gpioClass);
    unregister_chrdev(majorNumber, DEVICE_NAME);
}

module_init(gpio_driver_init);
module_exit(gpio_driver_exit);
MODULE_LICENSE("GPL");
```

### 4. File `"my_gpio_app.c"`
```bash
nano package/my_gpio_driver/src/my_gpio_app.c
```

```bash
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    int fd = open("/dev/my_gpio_dev", O_RDWR);
    if (fd < 0) return 1;

    int freq = (argc > 1) ? atoi(argv[1]) : 500;
    printf("Blinking GPIO 60 moi %d ms...\n", freq);

    while(1) {
        write(fd, "1", 1);
        usleep(freq * 1000);
        write(fd, "0", 1);
        usleep(freq * 1000);
    }
    close(fd);
    return 0;
}
```

#### 5. File Makefile
```bash
nano package/my_gpio_driver/src/Makefile
```

```bash
obj-m += my_gpio_driver.o

all:
        $(MAKE) -C $(LINUX_DIR) M=$(PWD) modules
        $(CC) my_gpio_app.c -o my_gpio_app

clean:
        $(MAKE) -C $(LINUX_DIR) M=$(PWD) clean
        rm -f my_gpio_app
```

### Thêm vào Config.in chính
- Lệnh `"nano package/Config.in"`
- Kéo xuống cuối file, thêm dòng sau vào trước dòng cuối cùng:
```bash
source "package/my_gpio_driver/Config.in"
```
### Cập nhật cấu hình
```bash
make menuconfig
```
tìm Target packages --> my_gpio_driver

### Biên dịch Driver
```bash
make my_gpio_driver
make
```
### Nạp lại img vào thẻ
```bash
sudo dd if=output/images/sdcard.img of=/dev/sdb bs=4M status=progress
sync
```
### Kiểm tra trên BBB
- Nạp Driver
```bash
# Nạp module
modprobe my_gpio_driver

# Kiểm tra xem module đã chạy chưa
lsmod | grep my_gpio_driver

# Kiểm tra file thiết bị (/dev) đã xuất hiện chưa
ls -l /dev/my_gpio_dev
```
- Kiểm tra log của kernel
```bash
dmesg | tail -n 10
```

- Chạy ứng dụng

```bash
# Chạy với tần số mặc định (500ms)
my_gpio_app

# Hoặc chạy với tần số nhanh hơn (100ms)
my_gpio_app 100
```

### Kết quả
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6070fe8d-1888-44a8-b6a2-7ae5badfaf2d" />

### Video demo kết quả


https://github.com/user-attachments/assets/aa0dade6-7484-4c6d-850e-e8dad2334d77



