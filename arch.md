# Arch Install Documentation
This documentation records the installation of Arch by Austin Lively for CS 3353, Fall 2021.

## Installation Process

### Prerequisites
This guide assumes the user has already created and booted from an Arch Install medium.

### 1) Network Check
To make sure I have internet for the duration of the install I started by checking the following command:
```
ip link
```

This command lists the network interfaces for the install process. In this, I found two interfaces: The Loopback, and an Ethernet interface. Knowing that I'm using a Virtual Machine, that told me that the VM is shown an Ethernet port as the abstraction for virutalization.
To test the connection, I used a ping command on the google dns `8.8.8.8`. This resulted in sucessful pings, showing me that I have internet connection for the install.

### 2) Time Update
Next I updated the internal timeclock using the following command:
```
timedatectl set-ntp true
```
This command sets the OS to check for the time using the internet. To check this i used `timedatectl status` to determine if the time had synced properly, which it had.

### 3) Disk Partitioning
```
fdisk -l
```
The above command lists the available disks that the operating system can see. Mine releaved two: the Virtual Disk I created as well as a "Loop" disk. I googled and discovered that the "Loop" disk is a pseudo-device that allows a file system to be accessed like a device block.
First, we entered Command Mode of `fdisk` by calling `fdisk /dev/sda`. Next, I used the "New Partion" command and provided it the following in it's prompts:
```
Partition Type: p
Partition number: 1
First Sector: (default)
Last Sector: +512M
```
This created a new partition. I then repeated the process to create another partition out of the remainder of the disk.
```
Partition Type: p
Partition number: 2
First Sector: (default)
Last Sector: (default)
```

### 4) Formatting the Disk

First, I formatted the EFI partition using `mkfs.fat -F32 /dev/sda1`. next I formatted the rest using `mkfs.ext4 /dev/sda2`

### 5) Preparing to work within the new install

```
mount /dev/sda2 /mnt
pacstrap /mnt base linux linux-firmware
genfstab -U /mnt >> /mnt/etc/fstab
```
This sets up the system such that we can chroot into it and begin installing packages, using the following command:
```
arch-chroot /mnt
```

### 6) Installing Packages
```
pacman -S iputils
pacman -S iproute2
pacman -S netctl
pacman -S dhcpcd
```

These packages ensure that we are able to use networking in the final installation, once the installation medium is unavailable

### 7) Localization settings- Timezone

Found timezone by: `timedatectl list-timezones |grep Central`

timezone by: `timedatectl set-timezone US/Central`

### 8) Installing a Boot loader
For a bootloader, I choose Grub

```
pacman -S grub #install Grub
grub-install --target=x86_64-efi --efi-directory=\mnt --bootloader-id=GRUB #Install Grub onto the mounted filesystem
grub-mkconfig -o /boot/grub/grub.cfg # Setup Grub configuration to detect operating systems
```
This set of commands installs grub, places it as part of the final installation drive, and then configures it to detect the Arch Operating system we are creating. Now, we are capable of actually rebooting the device and starting Arch linux without the live installation medium.

### 9) configuring a network
In order to have internet access, I had to configure the network control module for Arch. Network Control uses profiles stored in `/etc/netctl` to configure the network operations of the system. The profile I created for my virtual machine is as follows:

/etc/netctl/mainNet
```
Interface=ens33
Connection=ethernet
IP=dhcp
```

To apply this profile, I ran `netctl enable mainNet`. This starts the profile and sets it up to auto-start on boot later.

### 10) Desktop Environment
For a desktop environment, I used LXDE.

```
sudo pacman -Sy --noconfirm lxde lxdm
sudo systemctl enable lxdm
sudo sed -i /etc/lxdm/lxdm.conf \
     -e 's;^# session=/usr/bin/startlxde;session=/usr/bin/startlxde;g'
sudo reboot
```


### 11) User Setup
I wanted to create 3 accounts: austin, sal, and codi, each with the password "GraceHopper1906". On login, each user is to change their password. This is done using the following code template, where username is replaced with the username we want to create:

```
useradd -m username
passwd username
passwd --expire username
```

Next, I wanted to give each one sudo powers. To do this, I edited the `/etc/sudoers` group, allowing the group `wheel` to run sudo. After this, I simply added the users to the group `wheel`, using the following:
```
usermod -aG wheel username
```

### 12)  Install a different shell other than bash, such as zsh or fish.

I installed zsh using pacman:
`pacman -S zsh`

### 13)  Install ssh and use it to ssh into the class gateway.

I discovered that ssh was already installed, and so I simply found the command structure for connecting to the class gateway:

Connecting to the Gateway:
`ssh -p53997 azl0502@129.244.245.21`

### 14)  Add color coding to the terminal (like you see in the archiso during installation).

In order to create a color coding setup, we have to edit two files:

/etc/bash.bashrc
```
# /etc/bash.bashrc
#
# https://wiki.archlinux.org/index.php/Color_Bash_Prompt
#
# This file is sourced by all *interactive* bash shells on startup,
# including some apparently interactive shells such as scp and rcp
# that can't tolerate any output. So make sure this doesn't display
# anything or bad things will happen !

# Test for an interactive shell. There is no need to set anything
# past this point for scp and rcp, and it's important to refrain from
# outputting anything in those cases.

# If not running interactively, don't do anything!
[[ $- != *i* ]] && return

# Bash won't get SIGWINCH if another process is in the foreground.
# Enable checkwinsize so that bash will check the terminal size when
# it regains control.
# http://cnswww.cns.cwru.edu/~chet/bash/FAQ (E11)
shopt -s checkwinsize

# Enable history appending instead of overwriting.
shopt -s histappend

case ${TERM} in
	xterm*|rxvt*|Eterm|aterm|kterm|gnome*)
		PROMPT_COMMAND=${PROMPT_COMMAND:+$PROMPT_COMMAND; }'printf "\033]0;%s@%s:%s\007" "${USER}" "${HOSTNAME%%.*}" "${PWD/#$HOME/~}"'
		;;
	screen)
		PROMPT_COMMAND=${PROMPT_COMMAND:+$PROMPT_COMMAND; }'printf "\033_%s@%s:%s\033\\" "${USER}" "${HOSTNAME%%.*}" "${PWD/#$HOME/~}"'
		;;
esac

# fortune is a simple program that displays a pseudorandom message
# from a database of quotations at logon and/or logout.
# If you wish to use it, please install "fortune-mod" from the
# official repositories, then uncomment the following line:

#[[ "$PS1" ]] && /usr/bin/fortune

# Set colorful PS1 only on colorful terminals.
# dircolors --print-database uses its own built-in database
# instead of using /etc/DIR_COLORS. Try to use the external file
# first to take advantage of user additions. Use internal bash
# globbing instead of external grep binary.

# sanitize TERM:
safe_term=${TERM//[^[:alnum:]]/?}
match_lhs=""

[[ -f ~/.dir_colors ]] && match_lhs="${match_lhs}$(<~/.dir_colors)"
[[ -f /etc/DIR_COLORS ]] && match_lhs="${match_lhs}$(</etc/DIR_COLORS)"
[[ -z ${match_lhs} ]] \
	&& type -P dircolors >/dev/null \
	&& match_lhs=$(dircolors --print-database)

if [[ $'\n'${match_lhs} == *$'\n'"TERM "${safe_term}* ]] ; then

	# we have colors :-)

	# Enable colors for ls, etc. Prefer ~/.dir_colors
	if type -P dircolors >/dev/null ; then
		if [[ -f ~/.dir_colors ]] ; then
			eval $(dircolors -b ~/.dir_colors)
		elif [[ -f /etc/DIR_COLORS ]] ; then
			eval $(dircolors -b /etc/DIR_COLORS)
		fi
	fi

	PS1="$(if [[ ${EUID} == 0 ]]; then echo '\[\033[01;31m\]\h'; else echo '\[\033[01;32m\]\u@\h'; fi)\[\033[01;34m\] \w \$([[ \$? != 0 ]] && echo \"\[\033[01;31m\]:(\[\033[01;34m\] \")\\$\[\033[00m\] "

	# Use this other PS1 string if you want \W for root and \w for all other users:
	# PS1="$(if [[ ${EUID} == 0 ]]; then echo '\[\033[01;31m\]\h\[\033[01;34m\] \W'; else echo '\[\033[01;32m\]\u@\h\[\033[01;34m\] \w'; fi) \$([[ \$? != 0 ]] && echo \"\[\033[01;31m\]:(\[\033[01;34m\] \")\\$\[\033[00m\] "

	alias ls="ls --color=auto"
	alias dir="dir --color=auto"
	alias grep="grep --color=auto"
	alias dmesg='dmesg --color'

	# Uncomment the "Color" line in /etc/pacman.conf instead of uncommenting the following line...!

	# alias pacman="pacman --color=auto"

else

	# show root@ when we do not have colors

	PS1="\u@\h \w \$([[ \$? != 0 ]] && echo \":( \")\$ "

	# Use this other PS1 string if you want \W for root and \w for all other users:
	# PS1="\u@\h $(if [[ ${EUID} == 0 ]]; then echo '\W'; else echo '\w'; fi) \$([[ \$? != 0 ]] && echo \":( \")\$ "

fi

PS2="> "
PS3="> "
PS4="+ "

# Try to keep environment pollution down, EPA loves us :-)
unset safe_term match_lhs

# Try to enable the auto-completion (type: "pacman -S bash-completion" to install it).
[ -r /usr/share/bash-completion/bash_completion ] && . /usr/share/bash-completion/bash_completion

# Try to enable the "Command not found" hook ("pacman -S pkgfile" to install it).
# See also: https://wiki.archlinux.org/index.php/Bash#The_.22command_not_found.22_hook
[ -r /usr/share/doc/pkgfile/command-not-found.bash ] && . /usr/share/doc/pkgfile/command-not-found.bash
```

and

/etc/DIR_COLORS
```
# Configuration file for the color ls utility
# This file goes in the /etc directory, and must be world readable.
# You can copy this file to .dir_colors in your $HOME directory to override
# the system defaults.

# COLOR needs one of these arguments: 'tty' colorizes output to ttys, but not
# pipes. 'all' adds color characters to all output. 'none' shuts colorization
# off.
COLOR all

# Extra command line options for ls go here.
# Basically these ones are:
#  -F = show '/' for dirs, '*' for executables, etc.
#  -T 0 = don't trust tab spacing when formatting ls output.
OPTIONS -F -T 0

# Below, there should be one TERM entry for each termtype that is colorizable
TERM linux
TERM console
TERM con132x25
TERM con132x30
TERM con132x43
TERM con132x60
TERM con80x25
TERM con80x28
TERM con80x30
TERM con80x43
TERM con80x50
TERM con80x60
TERM xterm
TERM xterm-color
TERM xterm-256color
TERM vt100
TERM rxvt
TERM rxvt-256color
TERM rxvt-cygwin
TERM rxvt-cygwin-native
TERM rxvt-unicode
TERM rxvt-unicode-256color
TERM rxvt-unicode256
TERM screen

# EIGHTBIT, followed by '1' for on, '0' for off. (8-bit output)
EIGHTBIT 1

# Below are the color init strings for the basic file types. A color init
# string consists of one or more of the following numeric codes:
# Attribute codes: 
# 00=none 01=bold 04=underscore 05=blink 07=reverse 08=concealed
# Text color codes:
# 30=black 31=red 32=green 33=yellow 34=blue 35=magenta 36=cyan 37=white
# Background color codes:
# 40=black 41=red 42=green 43=yellow 44=blue 45=magenta 46=cyan 47=white
NORMAL 00	# global default, although everything should be something.
FILE 00 	# normal file
DIR 01;34 	# directory
LINK 01;36 	# symbolic link
FIFO 40;33	# pipe
SOCK 01;35	# socket
BLK 40;33;01	# block device driver
CHR 40;33;01 	# character device driver

# This is for files with execute permission:
EXEC 01;32 

# List any file extensions like '.gz' or '.tar' that you would like ls
# to colorize below. Put the extension, a space, and the color init string.
# (and any comments you want to add after a '#')
.cmd 01;32 # executables (bright green)
.exe 01;32
.com 01;32
.btm 01;32
.bat 01;32
.tar 01;31 # archives or compressed (bright red)
.tgz 01;31
.arj 01;31
.taz 01;31
.lzh 01;31
.zip 01;31
.z   01;31
.Z   01;31
.gz  01;31
.jpg 01;35 # image formats
.gif 01;35
.bmp 01;35
.xbm 01;35
.xpm 01;35
.tif 01;35	
```

### 15) Aliases

Aliases are appended to the end of the /etc/bash.bashrc file, so that all Bash shells have them avaiable.

I added an Alias for the Class Gateway SSH command:
`alias gateway = "ssh -p53997 azl0502@129.244.245.21"`

I also added a few utitlty aliases.

This one creates a path command, to display the path in readable format: `alias path='echo -e ${PATH//:/\\n}'`

This one prints a date time in readable format for the user: `alias now='date +"%T"'`

This one prints the current time of the system: `alias nowtime=now`

This one prints just the current data of the system: `alias nowdate='date +"%d-%m-%Y"'`
