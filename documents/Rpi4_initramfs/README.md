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
tar -zxvf buildroot-<year>-<month>.tar.gz
```
4. 다음 명령어로 자신이 원하는 작업 경로로 이동합니다.
```bash
mv buildroot-<year>-<month>.tar.gz path/to/where/you/want/buildroot
```
5. 다음 명령어로 동적 라이브러리 경로를 초기화합니다. 
```
unset LD_LIBRARY_PATH
```
6. buildroot 구성 인터페이스를 실행합니다.
```bash
make menuconfig
```

