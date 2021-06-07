---
layout: post
title: "Arch Linux 설치과정 정리"
categories: study
published: true
---

## 서문

이 글은 [Arch Linux](https://www.archlinux.org)를 설치해 보며 나름대로 과정을 정리한 것이다. 기본적인 과정은 공식 위키의 [Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide)를 그대로 따랐다.

다양한 환경이나 설치 중 발생할 수 있는 여러 문제에 대해서 기술하고 있지는 않다. whjeon 님의 [아치 리눅스 설치 가이드](https://whjeon.com/arch-install)을 참고하는 쪽을 추천한다.

## 설치 전 준비

[다운로드 페이지](https://www.archlinux.org/download/)의 적당한 미러에서 이미지를 내려받는다. [Rufus](https://rufus.ie)를 이용해 이미지를 USB 드라이브에 구워 부팅 가능하도록 만들었다.

만든 설치 디스크를 이용해 부팅하여 Arch Linux Install medium을 선택해 설치를 시작한다.

## 파일 시스템

### 파티셔닝

여러 방법으로 할 수 있지만 나는 가이드에 나온 `fdisk`를 이용했다.

`fdisk -l`로 현재 파티션 상태를 볼 수 있다. 내 경우 리눅스를 설치할 SSD가 `sda`라는 device로 인식이 되는 것을 확인하였다. 여기서는 UEFI with GPT로 아래와 같이 파티셔닝을 진행한다.

- sda1 - EFI System: 512MB
- sda2 - Linux swap: 16GB
- sda3 - Linux filesystem(ext4): 나머지

```bash
fdisk /dev/sda
```

- m: 메뉴얼
- d: 파티션 지우기
- n: 파티션 생성
- p: 만들어진 파티션 확인
- t: 각 파티션에 타입을 설정
- w: 저장

### 포멧

아래와 같이 파티션별 파일시스템에 맞게 포멧, swap을 켜 준다.

```bash
mkfs.fat -F32 /dev/sda1

mkswap /dev/sda2
swapon /dev/sda2

mkfs.ext4 /dev/sda3
```

### 마운트

루트 볼륨과 부트 파티션 등을 마운트해준다.

```bash
mount /dev/sda3 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

## 설치

### 인터넷 연결 확인

유선 인터넷으로 진행했다. 기본적으로 잡힐 것이다. ping을 날려 확인해 준다.

```bash
ping archlinux.org
```

### 시스템 시간 설정 및 확인

```bash
timedatectl set-ntp true
timedatectl status
```

### 미러 설정

패키지를 좀더 빠르게 다운로드하기 위해서 가까운 서버를 `/etc/pacman.d/mirrorlist`에 추가해준다. 윗줄에 있을 수록 우선순위가 높다. [미러 리스트](https://www.archlinux.org/mirrorlist/)를 참고하자. 나는 아래 서버만 추가해 주었다.

```
Server = <http://ftp.harukasan.org/archlinux/$repo/os/$arch>
```

### 필수 패키지 설치

base, linux, linux-firmware를 설치한다.

```bash
pacstrap /mnt base linux linux-firmware
```

기타 필요한 패키지도 추가로 설치해준다. 내가 설치한 것들은 아래와 같다. 나중에 필요할 때 설치해도 무방하다.

- iproute2
- neovim
- sudo
- dhcpcd
- man-db
- man-pages
- base-devel
- texinfo

## 시스템 설정

### 파일시스템

현재 파일시스템 테이블 정보를 저장해놓는다.

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

루트 디렉토리를 새 시스템으로 변경한다.

```bash
arch-chroot /mnt
```

### 타임존

타임존을 서울로 설정하고 hardware clock을 맞춘다.

```bash
ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime
hwclock --systohc
```

### 로케일

로케일 설정을 위해 `/etc/locale.gen` 을 열어 `en_US.UTF-8 UTF-8`과 필요한 locale의 주석을 해제한다. 나는 `ko_KR.UTF-8 UTF-8` 역시 주석 해재하였다. 그리고 locale generate를 해 준다.

```bash
locale-gen
```

`/etc/locale.conf`를 생성하여 로케일 설정을 적어준다. 나는 아래만 적어주었으며 로케일 설정에 관한 자세한 내용은 [여기](http://coffeenix.net/doc/misc/locale.html)를 참고.

```bash
LANG=en_US.UTF-8
```

### 네트워크 설정

`/etc/hostname`를 생성하고 hostname을 적어준다.

```bash
juoarch
```

`/etc/hosts`를 작성한다. 아래와 같이 작성했다.

```bash
127.0.0.1    localhost
::1          localhost
```

재부팅 후에도 `dhcpcd` 서비스를 자동 실행하여 랜선을 통한 인터넷이 가능하도록 한다.

```bash
systemctl enable dhcpcd
```

### 비밀번호

루트 비밀번호를 설정한다.

```bash
passwd
```

### 부트 로더 설치

[여러 가지 부트 로더](https://wiki.archlinux.org/index.php/Arch_boot_process#Boot_loader)가 있지만 여기서는 [systemd-boot](https://wiki.archlinux.org/index.php/Systemd-boot)를 사용한다.

```bash
bootctl install
```

`/boot/loader/loader.conf`를 수정해준다. 설정 내용은 [이 곳](https://wiki.archlinux.org/index.php/Systemd-boot#Loader_configuration)을 참고했다.

```bash
default arch.conf
timeout 3
editor yes (부팅 확인 후 no로 수정)
console-mode max
```

이제 로더를 추가한다. 위에서 기본 로더로 `arch.conf` 를 사용하도록 설정해 주었다. 이제 이 로더를 `/boot/loader/entries/arch.conf`로 추가한다.

```bash
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root=/dev/sda3 rw
```

`initrd`의 파일은 `/boot/loader`에 존재한다.

`options`는 위와 같이 장치의 경로를 지정해줄 수도 있지만 partition label을 써 줄 수도 있다. ArchWiki의 예제가 이 방식으로 되어 있다. 하지만 이것들이 변경될 경우를 대비해서 PARTUUID를 지정해줄 수도 있는데 방법은 [여기](https://whjeon.com/arch-install/#patisyeon_gyehoeg) 참고.

Intel CPU의 microcode를 로딩하기 위해 [여기](https://wiki.archlinux.org/index.php/Microcode)를 참고해서 `intel-ucode` package를 설치하고 `arch.conf`에 한 줄을 추가했다.

```
initrd /intel-ucode.img
```

`systemd` package가 업데이트될 때마다 `systemd-boot`도 자동 업데이트를 해 주기 위해 pacman hook도 추가해줬다. [Automatic update](https://wiki.archlinux.org/index.php/Systemd-boot#Updating_the_EFI_boot_manager)부분을 보자.

## 재부팅

`chroot` 환경을 나가고 재부팅한다. 이제 `root`계정으로 로그인하면 리눅스를 사용할 수 있다. `pacman`을 한 번 업데이트한다.

```bash
exit
reboot
sudo pacman -Syu
```

## 사용자 계정 추가

User 계정을 추가하고 비밀번호를 설정한다.

```bash
useradd -m -g users -G wheel 계정명
passwd 계정명
```

Shell은 지금은 기본적으로 설치되어 있는 `bash`를 사용하도록 특별히 지정하지 않았다.

`wheel` 그룹의 유저가 `sudo`를 사용할 수 있도록 아래 명령어를 이용한다. 그러면  `/etc/sudoers.d`가 열리게 된다.

```bash
EDITOR=nvim visudo
```

`%wheel ALL=`로 시작하는 줄 중 자신에게 필요한 부분을 주석 해제한다. 여기서는 아래만 주석 해제하여 명령은 사용할 수 있되 비밀번호는 필요하도록 하였다.

```
%wheel ALL=(ALL) ALL
```

## Zsh 설치

[zsh](https://wiki.archlinux.org/index.php/Zsh)를 `bash` 대신 사용해 보자. Package 설치 후 `zsh`를 실행하여 설정을 진행한다. 완료하면 설정이 반영된 `.zshrc`가 생성된다.

```bash
sudo pacman -S zsh zsh-completions
zsh
```

현재 사용가능한 쉘을 확인하고 기본 사용 쉘을 변경한다.

```bash
chsh -l
chsh -s /usr/bin/zsh
```

플러그인이나 각종 테마를 사용하기 위해 [oh-my-zsh](https://github.com/ohmyzsh/ohmyzsh)도 설치한다.

## GUI 설정

### X Window 설치

xorg group package를 설치한다.

```bash
sudo pacman -S xorg
```

여기서는 DE는 깔지 않는다. 대신 Openbox를 깔아보겠다.

[https://wiki.archlinux.org/index.php/openbox](https://wiki.archlinux.org/index.php/openbox)

```bash
sudo pacman -S openbox obconf ttf-dejavu xorg-xinit
```

xinit 설정을 만든다.

```bash
cp /etc/X11/xinit/xinitrc ~/.xinitrc
nvim .xinitrc
```

맨 아래 기본으로 실행되는 프로그램을 다 지우고 아래 줄을 추가한다.

```bash
exec openbox-session
```

아래 명령어로 X를 실행시킨다.

```bash
startx
```

로그인할 때 GUI가 자동으로 시작되게 하려면 [여기](https://wiki.archlinux.org/index.php/Xinit#Autostart_X_at_login)를 참고하자.

### 메뉴 설정

X 내에서 마우스 오른쪽 버튼을 누르면 메뉴를 볼 수 있다. 여기서는 [Terminator](https://wiki.archlinux.org/index.php/Terminator)를 설치 후 메뉴에 추가해 본다.

```bash
sudo pacman -S terminator
```

Openbox 메뉴에 terminator를 추가하려면 `/etc/xdg/openbox/menu.xml`을 직접 수정하거나 `~/.config/openbox/menu.xml`로 파일을 복사할 수도 있다. 설치된 패키지를 바탕으로 메뉴를 자동으로 수정해주거나 수정을 위한 UI를 제공하는 패키지도 있다.

### 한글 입력

한글 입력을 위해 [IBus](https://wiki.archlinux.org/index.php/IBus)를 설치한다. 설정 메뉴에 진입해 Input Method 탭에서 한글만 남긴다.

```bash
sudo pacman -S ibus ibus-hangul
ibus-setup
```

`.zshrc` 또는 `.bashrc` 또는 `.xinitrc`에 아래를 추가한다.

```bash
export GTK_IM_MODULE=ibus
export XMODIFIERS=@im=ibus
export QT_IM_MODULE=ibus
```

X 시작시 ibus daemon을 실행하도록 `.config/openbox/autostart`를 수정한다.

```bash
ibus-daemon -drx
```

### 기타 패키지

기타 필요한 패키지를 설치한다.

```bash
sudo pacman -S noto-fonts-cjk firefox
```

## SSH 설정

[OpenSSH](https://wiki.archlinux.org/index.php/OpenSSH)를 설치하고 서비스를 등록한다. 이제 이 서버로 `sshd` 명령을 이용해 접속할 수 있다.

```bash
sudo pacman -S openssh
sudo systemctl enable sshd
sudo systemctl start sshd
```

`/etc/ssh/sshd_config` 설정을 수정한다. 여기서는 비밀번호 로그인을 비활성화하고 등록된 SSH key를 이용해서만 로그인할 수 있도록 설정하는 법을 소개한다.

```
PubkeyAuthentication yes
PasswordAuthentication no
```

클라이언트에서 `ssh-keygen`으로 키를 생성한다. `~/ssh`에 공개키와 개인키가 생성되었을 것이다. SSH 서버에 퍼블릭 키를 복사한다. 아래 명령을 이용하면 간단하다.

```bash
ssh-copy-id 계정@서버_주소
```

인증이 완료되면 서버의 `ssh/authorized_key`에 public key가 복사된 것을 볼 수 있다.
