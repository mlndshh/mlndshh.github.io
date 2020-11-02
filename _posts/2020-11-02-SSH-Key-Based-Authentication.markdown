---
layout: post
title:  "How to Configure SSH Key-Based Authentication on a Linux Server and set up VS Code for Remote Development"
date:   2020-11-02 17:20:50 +0530
categories: ssh
---
**How to Configure SSH Key-Based Authentication on a Linux Server and set up VS Code for Remote Development**

	### Generate SSH Key pair on your local computer:
	1. `ssh-keygen`
	This utility will prompt you to select a location for the keys, and it's generally a good idea to leave it as the default directory. If you already had an ssh key pair it'll ask you to overwrite it. It will ask you to enter a passphrase which is optional. If you enter the passphrase, you will have to enter it every time you use the ssh key pair

	`milind@Milinds-MacBook-Air ~ % ssh-keygen
	Generating public/private rsa key pair.
	Enter file in which to save the key (/Users/milind/.ssh/id_rsa):
	Enter passphrase (empty for no passphrase):
	Enter same passphrase again:
	Your identification has been saved in /Users/milind/.ssh/id_rsa.
	Your public key has been saved in /Users/milind/.ssh/id_rsa.pub.
	The key fingerprint is:
	SHA256:D1G2Y64Yd3nHKpnFsqQ9WcLxRC4HRipSIVSA+Vvdsms milind@Milinds-MacBook-Air.local
	The key's randomart image is:
	+---[RSA 3072]----+
	| ++o+. .* .  |
	|  o  o = = |
	| .. ..o.* +  |
	|  ....o=.X . |
	| o. SoO * o  |
	|  .  +.O @ o |
	|  . o.X .  |
	|  E  o |
	| . |
	+----[SHA256]-----+`

	2. `ssh-copy-id username@remote_host`
	Here it scans for your ssh public key that was created earlier (which is why leaving it in the default directory is a good idea) and when it finds it, it'll ask you to enter the password of the account.

	`ssh-copy-id  milind.shah@192.168.1.17
	/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/Users/milind/.ssh/id_rsa.pub"
	/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
	/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
	milind.shah@192.168.1.17's password:
	Number of key(s) added:  1
	Now try logging into the machine, with: "ssh 'milind.shah@192.168.1.17'"
	and check to make sure that only the key(s) you wanted were added.`

	3. Now login using ssh username@remote_host
	You should be able to log in without entering your password!

	### Setting up VS Code for Remote Development using SSH
	1. Assuming you have VS Code installed, install the  [Remote Development extension pack](https://aka.ms/vscode-remote/download/extension).
	2. Verify that you can login via ssh using `ssh user@hostname`
	3. In VS Code, select **Remote-SSH: Connect to Host...** from the Command Palette (F1) and use the same `user@hostname` as in step 2.
	4. If it asks you to choose a config file, choose your local file. (/Users/username/.ssh/config)
	5. You should now be able to connect to your server via SSH using VS Code!


	Sources:
	1. https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server
	2. https://code.visualstudio.com/docs/remote/ssh 