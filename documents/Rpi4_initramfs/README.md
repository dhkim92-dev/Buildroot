# Raspberry pi 4 initramfs 구성하기
------
## 머릿말
  많은 분들이 상용화 된 라즈베리파이를 이용하여 홈서버 등을 돌리시는데  
SD 카드가 언제 사망할 지 모른다는 불안감 속에 사용하시는 걸 보다가  
Buildroot tutorial을 쓰게되었습니다.  
일반적으로 임베디드 시스템은 365일 24시간 작동하는 경우가 많습니다.  
때문에 SDCard 와 같은 장치에 직접 Root File System을 동작시키지 않고
RAM 영역 일부 공간을 Root File System으로 이용합니다. 즉 모든 IO 작업이 RAM에서  
이루어지기 때문에 SDCard의 수명으로 인한 문제로부터 자유롭습니다.  
이 튜토리얼에서는 Raspberry Pi 4 기반으로 initramfs 를 작성하여 간단한 Linux 시스템을
구성해 볼 것입니다.

------
## 준비물
* Raspberry Pi 4
* SD Card
* Linux System(윈도우 유저는 WSL을 이용)
------
## 절차
1. [Buildroot](https://buildroot.org/download.html) Download
2. Buildroot 의존성 패키지 설치  
	* which
	* sed
	* make
	* binutils
	* build-essential(debian-based linux)
	* Development Tools(RHEL-based linux)
	* gcc
	* g++
	* bash
	* patch
	* gzip
	* bzip2
	* perl
	* tar
	* cpio
	* unzip
	* rsync
	* file
	* bc
	* wget
	* 상기 dependency 는 Ubuntu에서 sudo apt install <package name> 으로 설치 가능합니다.  
	```bash
	sudo apt install build-essential  
	```
	만일 Fedora, Rocky Linux, Redhat Linux 등을 이용중이라면 yum 또는 dnf 명령어로 설치 가능합니다.  
	```bash
	sudo dnf groupinstall "Development Tools" 
	sudo yum groupinstall "Development Tools"
	```
3. Buildroot Install  
다음과 같은 명령어로 다운로드한 buildroot 압축 파일을 해제합니다.  
```bash
# gz file을 다운로드한 경우
tar -zxvf buildroot-<year>-<month>.tar.gz
# xz file을 다운로드한 경우
tar -xvf buildroot-<year>-<month>.tar.xz
```
4. 다음 명령어로 자신이 원하는 작업 경로로 폴더 명을 변경한 뒤 해당 경로로  이동합니다.
```bash
mv buildroot-<year>-<month> path/to/where/you/want/buildroot
cd path/to/where/you/want/buildroot
```
5. 다음 명령어로 동적 라이브러리 경로를 초기화합니다. 
```
unset LD_LIBRARY_PATH
```
6. buildroot 구성 인터페이스를 실행합니다.
```bash
make menuconfig
```

7. Raspberry Pi 4 
다음 과정을 정확히 수행하십시오.
* 화면 하단 Load 탭으로 화살표를 이용하여 이동한 뒤 Enter 를 입력합니다.
* 경로 입력 화면에 다음과 같은 값을 입력합니다.  
buildroot경로/configs/raspberrypi4_64_defconfig
* Build options/Enable compiler cache 에 체크합니다
* Toolchain/C library 를 uClibc-ng 에서 glibc로 변경합니다.]
* Toolchain/Install glibc utilities 에 체크합니다.
* System Configuration
	- root password 설정
* Target packages에서 다음을 선택합니다.
	- BusyBox/Show packages that are also provided by busybox
	- Development Tools
		- grep
		- make
		- sed
	- Libraries
		- Crypto/CA-Certificates
		- Crypto/openssl support
			- 기본 옵션 유지 + openssl binary 선택
		- Networking/libcurl 
			- 기본 옵션 유지 + curl binary 선택
	- Networking applications
		- openssh 선택(기본 옵션 유지)
	- Shell and Utilities
		- bash 선택
	- System tools
		- docker-cli 선택
		- docker-compose 선택
		- docker-engine 선택
			- btrfs filesystem driver 선택
			- devicemapper filesystem driver 선택
			- vfs filesystem driver 선택
		- htop 선택
	- Text editors and viewers
		- vim 선택
* Filesystem Images
	- cpio the rootfile system (for use as an initial RAM filesystem) 에 체크합니다.
		- Compression method 를 gzip으로 설정합니다.
	- exact size를 400M 으로 변경합니다.
* Save 를 이용하여 path/to/buildroot/configs/name_defconfig 형태로 저장합니다

8. make name_defconfig 을 입력
9. make linux-menuconfig 입력
* CPU Power Management/CPU Frequency Scaling
	- performance 모드로 변경
* 상기 옵션을 활성화하지 않을 경우 IO 성능이 정상적으로 나오지 않습니다.

10. path/to/buildroot/board/raspberrypi4-64/genimage-raspberrypi4-64.cfg 를 다음과 같이 편집합니다.
```
image boot.vfat {
	vfat {
		files = {
			"bcm2711-rpi-4-b.dtb",
			"rpi-firmware/cmdline.txt",
			"rpi-firmware/config.txt",
			"rpi-firmware/fixup4.dat",
			"rpi-firmware/start4.elf",
			"rpi-firmware/overlays",
			"Image"
		}
	}

	size = 300M
}

image sdcard.img {
	hdimage {
	}

	partition boot {
		partition-type = 0xC
		bootable = "true"
		image = "boot.vfat"
	}

	partition rootfs {
		partition-type = 0x83
		image = "rootfs.ext4"
	}
}

```
11. 다음 명령어를 이용하여 작업 경로를 변경합니다 
```bash
mkdir -p path/to/buildroot/system/skeleton/etc/init.d
cd path/to/buildroot/system/skeleton
```
12. 위 경로는 기본 루트 파일 시스템에 들어갈 파일들을 세팅하는 장소입니다.
* 다음 파일들을 etc/init.d 에 추가합니다.
```bash
#!/bin/sh
# 파일명 path/to/buildroot/etc/init.d/S50prepare
#!/bin/sh

NAME=prepare
export DOCKER_RAMDISK=true #docker 가 오버레이 기능을 RAM 상에서 동작하게 하기 위한 환경변수 지정

do_start() {
        echo -n "Starting $NAME: "
        rdate time.bora.net # 현재 시간을 인터넷을 통해 동기화
		rm -rf /var/lib/docker
		mkdir -p /var/lib/docker
		mount -t tmpfs -o size=2G ext4 /var/lib/docker # docker 디렉토리를 RAM 영역을 ext4 포맷으로 만들어 이용
}

do_stop() {
        echo -n "Stopping $NAME: "
		umount /var/lib/docker
}

case "$1" in
        start)
                do_start
                ;;
        stop)
                do_stop
                ;;
        restart)
                do_stop
                sleep 1
                do_start
                ;;
	*)
                echo "Usage: $0 {start|stop|restart}"
                exit 1
esac
```
```bash
#!/bin/sh
#!파일명 path/to/buildroot/etc/init.d/S999Services
NAME=Services
export DOCKER_RAMDISK=true

do_start() {
	/etc/init.d/S60dockerd stop
	/etc/init.d/S60dockerd start
	mount /dev/mmcblk0p2 /media
	docker-compose -f /media/docker-compose.yml up -d
}

do_stop() {
	docker-compose -f /media/docker-compose.yml down
	umount /dev/mmcblk0p2 /media
}

case "$1" in
        start)
                do_start
                ;;
        stop)
                do_stop
                ;;
        restart)
                do_stop
                sleep 1
                do_start
                ;;
	*)
                echo "Usage: $0 {start|stop|restart}"
                exit 1
esac
```
13. 다음 명령어를 실행합니다.
```
chmod +x S50prepare
chmod +x S999Services
```
14. 다음 명령어로 작업 디렉토리를 변경합니다.
```bash
mkdir -p path/to/buildroot/etc/init.d/ssh
cd path/to/buildroot/etc/init.d/ssh
```

15. sshd_config 파일을 생성하여 다음과 같이 입력후 저장합니다.
```
#	$OpenBSD: sshd_config,v 1.104 2021/07/02 05:11:21 dtucker Exp $

# This is the sshd server system-wide configuration file.  See
# sshd_config(5) for more information.

# This sshd was compiled with PATH=/bin:/sbin:/usr/bin:/usr/sbin

# The strategy used for options in the default sshd_config shipped with
# OpenSSH is to specify options with their default value where
# possible, but leave them commented.  Uncommented options override the
# default value.

#Port 22
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::

#HostKey /etc/ssh/ssh_host_rsa_key
#HostKey /etc/ssh/ssh_host_ecdsa_key
#HostKey /etc/ssh/ssh_host_ed25519_key

# Ciphers and keying
#RekeyLimit default none

# Logging
#SyslogFacility AUTH
#LogLevel INFO

# Authentication:

#LoginGraceTime 2m
PermitRootLogin yes
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10

#PubkeyAuthentication yes

# The default is to check both .ssh/authorized_keys and .ssh/authorized_keys2
# but this is overridden so installations will only check .ssh/authorized_keys
AuthorizedKeysFile	.ssh/authorized_keys

#AuthorizedPrincipalsFile none

#AuthorizedKeysCommand none
#AuthorizedKeysCommandUser nobody

# For this to work you will also need host keys in /etc/ssh/ssh_known_hosts
#HostbasedAuthentication no
# Change to yes if you don't trust ~/.ssh/known_hosts for
# HostbasedAuthentication
#IgnoreUserKnownHosts no
# Don't read the user's ~/.rhosts and ~/.shosts files
#IgnoreRhosts yes

# To disable tunneled clear text passwords, change to no here!
#PasswordAuthentication yes
#PermitEmptyPasswords no

# Change to no to disable s/key passwords
#KbdInteractiveAuthentication yes

# Kerberos options
#KerberosAuthentication no
#KerberosOrLocalPasswd yes
#KerberosTicketCleanup yes
#KerberosGetAFSToken no

# GSSAPI options
#GSSAPIAuthentication no
#GSSAPICleanupCredentials yes

# Set this to 'yes' to enable PAM authentication, account processing,
# and session processing. If this is enabled, PAM authentication will
# be allowed through the KbdInteractiveAuthentication and
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication via KbdInteractiveAuthentication may bypass
# the setting of "PermitRootLogin without-password".
# If you just want the PAM account and session checks to run without
# PAM authentication, then enable this but set PasswordAuthentication
# and KbdInteractiveAuthentication to 'no'.
#UsePAM no

#AllowAgentForwarding yes
#AllowTcpForwarding yes
#GatewayPorts no
#X11Forwarding no
#X11DisplayOffset 10
#X11UseLocalhost yes
#PermitTTY yes
#PrintMotd yes
#PrintLastLog yes
#TCPKeepAlive yes
#PermitUserEnvironment no
#Compression delayed
#ClientAliveInterval 0
#ClientAliveCountMax 3
#UseDNS no
#PidFile /var/run/sshd.pid
#MaxStartups 10:30:100
#PermitTunnel no
#ChrootDirectory none
#VersionAddendum none

# no default banner path
#Banner none

# override default of no subsystems
Subsystem	sftp	/usr/libexec/sftp-server

# Example of overriding settings on a per-user basis
#Match User anoncvs
#	X11Forwarding no
#	AllowTcpForwarding no
#	PermitTTY no
#	ForceCommand cvs server
```
16. 다음 명령어로 buildroot 디텍터리로 이동합니다.
```bash
# make -j자신의 시피유 쓰레드 수 x 1.25 를 입력합니다.
# 예) 4코어 8쓰레드의 경우 make -j10
make -j10
```

17. 빌드 과정이 끝나면 path/to/buildroot/output/images 에 결과물이 생성됩니다.
18. SD카드를 buildroot를 실행한 컴퓨터에 연결합니다.
19. sudo fdisk -l 입력을 통해 SDCard의 /dev 값을 찾습니다. 주로 /dev/sdx 형태입니다. 
20. 다음 명령어를 실행하여 SDCard 에 이미지를 플래싱합니다.
```bash
sudo dd if=path/to/buildroot/output/images/sdcard.img of=/dev/sdx bs=512 status=progress
```
21. 플래싱 과정이 끝난 이후 SDCard 를 탈착 후 다시 연결합니다.
22. SDCard 의 boot 파티션에 저장된 config.txt 를 수정해야합니다.
```
# config.txt
#initramfs rootfs.cpio.gz => 이 부분을 주석 해제합니다.
```
23. boot 파티션에 path/to/buildroot/output/images/rootfs.cpio.gz 를 복사합니다.
24. SDCard를 탈착하여 라즈베리 파이에 연결합니다.
25. 부팅이 정상적으로 이루어졌다면 df -alh 를 입력하면 root filesystem(/)이 별도로  
검색이 되지 않습니다.
26. 다음 명령어로 docker 정상 동작 여부를 확인합니다.
```bash
docker run -it --rm ubuntu bash
apt update && apt upgrade -y
apt install vim
```
27. 위 과정이 정상적으로 실행되면 vim 이 켜집니다.
28. vim 을 종료한 이후 exit 명령어를 입력하여 docker container 에서 탈출합니다.
29. buildroot에 새로운 스크립트를 추가하여 RAM 상에서 동작하는 시스템을 구성 해보세요
