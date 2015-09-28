# 커널 컴파일

라즈비안에서 커널 컴파일 방법은 크게 두 가지가 있다. 라즈베리 파이에서 직접 컴파일 하는 것과 개발 PC에서 크로스 컴파일하는 방법이다. 전자는 편하긴 하지만 오래 걸리고, 후자는 약간 귀찮긴 하지만 PC성능에 따라 다르긴해도 아주 빨리 컴파일할 수 있다.

## 라즈베리 파이에서 직접 컴파일 하기

먼저 최신 버전의 라즈비안 [Raspbian](http://www.raspberrypi.org/downloads) 을 받아서 설치한다. 파이를 부팅시키고 인터넷과 연결되도록 설정한다.

리눅스 커널 소스를 다운 받는다. 시간이 다소 걸릴 수 있다.

```
$ git clone --depth=1 https://github.com/raspberrypi/linux
```

커널 컴파일에 필요한 추가 패키지를 설치한다.

```
$ sudo apt-get install bc
```

Configure the kernel - as well as the default configuration you may wish to [configure your kernel in more detail](configuring.md) or [apply patches from another source](patching.md) to add or remove required functionality:

라즈베리 파이 버전에 따라 아래 두 가지중 하나를 수행한다.

####라즈베리 파이 1 (또는 컴퓨트 모듈) 디폴드 빌드 설정

```
$ cd linux
$ KERNEL=kernel
$ make bcmrpi_defconfig
```

####라즈베리 파이 2 디폴드 빌드 설정
```
$ cd linux
$ KERNEL=kernel7
$ make bcm2709_defconfig
```

커널을 컴파일하고 커널과 모듈, 디바이스 트리 블롭(DTB)을 설치한다. 이 과정은 시간이 상당히 오래 걸린다. 

```
$ make zImage modules dtbs
$ sudo make modules_install
$ sudo cp arch/arm/boot/dts/*.dtb /boot/
$ sudo cp arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
$ sudo cp arch/arm/boot/dts/overlays/README /boot/overlays/
$ sudo scripts/mkknlimg arch/arm/boot/zImage /boot/$KERNEL.img
```

## 크로스 컴파일링

적절한 성능의 개발 PC에 리눅스와 크로스 컴파일 환경이 설치되어 있어야 한다. 우분투 리눅스를 기준으로 설명할텐데, 이는 라즈비안과 마찬가지로 데비안을 기반으로 하고 있어서 명령어가 비슷하기 때문이다.

You can either do this using VirtualBox (or VMWare) on Windows, or install it directly onto your computer. For reference you can follow instructions online [at Wikihow](http://www.wikihow.com/Install-Ubuntu-on-VirtualBox).

### 툴체인 설치

다음 명령어를 실행한다.

```
$ git clone https://github.com/raspberrypi/tools
```

You can then copy the toolchain to a common location such as `/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian`, and add `/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian/bin` to your $PATH in the .bashrc in your home directory.
For 64bit, use /tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin.
While this step is not strictly necessary, it does make it easier for later command lines!

### 리눅스 커널 소스 다운로드

To get the sources, refer to the original [GitHub](https://github.com/raspberrypi/linux) repository for the various branches.
```
$ git clone --depth=1 https://github.com/raspberrypi/linux
```

### 커널 소스 컴파일링

To build the sources for cross-compilation there may be extra dependencies beyond those you've installed by default with Ubuntu. If you find you need other things please submit a pull request to change the documentation.

커널 소스와 디바이스 트리 파일을 빌드하기 위해 아래 명령어를 실행한다. 이후 모든 make 명령에 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- 를 주고 있음을 주의하여 보기 바란다.

라즈베리 파이 1이나 컴퓨트 모듈인 경우:
```
$ cd linux
$ KERNEL=kernel
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcmrpi_defconfig
```
라즈베리 파이 2의 경우:
```
$ cd linux
$ KERNEL=kernel7
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2709_defconfig
```
그 다음 두 경우 모두 다음을 수행한다.
```
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs
```

팁!: 멀티프로세서 시스템에서 컴파일을 빠르게 하기 위해 ```-j n``` 옵션을 줄 수 있다. 여기에서 n은 병행 수행할 컴파일 작업의 수로 보통 프로세서 수x1.5 정도를 사용한다. 멀티코어 시스템에서 강력 추천함.

### SD카드에 빌드 경과물 직접 설치(복사)

Having built the kernel you need to copy it onto your Raspberry Pi and install the modules; this is best done directly using an SD card reader.

First use lsblk before and after plugging in your SD card to identify which one it is; you should end up with something like this:

```
sdb
   sdb1
   sdb2
```

If it is a NOOBS card you should see something like this:

```
sdb
  sdb1
  sdb2
  sdb3
  sdb5
  sdb6
```

In the first case `sdb1/sdb5` is the FAT partition, and `sdb2/sdb6` is the ext4 filesystem image (NOOBS).

SD카드의 각 파티션 중 boot영역과 라즈비안이 설치된 영역을 다음과 같이 마운트한다. 이 예에서는 sdb1이 boot영역이고, sdb2가 라즈비안 설치 영역인 경우이다. NOOBS 이미지를 사용하여 설치한 경우에 다른 위치를 사용해야 할 것이다.

```
$ mkdir mnt/fat32
$ mkdir mnt/ext4
$ sudo mount /dev/sdb1 mnt/fat32
$ sudo mount /dev/sdb2 mnt/ext4
```

먼저 모듈을 설치한다:

```
$ sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=mnt/ext4 modules_install
```
모듈이 설치될 위치를 INSTALL_MOD_PATH 변수로 지정하도록 되어 있다.

마지막으로 커널과 디바이스 트리 블롭을 SD카드로 복사한다. 복사하기 전에 이전 커널에 대해 백업을 권장한다.

```
$ sudo cp mnt/fat32/$KERNEL.img mnt/fat32/$KERNEL-backup.img
$ sudo scripts/mkknlimg arch/arm/boot/zImage mnt/fat32/$KERNEL.img
$ sudo cp arch/arm/boot/dts/*.dtb mnt/fat32/
$ sudo cp arch/arm/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
$ sudo cp arch/arm/boot/dts/overlays/README mnt/fat32/overlays/
$ sudo umount mnt/fat32
$ sudo umount mnt/ext4
```

이전 커널을 백업하지 않고, 새 커널을 다른 이름으로 복사하는 방법도 가능하다. 예를 들어 새 커널의 이름을 kernel-myconfig.img라고 하고 boot영역의 config.txt에 부팅할 커널의 이름을 새 커널 이름인 kernel-myconfig.img로 변경한다.

```
kernel=kernel-myconfig.img
```

This has the advantage of keeping your kernel separate from the kernel image managed by the system and any automatic update tools, and allowing you to easily revert to a stock kernel in the event that your kernel cannot boot.

SD카드를 뽑아서 라즈베리 파이에 꽂고 부팅한다.

## Links

Building / cross-compiling on/for other operating systems

- Pidora
- ArchLinux
- RaspBMC
- OpenELEC
