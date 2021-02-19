---
layout: default
title: Authorized SSH Keys
parent: Linux
nav_order: 2
---
# Authorized SSH Keys
{: .no_toc }
This is how I generated a private/public key pair for securing the SSH logins for a user on a remote resource. 

<details open markdown="block">
  <summary>
   Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

{: .no_toc .text-delta }

---

## Getting started
Secure shell (SSH) is the encrypted protocol used to log in to user accounts on remote Linux or Unix-like computers. Typically such user accounts are secured using passwords. When you log in to a remote computer, you must provide the user name and password for the account you are logging in to.

Passwords are the most common means of securing access to computing resources. Despite this, password-based security does have its flaws. People choose weak passwords, share passwords, use the same password on multiple systems, and so on.

SSH keys are much more secure, and once they’re set up, they’re just as easy to use as passwords.


SSH keys are created and used in pairs. The two keys are linked and cryptographically secure. One is your public key, and the other is your private key. They are tied to your user account. If multiple users on a single computer use SSH keys, they will each receive their own pair of keys.

Your private key is installed in your home folder (usually), and the public key is installed on the remote computer—or computers—that you will need to access.

Your private key must be kept safe. If it is accessible to others, you are in the same position as if they had discovered your password. A sensible—and highly recommended—precaution is for your private key to be encrypted on your computer with a robust passphrase.

The public key can be shared freely without any compromise to your security. It is not possible to determine what the private key is from an examination of the public key. The private key can encrypt messages that only the private key can decrypt.

When you make a connection request, the remote computer uses its copy of your public key to create an encrypted message. The message contains a session ID and other metadata. Only the computer in possession of the private key—your computer—can decrypt this message.

Your computer accesses your private key and decrypts the message. It then sends its own encrypted message back to the remote computer. Amongst other things, this encrypted message contains the session ID that was received from the remote computer.

The remote computer now knows that you must be who you say you are because only your private key could extract the session Id from the message it sent to your computer.

### Prerequisites
* pfSense 2.4.5-RELEASE-p3 (amd64)
* linux client

---

## ssh-keygen
```bash
rogue:.ssh one$ ssh-keygen -b 4096 
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/one/.ssh/id_rsa): /Users/one/.ssh/FR-LNX-01-luke
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /Users/one/.ssh/FR-LNX-01-luke.
Your public key has been saved in /Users/one/.ssh/FR-LNX-01-luke.pub.
The key fingerprint is:
SHA256:XGVdbdGxzJky2KYODOPiAnFFsUivqsIczqqKfzKPjsKI one@rogue.local
The key's randomart image is:
+---[RSA 4096]----+
| ...++. .+. .==. |
|  .. .   + *o    |
|o.. =   = +.     |
|.  + S . o       |
|. +   + o        |
|o0+  o .         |
|oo.+.            |
|&oS+             |
|F*so  .          |
+----[SHA256]-----+
rogue:.ssh one$ 
```

### Change passphrase
To change or remove the passphrase, simply type:
```bash
rogue:.ssh one$ ssh-keygen -p
```
```bash
Enter file in which the key is (/root/.ssh/id_rsa):
```
You can type the location of the key you wish to modify or press ENTER to accept the default value:

```bash
Enter old passphrase:
```
Enter the old passphrase that you wish to change. You will then be prompted for a new passphrase:

```bash
Enter new passphrase (empty for no passphrase):
Enter same passphrase again:
```
Here, enter your new passphrase or press ENTER to remove the passphrase.

### ssh-copy-id
This is useful, if the `sshd` on the remote site allows for password-login. When you are prompted for a password in this command, it is the password for the user on the remote site. 
```bash
rogue:.ssh one$ ssh-copy-id -i /Users/one/.ssh/FR-LNX-01-luke.pub luke@192.168.1.1
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/Users/one/.ssh/FR-LNX-01-luke.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
luke@192.168.1.1's password: 

Number of key(s) added:        1

Now try logging into the machine, with:   "ssh 'luke@192.168.1.1'"
and check to make sure that only the key(s) you wanted were added.

rogue:.ssh one$ 
```

To use this key, remember to add the `-i` option:
```bash
rogue:.ssh one$ ssh -i FR-LNX-01-luke luke@192.168.1.1
Enter passphrase for key 'FR-LNX-01-luke': 
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-59-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Jan 11 23:00:48 UTC 2021

  System load:  0.3                Processes:                141
  Usage of /:   13.4% of 57.70GB   Users logged in:          0
  Memory usage: 12%                
  Swap usage:   0%                 IPv4 address for enp2s0:  192.168.1.1
  Temperature:  28.0 C

 * Introducing self-healing high availability clusters in MicroK8s.
   Simple, hardened, Kubernetes for production, from RaspberryPi to DC.

     https://microk8s.io/high-availability

10 updates can be installed immediately.
0 of these updates are security updates.
To see these additional updates run: apt list --upgradable


*** System restart required ***
Last login: Mon Jan 11 23:00:35 2021 from 10.0.22.6
luke@FR-LNX-01:~$ 
```

### sshd_config
Allright, it worked. Now, let us secure the resource furthermore by editing the `sshd_config` and disable `PassWordAuthentication` (it is at the bottom of the file)
.
```bash
luke@FR-LNX-01:~$ sudo vim /etc/ssh/sshd_config

PasswordAuthentication no
```
Restart `sshd`:
```bash
luke@FR-LNX-01:~$ sudo service ssh restart
```

Try to log in with a password:
```bash
luke@FR-LNX-01:~$ exit
logout
Connection to 192.168.1.1 closed.
rogue:.ssh one$ ssh -l luke 192.168.1.1
luke@192.168.1.1: Permission denied (publickey).
rogue:.ssh one$ 
```

Great! Let us use the private key from now on, yehaw!

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://askubuntu.com/questions/53553/how-do-i-retrieve-the-public-key-from-a-ssh-private-key](https://askubuntu.com/questions/53553/how-do-i-retrieve-the-public-key-from-a-ssh-private-key)
* [https://www.howtogeek.com/424510/how-to-create-and-install-ssh-keys-from-the-linux-shell/](https://www.howtogeek.com/424510/how-to-create-and-install-ssh-keys-from-the-linux-shell/)
* [https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-1604](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-1604)
* [https://www.freecodecamp.org/news/the-ultimate-guide-to-ssh-setting-up-ssh-keys/](https://www.freecodecamp.org/news/the-ultimate-guide-to-ssh-setting-up-ssh-keys/)
* [https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys)