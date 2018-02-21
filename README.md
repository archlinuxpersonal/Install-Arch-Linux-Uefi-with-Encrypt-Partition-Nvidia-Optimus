# Install-Arch-Linux-Uefi-with-Encrypt-Partition-Nvidia-Optimus
Personal setup


+-----------------Install Arch Linux Uefi with Encrypt Partition Nvidia Optimus-----------------+
|	#BIOS SETUP F2										|
|	#install on uefi with GPT partition							|
|	#disable discrete graphic disable secure boot						|
+-----------------------------------------------------------------------------------------------+

+--------------Clear disk with gpt partition------------+
|	gdisk /dev/sda					|
|	=> p #print partition				|
|	=> o						|
|	=> w						|
|	=> y						|
+-------------------------------------------------------+

+---------------------Create partition----------------------------------+					
|	gdisk /dev/sda							|
|	=> n #create new partition for efi				|
|	=> 1 # select partition /dev/sda1				|
|	=> 2048 #begin partition					|
|	=> +500M #End partition size					|
|	=> ef00 #efi partition filesystem				|
|									|
|	=> n #create new partition for boot				|
|	=> 2 # select partition /dev/sda2				|
|	=> number_size #begin partition default value			|
|	=> +250M #End partition size					|
|	=> 8300 #linux filesystem					|
|									|
|	=> n #create new partition for encrypted			|
|	=> 3 # select partition /dev/sda3				|
|	=> number_size #begin partition default value			|
|	=> number_size #End Partition size use default value		|
|	=> 8300 #linux filesystem					|
|									|
|	=> p #print partition						|
|	=> w #write partition						|
|	=> y #yes and exit						|
|	reboot								|
+-----------------------------------------------------------------------+

+---------#setup internet connection------------+
|	rfkill list #look block phy0 number	|	
|	rfkill unblock 3			|
|	wifi-menu				|
|	dhcpcd wlp2s0				|
|	ping -c 5 8.8.8.8			|
+-----------------------------------------------+

+-----------#zero every partitions--------------+
|	cat /dev/zero > /dev/sda1		|
|	cat /dev/zero > /dev/sda2		|
|	cat /dev/zero > /dev/sda3 #ifneeded	|
+-----------------------------------------------+

+-------------------------------------#Format Partition efi boot Lvm------------------------------------+
|	#Create filesystems for /boot/efi and /boot							|
|	mkfs.vfat -F 32 /dev/sda1									|
|	mkfs.ext2 /dev/sda2										|
|													|
|	#Encrypt & open luks system partition								|
|	cryptsetup -c aes-xts-plain64 -h sha512 -s 512 --use-random luksFormat /dev/sda3		|
|	cryptsetup luksOpen /dev/sda3 Name-Of-Encrypted-Disk						|
|													|
|	#Create volumn group encrypted LVM partitions							|
|	pvcreate /dev/mapper/Name-Of-Encrypted-Disk							|
|	vgcreate ArchLinux /dev/mapper/Name-Of-Encrypted-Disk						|
|	lvcreate -L +8G ArchLinux -n swap								|
|	lvcreate -l +100%FREE ArchLinux -n root								|
|													|
|	#Create filesystems on your encrypt partitions							|
|	mkswap /dev/mapper/ArchLinux-swap								|
|	mkfs.ext4 /dev/mapper/ArchLinux-root								|
+-------------------------------------------------------------------------------------------------------+

+----------------------------#Mount the new system------------------------------+
|	mount /dev/mapper/ArchLinux-root /mnt					|
|	swapon /dev/mapper/ArchLinux-swap					|
|	mkdir /mnt/boot								|
|	mount /dev/sda2 /mnt/boot						|
|	mkdir /mnt/boot/efi							|
|	mount /dev/sda1 /mnt/boot/efi						|
+-------------------------------------------------------------------------------+

+------------------------------------#setup mirrorlist List by speed------------------------------------+
|	cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup					|
|	sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.backup					|
|	rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist			|
+-------------------------------------------------------------------------------------------------------+

+----------------------------------------#Install Arch system-------------------------------------------+
|	pacstrap /mnt base base-devel grub-efi-x86_64 efibootmgr dialog wpa_supplicant zsh		|
+-------------------------------------------------------------------------------------------------------+

+---------------------------------#Create and review FSTAB------------------------------+
|	genfstab -U /mnt >> /mnt/etc/fstab						|
|	cat /mnt/etc/fstab								|
+---------------------------------------------------------------------------------------+

+------------------------------------------------#chroot And Setup new system-----------------------------------------------------------------------------------+
|	arch-chroot /mnt /bin/zsh																|
|																				|
|	#Set the system clock																	|
|	ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime													|
|	hwclock --systohc --utc																	|																				|
|	#Assign hostname																	|
|	echo MyHostName > /etc/hostname																|
|																				|
|	#Set locale																		|
|	nano /etc/locale.gen **uncomment only**: en_US.UTF-8 UTF-8												|
|	echo LANG=en_US.UTF-8 > /etc/locale.conf														|
|	locale-gen																		|
|																				|
|	#set hosts																		|
|	echo 127.0.0.1 localhost > /etc/hosts															|
|																				|
|	#Set root password																	|
|	passwd																			|
|																				|
|	#Create a User, assign appropriate Group membership, and set a User password.  'Wheel' is just one important Group.					|
|	useradd -m -G wheel -s /bin/bash MyUserName														|
|	passwd MyUserName																	|
|	nano /etc/sudoers																	|
|	#uncomment #%wheel ALL=(ALL) ALL															|
|																				|
|	#Configure mkinitcpio with the correct HOOKS required for initrd image											|
|	nano /etc/mkinitcpio.conf																|
|	HOOKS=(base udev autodetect modconf block keymap encrypt lvm2 resume filesystems keyboard fsck)								|
|																				|
|	#Generate initrd image																	|
|	mkinitcpio -p linux																	|
|																				|
|	#Install and configure Grub-EFI																|
|	grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux									|
|																				|
|	nano /etc/default/grub																	|
|	GRUB_CMDLINE_LINUX="ipv6.disable=1 net.ifnames=0 biosdevname=0 cryptdevice=/dev/sda3:Name-Of-Encrypted-Disk resume=/dev/mapper/ArchLinux-swap"		|
|	#Generate Grub Configuration:																|
|	grub-mkconfig -o /boot/grub/grub.cfg															|
|																				|
|	#install Xorg System																	|
|	pacman -S xorg-server xorg-xinit mesa xf86-input-synaptics												|
|																				|
|	#install desktop environment																|
|	pacman -S gnome gnome-extra gdm networkmanager network-manager-applet											|
|	systemctl enable gdm																	|
|	systemctl enable NetworkManager																|
|																				|
|	#install Graphics Drivers																|
|	#intel																			|
|	#enable Multilib /etc/pacman.conf															|
|	nano /etc/pacman.conf																	|
|	#uncomment																		|
|	#[multilib]																		|
|	#Include = /etc/pacman.d/mirrorlist															|
|	pacman -Sy																		|
|	pacman -S xf86-video-intel intel-dri lib32-intel-dri libva-intel-driver libva										|
|	#Exit chroot Arch System																|
|	exit																			|																		
+---------------------------------------------------------------------------------------------------------------------------------------------------------------+

+---------------#Unmount all partitions---------+
|	umount -R /mnt				|
|	swapoff -a				|
+-----------------------------------------------+

+--------------#Reboot Arch Linux System!-------+
|	reboot					|
+-----------------------------------------------+

+---------------------------#for optimus laptop with dual intel VGA compatible controller and nvidia 3D controller--------------+
|	#login to new arch linux system as root											|
|	#install Nvidia Optimus With Bumblebee											|
|	echo -e "blacklist nouveau\noptions nouveau modeset=0\nalias nouveau off" > /etc/modprobe.d/blacklist-nouveau.conf	|
|	reboot															|
|																|
|	#enable discrete graphic & enable secure boot on Bios F2 setup										|
|	#login as user														|
|	#Installing Yaourt													|
|	nano /etc/pacman.conf													|
|	[archlinuxfr]														|
|	SigLevel = Never													|
|	Server = http://repo.archlinux.fr/$arch											|
|																|
|	sudo pacman -Syu yaourt													|
|																|
|	#Use the command below to sync Yaourt with AUR:										|
|	yaourt -Syy														|
|																|
|	#remove Nouveau if installed												|
|	sudo pacman -Rc xf86-video-nouveau											|
|	yaourt -Syyua														|
|																|
|	#Install Bumblebee													|
|	sudo pacman -S bumblebee mesa xf86-video-intel nvidia lib32-nvidia-utils lib32-virtualgl nvidia-settings bbswitch	|
|																|
|	#Add yourself to bumblebee group											|
|	sudo gpasswd -a $USER bumblebee												|
|	sudo -s #add root to bumblebee												|
|	sudo gpasswd -a $USER bumblebee												|
|	exit															|
|																|
|	sudo gpasswd -a $USER video												|
|	sudo -s #add root to video												|
|	sudo gpasswd -a $USER video												|
|	exit															|
|																|
|	#Enable bumblebeed service												|
|	sudo systemctl enable bumblebeed.service										|
|	sudo shutdown -r now													|
|																|
|	#check Bumblebee iw work												|
|	optirun --status													|
|	sudo pacman -S screenfetch												|
|	optirun screenfetch													|
|	optirun --status													|
+-------------------------------------------------------------------------------------------------------------------------------+

+--------#Install GUI Package Manager Pamac-------------+
|	yaourt -S pamac-aur				|
+-------------------------------------------------------+

+---------------------------------------------------#Install & setup plymouth-----------------------------------------------------------------------------------+
|	yaourt -S plymouth gdm-plymouth																|
|																				|
|	sudo nano /etc/mkinitcpio.conf																|
|																				|
|	MODULES="intel_agp i915"																|
|	HOOKS=(base udev plymouth autodetect modconf block keymap plymouth-encrypt lvm2 resume filesystems keyboard fsck)					|
|																				|
|	sudo mkinitcpio -p linux																|
|																				|
|	#disable gdm																		|
|	systemctl disable gdm.service																|
|	#enable gdm plymouth																	|
|	systemctl enable gdm-plymouth.service															|
|																				|
|	#install grub silent																	|
|	yaourt -S grub-silent																	|
|																				|
|	#reinstall grub																		|
|	sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux								|
|																				|
|	#edit /etc/default/grub																	|
|	sudo nano /etc/default/grub																|
|	GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"														|
|	GRUB_CMDLINE_LINUX="ipv6.disable=1 net.ifnames=0 biosdevname=0 cryptdevice=/dev/sda3:Name-Of-Encrypted-Disk resume=/dev/mapper/ArchLinux-swap"		|
|																				|
|	sudo grub-mkconfig -o /boot/grub/grub.cfg														|
|																				|
|	#install plymouth theme																	|
|	yaourt -S plymouth-theme-arch-logo-new															|
+---------------------------------------------------------------------------------------------------------------------------------------------------------------+

+----------------------------------#install gesture touchpad----------------------------+
|	sudo pacman -S xdotool wmctrl							|
|	yaourt -S libinput-gestures							|
|											|
|	#add user to grup input								|
|	sudo gpasswd -a $USER input							|
|											|
|	#add root to grub input								|
|	sudo -s										|
|	sudo gpasswd -a $USER input							|
|	exit										|
|											|
|	#CONFIGURATION gesture touchpad							|
|	cp /etc/libinput-gestures.conf ~/.config/libinput-gestures.conf			|
|											|
|	#start gesture									|
|	libinput-gestures-setup autostart						|
+---------------------------------------------------------------------------------------+

+------------------------------#install postgresql--------------------------------------+
|	sudo pacman -S postgresql							|
|	sudo -u postgres -i								|
|	initdb --locale $LANG -E UTF8 -D '/var/lib/postgres/data'			|
|	touch /var/lib/postgres/.psql_history						|
|	chown -R postgres:postgres /var/lib/postgres/.psql_history			|
|	psql										|
|	\q										|
|	exit										|			
+---------------------------------------------------------------------------------------+

+--------------------------------------------#Custom GTK--------------------------------+
|	yaourt -S flat-remix-git flat-remix-gnome-git flat-remix-gtk-git		|
|											|
|	#list best theme & icons							|
|	arc-solid-gtk-theme								|
|	arc-gtk-theme									|
|	gtk-arc-flatabulous-theme							|
|	zukitwo-themes-git								|
|	gnome-osx-shell-theme								|
|	gnome-shell-theme-arc-clearly-dark-git						|
|	arc-icon-theme									|
|	elementary-icon-theme								|
+---------------------------------------------------------------------------------------+

+-----------------#install ohmyzsh--------------+
|	yaourt -S oh-my-zsh-git			|
+-----------------------------------------------+

+----------------#List best gnome-shell extension-----------------------+
|	gnome-shell-extension-arc-menu-git				|
|	gnome-shell-extension-coverflow-alt-tab				|
|	gnome-shell-extension-dash-to-dock				|
|	gnome-shell-extension-drop-down-terminal			|
|	gnome-shell-extension-more-columns-in-applications-view		|
|	gnome-shell-extension-workspaces-to-dock			|
+-----------------------------------------------------------------------+
