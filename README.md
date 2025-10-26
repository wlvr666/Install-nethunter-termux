# install-nethunter-termux
install-nethunter-termux with sha256sums verified 


#!/data/data/com.termux/files/usr/bin/bash -e

VERSION=2025.3
BASE_URL=https://kali.download/nethunter-images/current/rootfs
USERNAME=kali

function unsupported_arch() {
    printf "${red}"
    echo "[*] Unsupported Architecture\n\n"
    printf "${reset}"
    exit
}

function ask() {
    while true; do
        if [ "${2:-}" = "Y" ]; then
            prompt="Y/n"
            default=Y
        elif [ "${2:-}" = "N" ]; then
            prompt="y/N"
            default=N
        else
            prompt="y/n"
            default=
        fi

        printf "${light_cyan}\n[?] "
        read -p "$1 [$prompt] " REPLY

        if [ -z "$REPLY" ]; then
            REPLY=$default
        fi

        printf "${reset}"

        case "$REPLY" in
            Y*|y*) return 0 ;;
            N*|n*) return 1 ;;
        esac
    done
}

function get_arch() {
    printf "${blue}[*] Checking device architecture ..."
    case $(getprop ro.product.cpu.abi) in
        arm64-v8a)
            SYS_ARCH=arm64
            ;;
        armeabi|armeabi-v7a)
            SYS_ARCH=armhf
            ;;
        *)
            unsupported_arch
            ;;
    esac
}

function set_strings() {
    echo && echo ""
    if [[ ${SYS_ARCH} == "arm64" ]]; then
        echo "[1] NetHunter ARM64 (full)"
        echo "[2] NetHunter ARM64 (minimal)"
        echo "[3] NetHunter ARM64 (nano)"
        read -p "Enter the image you want to install: " wimg
        case $wimg in
            1) wimg="full" ;;
            2) wimg="minimal" ;;
            3) wimg="nano" ;;
            *) wimg="full" ;;
        esac
    elif [[ ${SYS_ARCH} == "armhf" ]]; then
        echo "[1] NetHunter ARMhf (full)"
        echo "[2] NetHunter ARMhf (minimal)"
        echo "[3] NetHunter ARMhf (nano)"
        read -p "Enter the image you want to install: " wimg
        case $wimg in
            1) wimg="full" ;;
            2) wimg="minimal" ;;
            3) wimg="nano" ;;
            *) wimg="full" ;;
        esac
    fi

    CHROOT=kali-${SYS_ARCH}
    IMAGE_NAME=kali-nethunter-rootfs-${wimg}-${SYS_ARCH}.tar.xz
    SHA_NAME=${IMAGE_NAME}SHA256SUMS
}

function prepare_fs() {
    unset KEEP_CHROOT
    if [ -d ${CHROOT} ]; then
        if ask "Existing rootfs directory found. Delete and create a new one?" "N"; then
            rm -rf ${CHROOT}
        else
            KEEP_CHROOT=1
        fi
    fi
}

function cleanup() {
    if [ -f "${IMAGE_NAME}" ]; then
        if ask "Delete downloaded rootfs file?" "N"; then
            rm -f "${IMAGE_NAME}"
            rm -f "${SHA_NAME}"
        fi
    fi
}

function check_dependencies() {
    printf "${blue}\n[*] Checking package dependencies...${reset}\n"
    apt update -y &> /dev/null
    for i in proot tar axel; do
        if command -v $i &> /dev/null; then
            echo "  $i is OK"
        else
            printf "Installing ${i}...\n"
            apt install -y $i || {
                printf "${red}ERROR: Failed to install packages.\n Exiting.\n${reset}"
                exit
            }
        fi
    done
    apt upgrade -y
}

function get_url() {
    ROOTFS_URL="${BASE_URL}/${IMAGE_NAME}"
    SHA_URL="${BASE_URL}/SHA256SUMS"
}

function get_rootfs() {
    unset KEEP_IMAGE
    if [ -f "${IMAGE_NAME}" ]; then
        if ask "Existing image file found. Delete and download a new one?" "N"; then
            rm -f "${IMAGE_NAME}"
        else
            printf "${yellow}[!] Using existing rootfs archive${reset}\n"
            KEEP_IMAGE=1
            return
        fi
    fi
    printf "${blue}[*] Downloading rootfs...${reset}\n\n"
    get_url
    axel -o "${IMAGE_NAME}" "${ROOTFS_URL}" || wget --continue "${ROOTFS_URL}"
}

function verify_sha() {
    if [ -z $KEEP_IMAGE ]; then
        printf "\n${blue}[*] Verifying integrity of rootfs...${reset}\n\n"

        if [ -f "SHA256SUMS" ]; then
            # Try to find the checksum line (with or without version number)
            SHA_LINE=$(grep "rootfs-${wimg}-${SYS_ARCH}.tar.xz" "SHA256SUMS" | head -1)

            if [ -n "$SHA_LINE" ]; then
                EXPECTED_HASH=$(echo "$SHA_LINE" | awk '{print $1}')
                ACTUAL_HASH=$(sha256sum "$IMAGE_NAME" | awk '{print $1}')

                if [ "$EXPECTED_HASH" = "$ACTUAL_HASH" ]; then
                    printf "${green}[✓] Checksum verification passed!${reset}\n"
                else
                    printf "${red}[✗] Rootfs corrupted. Please run this installer again\n${reset}"
                    exit 1
                fi
            else
                echo "[!] Image not found in SHA256SUMS file. Skipping verification..."
            fi
        else
            echo "[!] SHA256SUMS file not found. Skipping verification..."
        fi
    fi
}

function get_sha() {
    if [ -z $KEEP_IMAGE ]; then
        printf "\n${blue}[*] Getting SHA ... ${reset}\n\n"
        get_url
        if [ -f "${SHA_NAME}" ]; then
            rm -f "${SHA_NAME}"
        fi
        if curl --output /dev/null --silent --head --fail "${SHA_URL}"; then
            wget --continue "${SHA_URL}"
        else
            echo "[!] SHA_URL does not exist. Skipping download."
        fi
    fi
}

function extract_rootfs() {
    if [ -z $KEEP_CHROOT ]; then
        printf "\n${blue}[*] Extracting rootfs... ${reset}\n\n"
        proot --link2symlink tar -xf "$IMAGE_NAME" 2> /dev/null || :
    else
        printf "${yellow}[!] Using existing rootfs directory${reset}\n"
    fi
}

function create_launcher() {
    NH_LAUNCHER=${PREFIX}/bin/nethunter
    NH_SHORTCUT=${PREFIX}/bin/nh
    cat > "$NH_LAUNCHER" <<- EOF
#!/data/data/com.termux/files/usr/bin/bash -e
cd \${HOME}
unset LD_PRELOAD
if [ ! -f $CHROOT/root/.version ]; then
    touch $CHROOT/root/.version
fi

user="$USERNAME"
home="/home/\$user"
start="sudo -u kali /bin/bash"

if ! grep -q "kali" ${CHROOT}/etc/passwd 2>/dev/null || [[ "\$#" != "0" && ("\$1" == "-r" || "\$1" == "-R") ]]; then
    user="root"
    home="/\$user"
    start="/bin/bash --login"
    if [[ "\$#" != "0" && ("\$1" == "-r" || "\$1" == "-R") ]]; then
        shift
    fi
fi

cmdline="proot \\
        --link2symlink \\
        -0 \\
        -r $CHROOT \\
        -b /dev \\
        -b /proc \\
        -b /sdcard \\
        -b $CHROOT\$home:/dev/shm \\
        -w \$home \\
           /usr/bin/env -i \\
           HOME=\$home \\
           PATH=/usr/local/sbin:/usr/local/bin:/bin:/usr/bin:/sbin:/usr/sbin \\
           TERM=\$TERM \\
           LANG=C.UTF-8 \\
           \$start"

cmd="\$@"
if [ "\$#" == "0" ]; then
    exec \$cmdline
else
    \$cmdline -c "\$cmd"
fi
EOF

    chmod 700 "$NH_LAUNCHER"
    ln -sf "${NH_LAUNCHER}" "${NH_SHORTCUT}"
}

function create_kex_launcher() {
    KEX_LAUNCHER=${CHROOT}/usr/bin/kex
    mkdir -p $(dirname "$KEX_LAUNCHER")
    cat > "$KEX_LAUNCHER" <<- 'EOF'
#!/bin/bash
function start-kex() {
    if [ ! -f ~/.vnc/passwd ]; then
        passwd-kex
    fi
    USR=$(whoami)
    SCREEN=":1"
    [ "$USR" = "root" ] && SCREEN=":2"
    export USER="$USR"
    vncserver "$SCREEN" -localhost no
}

function stop-kex() {
    vncserver -kill :1 2>/dev/null
    vncserver -kill :2 2>/dev/null
}

function passwd-kex() {
    vncpasswd
}

function status-kex() {
    vncserver -list
}

case "$1" in
    start) start-kex ;;
    stop) stop-kex ;;
    status) status-kex ;;
    passwd) passwd-kex ;;
    *) start-kex; status-kex ;;
esac
EOF
    chmod 700 "$KEX_LAUNCHER"
}

function fix_profile_bash() {
    if [ -f ${CHROOT}/root/.bash_profile ]; then
        sed -i '/if/,/fi/d' "${CHROOT}/root/.bash_profile"
    fi
}

function fix_sudo() {
    chmod +s $CHROOT/usr/bin/sudo 2>/dev/null || :
    chmod +s $CHROOT/usr/bin/su 2>/dev/null || :
    echo "kali    ALL=(ALL:ALL) ALL" > $CHROOT/etc/sudoers.d/kali 2>/dev/null || :
    echo "Set disable_coredump false" > $CHROOT/etc/sudo.conf 2>/dev/null || :
}

function fix_uid() {
    USRID=$(id -u)
    GRPID=$(id -g)
    nh -r usermod -u $USRID kali 2>/dev/null || :
    nh -r groupmod -g $GRPID kali 2>/dev/null || :
}

function print_banner() {
    clear
    printf "${blue}##################################################\n"
    printf "${blue}##                                              ##\n"
    printf "${blue}##  88      a8P         db        88        88  ##\n"
    printf "${blue}##  88    .88'         d88b       88        88  ##\n"
    printf "${blue}##  88   88'          d8''8b      88        88  ##\n"
    printf "${blue}##  88 d88           d8'  '8b     88        88  ##\n"
    printf "${blue}##  8888'88.        d8YaaaaY8b    88        88  ##\n"
    printf "${blue}##  88P   Y8b      d8''''''''8b   88        88  ##\n"
    printf "${blue}##  88     '88.   d8'        '8b  88        88  ##\n"
    printf "${blue}##  88       Y8b d8'          '8b 888888888 88  ##\n"
    printf "${blue}##                                              ##\n"
    printf "${blue}####  ############# NetHunter ####################${reset}\n\n"
}

red='\033[1;31m'
green='\033[1;32m'
yellow='\033[1;33m'
blue='\033[1;34m'
light_cyan='\033[1;96m'
reset='\033[0m'

cd "$HOME"
print_banner
get_arch
set_strings
prepare_fs
check_dependencies
get_rootfs
get_sha
verify_sha   # Add this as a separate call
extract_rootfs
create_launcher
cleanup

printf "\n${blue}[*] Configuring NetHunter for Termux ...\n"
fix_profile_bash
fix_sudo
create_kex_launcher
fix_uid

print_banner
printf "${green}[=] Kali NetHunter for Termux installed successfully${reset}\n\n"
printf "${green}[+] To start Kali NetHunter, type:${reset}\n"
printf "${green}[+] nethunter             # To start NetHunter CLI${reset}\n"
printf "${green}[+] nethunter kex passwd  # To set the KeX password${reset}\n"
printf "${green}[+] nethunter kex start   # To start NetHunter GUI${reset}\n"
printf "${green}[+] nethunter kex stop    # To stop NetHunter GUI${reset}\n"
printf "${green}[+] nethunter -r          # To run NetHunter as root${reset}\n"
printf "${green}[+] nh                    # Shortcut for nethunter${reset}\n\n"

