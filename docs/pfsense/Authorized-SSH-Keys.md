---
layout: default
title: Authorized SSH Keys
parent: pfSense
nav_order: 1
---
# Authorized SSH Keys
{: .no_toc }
This is how I generated a private/public key pair for securing the SSH logins for the user `admin` on my pfSense.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
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

## Prerequisites
* pfSense 2.4.5-RELEASE-p3 (amd64)
* linux client

## ssh-keygen
```bash
┌─[jd@asdf]─[~]
└──╼ $ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/jd/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/jd/.ssh/id_rsa
Your public key has been saved in /home/jd/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:JD2Hht3UTSSWYmroN4X5dx5DGe8YRNJu/co4YMAa7JY jd@asdf
The key's randomart image is:
+---[RSA 3072]----+
|      .o+o.=oo   |
|       ++=oo*    |
| . . ..==+Bo.    |
|  o o +oZ*..     |
| . + o *S+..     |
|  D   * o . .    |
| .   . + + .     |
|        o p      |
|         .       |
+----[SHA256]-----+
┌─[jd@asdf]─[~]
```
Remember the passphrase, and I suggest renaming the key pair to something which you can remember which server this pair is used for. 

To generate the pair of keys using 4096 bits of encryption, do this: 

```bash
┌─[jd@asdf]─[~]
└──╼ $ssh-keygen -b 4096
```

## Public Key
Navigate to System > User Management and edit the user you want to log in to pfSense with. 

Copy the output from `id_rsa.pub` under <kbd>Keys</kbd> for the user in question:
```bash
┌─[jd@asdf]─[~]
└──╼ $cat /home/jd/.ssh/id_rsa.pub 
```

## Try it
```bash
┌─[jd@asdf]─[~]
└──╼ $ssh -l admin 192.168.234.33
Enter passphrase for key '/home/jd/.ssh/id_rsa':
```

Use specific private key:
```bash
┌─[jd@asdf]─[~]
└──╼ $ssh -l -i /home/jd/.ssh/server-1-admin admin 192.168.234.33
Enter passphrase for key '/home/jd/.ssh/server-1-admin:
```

## Authors
Mr. Johnson


## Acknowledgments
* [https://askubuntu.com/questions/53553/how-do-i-retrieve-the-public-key-from-a-ssh-private-key](https://askubuntu.com/questions/53553/how-do-i-retrieve-the-public-key-from-a-ssh-private-key)
* [https://www.howtogeek.com/424510/how-to-create-and-install-ssh-keys-from-the-linux-shell/](https://www.howtogeek.com/424510/how-to-create-and-install-ssh-keys-from-the-linux-shell/)
* [https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-1604](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-1604)
* [https://www.freecodecamp.org/news/the-ultimate-guide-to-ssh-setting-up-ssh-keys/](https://www.freecodecamp.org/news/the-ultimate-guide-to-ssh-setting-up-ssh-keys/)
* [https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys)