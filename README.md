# standAlone		# Enabling cryptomount in GRUB2 + REMOVABLE MODE, KEYSLOT SETUP & AUTO-UNLOCK
	# Enable the feature, update the GRUB image and reinstall in removable mode:

echo "GRUB_ENABLE_CRYPTODISK=y" | sudo tee -a /etc/default/grub
sudo update-grub

sudo grub-install /dev/nvme0n1 --force-extra-removable --no-nvram
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --boot-directory=/boot --bootloader-id=GRUB --force-extra-removable --no-nvram

sudo cryptsetup luksDump /dev/nvme0n1p3 | grep -B1 "Iterations:"
#	Key Slot 0: ENABLED
#	Iterations:             1000000

sudo cryptsetup luksChangeKey --pbkdf-force-iterations 420690 /dev/nvme0n1p3
# 	Enter passphrase to be changed:
#	Enter new passphrase:
#	Verify passphrase:
	# (You can reuse the existing passphrase in the above prompts.)

sudo cryptsetup luksOpen --test-passphrase --verbose /dev/nvme0n1p3
#	Enter passphrase for /dev/nvme0n1p3:
#	Key slot 1 unlocked.
#	Command successful.

		# Avoiding the Extra Password Prompt
  
	# Generate the shared secret (here with 512 bits of entropy as it’s also the size of the volume key) inside a new file.
sudo mkdir -m0700 /etc/keys
bash
( umask 0077 && sudo dd if=/dev/urandom bs=1 count=64 of=/etc/keys/root.key conv=excl,fsync )
#	64+0 records in
#	64+0 records out
#	64 bytes copied, 0.000698363 s, 91.6 kB/s
zsh
sudo chmod u=rx,go-rwx /etc/keys
sudo chmod u=r,go-rwx /etc/keys/root.key

	# Create a new key slot with that key file.
sudo cryptsetup luksAddKey /dev/nvme0n1p3 /etc/keys/boot.key

	# and add in the other voleumes too.
sudo cryptsetup luksAddKey /dev/nvme0n1p4 /etc/keys/swap.key
sudo cryptsetup luksAddKey /dev/nvme0n1p5 /etc/keys/root.key

	# add these entries into the ccrypttab.
echo "nvme0n1p3_crypt UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p3) /etc/keys/root.key luks,key-slot=1" | sudo tee -a /etc/crypttab
echo "nvme0n1p4_crypt UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p4) /etc/keys/root.key luks,discard,key-slot=1" | sudo tee -a /etc/crypttab
echo "nvme0n1p5_crypt UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p5) /etc/keys/root.key luks,swap,discard,key-slot=1" | sudo tee -a /etc/crypttab

	# make sure its all correct and comment out <#> the original entries with:
sudo nano /etc/crypttab - the mapper name for boot may change, so you may want to run lsblk to check it.
#
#
#

			# Finishing up BTRFS
	# now we can add the final modifications to our /etc/fstab for our btrfs filesystem with:
sudo nano /etc/fstab

	# like so:

/dev/mapper/nvme0n1p4_crypt	/		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@		0	0
/dev/mapper/nvme0n1p4_crypt	/.snapshots     btrfs   defaults,noatime,ssd,compress=lzo,subvol=@.snapshots	0	4
# /boot was on /dev/nvme0n1p3 during installation
UUID=9a90bf5e-fb5b-4a00-beba-616bc0092abe	/boot	ext4	defaults,noatime				0	1
# /boot/efi was on /dev/nvme0n1p1 during installation
UUID=EB8C-72F1			/boot/efi	vfat	umask=0077						0	1
/dev/mapper/nvme0n1p4_crypt	/home		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@home		0	2
/dev/mapper/nvme0n1p4_crypt	/root		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@root		0	3
/dev/mapper/nvme0n1p4_crypt	/srv		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@srv		0	0
/dev/mapper/nvme0n1p4_crypt	/tmp		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@tmp		0	0
/dev/mapper/nvme0n1p4_crypt	/usr/local	btrfs	defaults,noatime,ssd,compress=lzo,subvol=@usr@local	0	0
/dev/mapper/nvme0n1p4_crypt	/var/log	btrfs	defaults,noatime,ssd,compress=lzo,subvol=@var@log	0	0
/dev/mapper/nvme0n1p5_crypt	none		swap	sw							0	0
# /var/lib/gdm3 was on /dev/mapper/nvme0n1p4_crypt during installation
UUID=e1005fc7-0668-4322-b272-22b6f4727e2a	/var/lib/gdm3	btrfs	defaults,subvol=@var@lib@gdm3		0	0
# /var/lib/AccountsService was on /dev/mapper/nvme0n1p4_crypt during installation
UUID=e1005fc7-0668-4322-b272-22b6f4727e2a	/var/lib/AccountsService	btrfs	defaults,subvol=@var@lib@AccountsService	0	0 

	# when done editing save your work, exit the editor then run:
sudo systemctl daemon-reload

	# finally lets tell initramfs to use our keyfile:
echo "KEYFILE_PATTERN=/etc/keys/*.key" | sudo tee -a /etc/cryptsetup-initramfs/conf-hook

	# include restrictive permissions to avoid leaking key material:
echo "UMASK=0077" | sudo tee -a /etc/initramfs-tools/initramfs.conf

	# regenerate initramfs:
sudo update-initramfs -u -k all

	# update grub
sudo grub-update

	# and reboot for the changes to take affect:
sudo reboot now -f

			# We're in....

   
	# At this point I would open snapper-gui, make a new pre-boot snapshot for root, set it for 3 with 'name' as NUMBER_LIMIT and count as 10.

		# Compilation of a custom kernel:
sudo tar -xaf /usr/src/linux-source-6.1.tar.xz
sudo cp /booot/config-6.1.0-kali9-amd64 ~/kernel/linux-source-6.1/.config
sudo cp /booot/config-6.1.0-kali9-amd64 ~/src/linux-source-6.1/.config
cd ~/src/linux-source-6.1
sudo make menuconfig
sudo make clean
sudo make deb-pkg LOCALVERSION=-custom KDEB_PKGVERSION=$(make kernelversion)-1

	# let's confirm it worked with:
ls ../*.deb

	# this will display the contents we just created, and finally:
sudo dpkg -i ~/src/custom-linux-image-6.1.deb

# Ultimately I would argue that this more than helps against evil maid attacks if set up properly, although truthfully an easier way to do the same this would be to just modify the grub menu post install, so it contains unique menue item name, a quote or nmemonic. This could act as a buffer between the insecurity of a corrupted bootloadder and a system's ability to reach the net if worse comes to worse, and dare I say more difficult if the kernel involved is custom.

mv /usr/share/grub/themes/kali/grub-16x9.png  ~/Documents && mv /usr/share/grub/themes/kali/grub-4x3.png ~/Documents
mv ~/Documents/my-grub-img-16.9 /usr/share/grub/themes/kali && mv ~/Documents/my-grub-img-4x3.png /usr/share/grub/themes/kali

# replace these files with black pngs unless you have something more 'cool', as if black pngs aren't cool enough as it is in the proverbial sense.

# it seems either the method I've outlined which automatically unlocks /boot at startup does not work, or it was an issue until I tried it with gnome.. If this is the case just boot in and use the 'disks' gui to alter it's settings there; deactivate and then reactivate password protection then deactivate and reactivate user settings from the menu to the crypttab settings via the same gui it should update initramfs and related files from there then update inittramfss and grub, reboot after this change and it should work fine. I do have an outline mapped to trace down the problem however life has begun to pile up so I hope that maybe the community can continue with any work that may need to be done. One such issue is the outdated usage of grub2 where I believe it now unlocks luks2 and may contain an internal password management function.
