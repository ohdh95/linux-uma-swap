# linux-uma-swap

DGX Spark / Ubuntu 24.04 / aarch64에서 개발용 커널을 컴파일하고 적용하는 절차다. 아래 명령은 Bash에서 실행한다.

- 원본: <https://git.launchpad.net/~canonical-kernel/ubuntu/+source/linux-nvidia/+git/noble>
- 태그: `Ubuntu-nvidia-7.0-7.0.0-1019.19_24.04.2`
- 원본 커밋: `ae07646a606b3f18707246b0762464452cb40cbe`
- 배포판 버전: `7.0.0-1019.19~24.04.2` / 기존 실행 커널: `7.0.0-1019-nvidia`

개발 커널에는 `-uma-swap` 접미사를 붙여 별도로 설치한다. 배포판의 ABI 이름과 소스 Makefile의 버전은 다를 수 있으므로 실제 설치·드라이버 빌드에는 아래 `kernelrelease` 결과를 사용한다.

## 1. 빌드 도구 설치

```bash
sudo apt update
sudo apt install build-essential gcc-13 g++-13 bc bison flex libssl-dev \
    libelf-dev libncurses-dev libdw-dev zlib1g-dev dwarves pkg-config \
    cpio rsync kmod fakeroot dpkg-dev debhelper zstd lz4 python3 \
    openssl sbsigntool mokutil
```

GCC 13을 사용하고, BTF 설정은 기존 커널 설정대로 유지한다. 빌드 산출물은 저장소 밖의 `../build/linux-uma-swap`에 생성한다.

## 2. Secure Boot 서명 키 준비 — 최초 한 번

현재 Spark는 Secure Boot가 켜져 있다. 개발 커널과 NVIDIA 모듈에 같은 키를 사용한다. 개인 키는 저장소 밖에 보관한다.

```bash
UMA_SIGNING_DIR="$HOME/.local/share/uma-swap/keys"
mkdir -p "$UMA_SIGNING_DIR"
chmod 700 "$UMA_SIGNING_DIR"

if [ ! -e "$UMA_SIGNING_DIR/MOK.key" ] && [ ! -e "$UMA_SIGNING_DIR/MOK.crt" ]; then
    (umask 077
     openssl req -new -x509 -newkey rsa:3072 -sha256 -nodes -days 3650 \
         -subj '/CN=UMA swap development/' \
         -addext 'basicConstraints=critical,CA:FALSE' \
         -addext 'keyUsage=critical,digitalSignature' \
         -addext 'extendedKeyUsage=codeSigning' \
         -keyout "$UMA_SIGNING_DIR/MOK.key" \
         -out "$UMA_SIGNING_DIR/MOK.crt")
fi

test -f "$UMA_SIGNING_DIR/MOK.key" || exit 1
test -f "$UMA_SIGNING_DIR/MOK.crt" || exit 1
openssl x509 -in "$UMA_SIGNING_DIR/MOK.crt" -outform DER \
    -out "$UMA_SIGNING_DIR/MOK.der"
(umask 077
 cat "$UMA_SIGNING_DIR/MOK.key" "$UMA_SIGNING_DIR/MOK.crt" \
     > "$UMA_SIGNING_DIR/MOK.pem")

sudo mokutil --import "$UMA_SIGNING_DIR/MOK.der"
sudo reboot
```

재부팅할 때 MOK Manager에서 `Enroll MOK` → `Continue` → `Yes`를 선택하고 `mokutil --import`에서 설정한 일회용 비밀번호를 입력한다. 등록 후 기존 커널로 부팅하여 확인한다.

```bash
mokutil --sb-state
mokutil --test-key "$HOME/.local/share/uma-swap/keys/MOK.der"
```

이미 이 키를 등록했다면 생성·import·이 단계의 재부팅은 생략한다. Ubuntu가 자동 생성한 module-signing-only 키 대신, 커널 이미지에도 쓸 수 있도록 위에서 생성한 키를 사용한다.

## 3. 초기 설정

이 절은 출력 디렉터리를 처음 만들 때 실행한다. 이후 소스를 변경할 때는 설정을 다시 복사하지 말고 컴파일 절을 반복한다.

```bash
cd ~/Downloads/dgx-spark-source/linux-uma-swap
UMA_KERNEL_SRC="$(pwd -P)"
UMA_KERNEL_OUT="$(dirname "$UMA_KERNEL_SRC")/build/linux-uma-swap"
UMA_SIGNING_DIR="$HOME/.local/share/uma-swap/keys"
UMA_LOCALVERSION='-uma-swap'
mkdir -p "$UMA_KERNEL_OUT"
cp "$UMA_KERNEL_SRC/configs/spark.config" "$UMA_KERNEL_OUT/.config"

"$UMA_KERNEL_SRC/scripts/config" --file "$UMA_KERNEL_OUT/.config" \
    --set-str LOCALVERSION '' --disable LOCALVERSION_AUTO \
    --enable MODULE_SIG --enable MODULE_SIG_ALL --enable MODULE_SIG_SHA512 \
    --set-str MODULE_SIG_KEY "$UMA_SIGNING_DIR/MOK.pem" \
    --set-str SYSTEM_TRUSTED_KEYS '' --set-str SYSTEM_REVOCATION_KEYS ''

make -C "$UMA_KERNEL_SRC" O="$UMA_KERNEL_OUT" ARCH=arm64 \
    CC=gcc-13 HOSTCC=gcc-13 LOCALVERSION="$UMA_LOCALVERSION" olddefconfig
printf '%s\n' "$UMA_LOCALVERSION" > "$UMA_KERNEL_OUT/uma-localversion.txt"
```

배포판 인증서 경로를 비우고 지정한 개발용 MOK를 커널에 포함한다. 4 KiB 페이지 등 나머지 설정은 기존 설정을 유지한다.

## 4. 컴파일

새 셸에서 빌드를 다시 시작할 때도 이 절의 변수 설정부터 실행한다.

```bash
cd ~/Downloads/dgx-spark-source/linux-uma-swap
UMA_KERNEL_SRC="$(pwd -P)"
UMA_KERNEL_OUT="$(dirname "$UMA_KERNEL_SRC")/build/linux-uma-swap"
UMA_SIGNING_DIR="$HOME/.local/share/uma-swap/keys"
UMA_LOCALVERSION="$(cat "$UMA_KERNEL_OUT/uma-localversion.txt")"

make -C "$UMA_KERNEL_SRC" O="$UMA_KERNEL_OUT" ARCH=arm64 \
    CC=gcc-13 HOSTCC=gcc-13 LOCALVERSION="$UMA_LOCALVERSION" olddefconfig
make -C "$UMA_KERNEL_SRC" O="$UMA_KERNEL_OUT" -j"$(nproc)" ARCH=arm64 \
    CC=gcc-13 HOSTCC=gcc-13 LOCALVERSION="$UMA_LOCALVERSION" bindeb-pkg

UMA_KERNEL_RELEASE="$(cat "$UMA_KERNEL_OUT/include/config/kernel.release")"
printf '빌드한 커널: %s\n' "$UMA_KERNEL_RELEASE"
```

소스 Makefile 버전이 `7.0.14`이면 출력 이름은 `7.0.14-uma-swap`이 된다. 기존 `7.0.0-1019-nvidia`와 구분된다. 다른 실험 버전도 함께 설치하려면 초기 설정의 `UMA_LOCALVERSION`을 `-uma-swap-v2` 등으로 바꾼다.

## 5. 패키지 설치·커널 이미지 서명

```bash
sudo dpkg -i "$UMA_KERNEL_OUT/../linux-image-$UMA_KERNEL_RELEASE"_*.deb \
    "$UMA_KERNEL_OUT/../linux-headers-$UMA_KERNEL_RELEASE"_*.deb

sudo sbsign --key "$UMA_SIGNING_DIR/MOK.key" \
    --cert "$UMA_SIGNING_DIR/MOK.crt" \
    --output "/boot/vmlinuz-$UMA_KERNEL_RELEASE.signed" \
    "/boot/vmlinuz-$UMA_KERNEL_RELEASE"
sudo mv "/boot/vmlinuz-$UMA_KERNEL_RELEASE.signed" \
    "/boot/vmlinuz-$UMA_KERNEL_RELEASE"
sudo sbverify --cert "$UMA_SIGNING_DIR/MOK.crt" \
    "/boot/vmlinuz-$UMA_KERNEL_RELEASE"
```

`bindeb-pkg`의 image 패키지에는 커널과 함께 빌드한 모듈도 포함된다. 여기서는 `linux-libc-dev`를 설치하지 않는다. 커널 모듈은 `MODULE_SIG_ALL=y`와 위에서 지정한 키로 패키지 생성 중 서명한다.

아직 재부팅하지 말고 형제 디렉터리 `nvidia-uma-swap/README.md`에 따라 이 커널의 소스·출력 디렉터리를 대상으로 NVIDIA `580.173.02`를 빌드·서명·설치한다.

## 6. 재부팅·적용 확인

NVIDIA 모듈 설치까지 마친 후 실행한다.

```bash
sudo update-initramfs -u -k "$UMA_KERNEL_RELEASE"
sudo update-grub
sudo reboot
```

GRUB의 `Advanced options for Ubuntu`에서 이름에 `uma-swap`이 붙은 커널을 선택한다. 메뉴가 숨겨져 있으면 UEFI 부팅 중 Esc로 표시한다.

```bash
uname -r
getconf PAGE_SIZE
cat /proc/driver/nvidia/version
nvidia-smi
sudo modprobe nvidia_uvm
modinfo -n nvidia_uvm
```

`uname -r`이 빌드한 `$UMA_KERNEL_RELEASE`, 페이지 크기는 `4096`, NVIDIA 버전은 `580.173.02`인지 확인한다.

## 7. 기존 커널로 부팅

GRUB의 `Advanced options for Ubuntu`에서 `7.0.0-1019-nvidia`를 선택한다. 개발 커널을 별도 이름으로 설치하므로 기존 커널과 그 모듈은 유지된다.

## 참고

- [Ubuntu 커널 빌드](https://canonical-kernel-docs.readthedocs-hosted.com/how-to/develop-customise/build-kernel/)
- [Kbuild 출력 디렉터리](https://docs.kernel.org/kbuild/kbuild.html)
- [Linux 모듈 서명](https://docs.kernel.org/admin-guide/module-signing.html)
- [Ubuntu Secure Boot](https://documentation.ubuntu.com/security/security-features/platform-protections/secure-boot/)

## 실험 코드

기본 설정은 `configs/spark.config`에 보관한다. 커널의 기존 테스트는 `tools/testing/selftests/`에 포함되어 있다. GPU 메모리·fault·folio 실험 소스와 재현 스크립트는 [nvidia-uma-swap의 experiments](https://github.com/ohdh95/nvidia-uma-swap/tree/main/experiments)에 보관한다. 빌드 산출물과 측정 결과는 추적하지 않는다.
