---
layout: default
title: Dedicated Quake3 servers on Ubuntu
parent: Linux
nav_order: 2
---
*WORK IN PROGRESS as of 2019-08-01*
## How to set up dedicated Quake 3 Arena server(s) with different mods on Ubuntu Server 16.04
{: .no_toc }
I used to play Quake III: Arena with my friends when I grew up (way back in the 00s). I wanted to introduce Q3 to my kid and the easiest solution would be to just go online and download Quake Live from Steam (I remember trying QL when it was playable through the browser). However, since I am using macOS and QL is now only available on Windows, I had to think of alternatives.


I took the challenge of trying to host my own dedicated Q3 server with the latest mods so I could bring some QUAD POWER, DENIED and M-M-M-MMULTIKILL and show my kid how a proper KILLING-SPREE is done.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---

## Getting started
Googling ‘how to set up a dedicated quake 3 server’ only got my that far.. . I wanted to run multiple dedicated Q3 servers (Free For All/Duel/Team Death Match on a single box, with the latest and best mods with proper server settings and an emphasis on network-security.

Read on to learn how I was able to host multiple Q3 servers with different mods on a single virtualized Ubuntu Server 16.04 installation. 

### Prerequsites
Some basic knowledge of UNIX commands (and using SSH) and FileZilla (or any other program) for easy transfering of files.

* Ubuntu Server 16.04 - [http://releases.ubuntu.com/16.04/](http://releases.ubuntu.com/16.04/) (64-bit PC (AMD64) server install image)
* pak03.pak from the original Quake III: Arena installation CD (the Quake 3 engine is open source. The Quake III: Arena game itself is not for free)

I am running Q3 server instances on a Ubuntu Server 16.04 installation as a virtual machine (bhyve). This game is now almost 20 years old, so the system requirements I needed to allocate for my virtual machine are pretty low - [https://www.systemrequirementslab.com/cyri/requirements/quake-iii-arena/11592](https://www.systemrequirementslab.com/cyri/requirements/quake-iii-arena/11592). I have 1 CPU and 2GB of RAM allocated.

### Disclaimer
If you don’t write 
```
sudo rm -rf / 
```
in your terminal, following the guide below should be pretty safe, but I take no responsibility for whatever comes your way :-) 

And as with every guide, I recommend to read through all of it before you begin.


## Installation of Ubuntu Server 16.04
If you do not have a dedicated machine to use for the installation of Ubuntu, you could download VirtualBox from Oracle ([https://www.virtualbox.org](https://www.virtualbox.org)) or use (any) another hypervisor for your platform ([https://en.wikipedia.org/wiki/Hypervisor](https://en.wikipedia.org/wiki/Hypervisor).

Ubuntu tutorials has an excellent guide for installing Ubuntu Server 16.04; [https://tutorials.ubuntu.com/tutorial/tutorial-install-ubuntu-server-1604](https://tutorials.ubuntu.com/tutorial/tutorial-install-ubuntu-server-1604).

After following the guide above, update and upgrade your Ubuntu Server 16.04 installation:
```bash
sarge@quake3server:~$ sudo apt-get update && sudo apt-get upgrade
```

## Configuration
I thought it would be an easy quest of just importing some Q3 binaries from my old ripped up CD and do some tweaking and voilà- I have some executables to configure into a server. But it quickly came to my attention that the Q3 scene has gone (in many directions). 
The latest official engine/point release for Q3 from id Software is 1.32c, released all the way back in 200x. But thankfully to the Open Source Community, updates have been made making us able to enjoy Q3 on the latest and greatest hardware. 


### Your preferred engine? Say what?
What I (think I) found out, is that there are several engines out there which can start your Q3 server slash game these days. id Software stopped fixing bugs, security issues, and adding features to Q3 more than a decade ago. 
Now you have something called Quake3 1.32e [http://edawn-mod.org/forum/viewtopic.php?f=5&t=7]((http://edawn-mod.org/forum/viewtopic.php?f=5&t=7)), Quake III Kenny Edition [https://github.com/kennyalive/Quake-III-Arena-Kenny-Edition](https://github.com/kennyalive/Quake-III-Arena-Kenny-Edition), CNQ3 1.50 [https://playmorepromode.com/downloads]((https://playmorepromode.com/downloads)), a quake3-server package from Ubuntu ([https://packages.ubuntu.com/trusty/games/quake3-server](https://packages.ubuntu.com/trusty/games/quake3-server)) and a bit further Googling sent me to the project ioquake3 which is a free software FPS game engine based on Q3 for Windows, Linux and macOs. 

I chose to base my server on the ioquake3 project because they had a SysAdmin Guide for Linux, it is compatible with mods, is available under Linux, Windows and macOS and ioquake3 have added many features and fixed “too many bugs to count” . Go have a look at the project here: [https://ioquake3.org/](https://ioquake3.org/).

What I did next, is that I  followed the Sys Admin Guide over at ioquake3 [http://wiki.ioquake3.org/Sys_Admin_Guide](http://wiki.ioquake3.org/Sys_Admin_Guide) - you should use that for references when you start.

This build script we are downloading in just a second, requires `git`, `make` and `gcc` to be installed, so we do that now after we have upgraded and updated our Ubuntu installation (+ we are installing the `screen` utility):
```bash
sarge@quake3server:~$ sudo apt-get install git && sudo apt-get install make && sudo apt-get install gcc && sudo apt-get install screen
```
Create a new user (ioq3srv) for your ioquake3 server to run as:
```bash
sarge@quake3server:~$ sudo adduser ioq3srv 
```
Change our current user to the newly created user: 
```bash
sarge@quake3server:~$ su ioq3srv 
```
Change directory to the newly created user’s home folder:
```bash
ioq3srv@quake3server:~/home/sarge$ cd /home/ioq3srv
ioq3srv@quake3server:~$ pwd
/home/ioq3srv
```
Download `server_compile.sh`:
```bash
ioq3srv@quake3server:~$ wget https://raw.githubusercontent.com/ioquake/ioq3/master/misc/linux/server_compile.sh
```
Run the .sh and compile ioquake3:
```bash
ioq3srv@quake3server:~$ sh server_compile.sh
```
Answer yes and press enter when asked “Are you ready to compile ioquake3 from [https://github.com/ioquake/ioq3.git](https://github.com/ioquake/ioq3.git), and have it installed into ~/ioquake3? “ 

After ioquake3 has been compiled, open up FileZilla which we are going to use for the transfer of our pak0.pk3 file. Create a new site in FileZilla with Protocol: SFTP - SSH File Transfer Protocol and put in your credentials for the ioq3srv user (and obviosly the IP address of the Ubuntu Server 16.04 installation- default port for SSH is 22).
When you are connected, transfer pak0.pk3 to the `baseq3` sub-directory of ioquak3; `/home/ioq3srv/ioquake3/baseq3/`

The pak0.pk3 file resides inside your baseq3 folder on your original Quake III: Arena installation media.

When that is done, you need to transfer some more files. Download the patch zip file from [https://ioquake3.org/extras/patch-data/](https://ioquake3.org/extras/patch-data/) and unzip the contents into your baseq3 and missionpack sub-directories of ioquake3. 

Go back to your Ubuntu installation to `wget start_server.sh`; 
```bash
ioq3srv@quake3server:~$ wget https://raw.githubusercontent.com/ioquake/ioq3/master/misc/linux/start_server.sh
```
### Command line arguments

Take a look at the start_server.sh file by using the command `more`:

```bash
ioq3srv@quake3server:~$ more start_server.sh 
#!/bin/sh
echo "Edit this script to change the path to ioquake3's dedicated server executable and which binary if you aren't on x86_64.
"
echo "Set the sv_dlURL setting to a url like http://yoursite.com/ioquake3_path for ioquake3 clients to download extra data."
# sv_dlURL needs to have quotes escaped like \"http://yoursite.com/ioquake3_path\" or it will be set to "http:" in-game.
~/ioquake3/ioq3ded.x86_64 +set dedicated 2 +set sv_allowDownload 1 +set sv_dlURL \"\" +set com_hunkmegs 64 "$@"
```
What we want to edit right now is the `+set dedicated 2` parameter.  `+set dedicated 2` is public Internet, 1 is LAN/connect by IP (the default), 0 is non-dedicated (client binary only). It must be set in the command line arguments (which means after ~/ioquake3/ioq3ded.x8664 in the `startserver.sh` file). For now we do not want to broadcast our server to the world, so we change that to 1. Use your favourite text editor (‘nano’ is a good and easy choice), and alter it to 1. 

`sv_allowDownload 1` means that players connected to your server are able to download missing maps automatically, thats good. The `+set svdlURL` enables those downloads through http, but we wont use that, so we’ll remove it. `com_hunkmegs` sets the amount of memory you want ioq3ded.x86_64 to reserve for game play dedicated server memory optimizations. We’ll change it to 128.

Use `nano` or any other text editor to alter the start_server.sh file:
```bash
ioq3srv@quake3server:~$ nano start_server.sh 
~/ioquake3/ioq3ded.x86_64 +set dedicated 1 +set sv_allowDownload 1 +set com_hunkmegs 128 "$@"
```
To start a server, you go:
```bash
ioq3srv@quake3server:~$ sh start_server.sh 
```
and then you can write `map q3dm17` in your server’s console. To stop the server, press ctrl + c within that window.
Now you have created a Q3 server with _default_ server configuration settings, and started a game on q3dm17 where players can join by /connect ip-adress:27960 in Q3's console!

### Configuration files
The first time you start your server, a `q3config_server.cfg` is being created. This server config is a default one and it is created in `/home/ioq3srv/.q3a/baseq3/`. 

You can check out the default Q3 server settings by changing directory and use the `more` command:
```bash
ioq3srv@quake3server:~$ cd .q3a/baseq3/
ioq3srv@quake3server:~/.q3a/baseq3$ more q3config_server.cfg 
```
Because I wanted proper server settings, I searched online and looked for some “official” server settings from big tournaments such as QuakeCon, ESWC or CPL, but I was unsuccessfull. The closest thing I found was a guy ([http://www.esreality.com/post/2848596/new-q3-pro-osp-configs/](http://www.esreality.com/post/2848596/new-q3-pro-osp-configs/) who had some server config files stored in a shared dropbox amongst config files from pro players ([https://www.dropbox.com/s/6z2iz4ugd0zket6/configs.zip?dl=0&filesubpath=%2Fquake3](https://www.dropbox.com/s/6z2iz4ugd0zket6/configs.zip?dl=0&filesubpath=%2Fquake3). But, are those server configs valid? Now 10+ years after it presumably dates from? And is that config for Q3 or does it include settings for osp_mod/CPMA? Or any other mods? Read on. .

## The Contest
Correct me if I am mistaken,

with Quake III: Arena, id Software allowed you to modify the game. When I was playing back in 2003, there were several different mods I played. There was a mod called osp_mod from Orange Smoothie Production ([http://www.orangesmoothie.org](http://www.orangesmoothie.org)), Challenge ProMode Arena (CPMA) ([https://en.wikipedia.org/wiki/Challenge_ProMode_Arena)](https://en.wikipedia.org/wiki/Challenge_ProMode_Arena)), I remember I played Rocket Arena as well ([https://en.wikipedia.org/wiki/Rocket_Arena](https://en.wikipedia.org/wiki/Rocket_Arena)) (that was awesome) and I had a couple of hours trickjumping in DeFrag too ([https://en.wikipedia.org/wiki/DeFRaG](https://en.wikipedia.org/wiki/DeFRaG)).
Writing this how-to, I discovered there’s many more mods that have come out since then; Open Arena ([https://en.wikipedia.org/wiki/OpenArena)](https://en.wikipedia.org/wiki/OpenArena)), Smokin Guns ([https://en.wikipedia.org/wiki/Smokin%27_Guns](https://en.wikipedia.org/wiki/Smokin%27_Guns), Quake 3 Fortress [https://www.moddb.com/mods/q3f](https://www.moddb.com/mods/q3f) and Tremulos ([https://en.wikipedia.org/wiki/Tremulous](https://en.wikipedia.org/wiki/Tremulous)). It might even be many more.

I don’t really know what the latest mods the Quake scene/pro gamers/compos have been using are. Is the Q3 scene completely dead, everyone is either playing QL or have they all shifted over to Quake Champions? (QC is unfortnuately not playable on macOS either).

### All your mods are belongs to us
Scouring through various sites, what I found out was that the CPMA scene was fairly active ([https://twitter.com/playmorepromode](https://twitter.com/playmorepromode). The CPMA 1.50 mod was released 7th of January 2018. Back when I played CPMA, coming from osp_mod, I did not get familiar with the air-control in the gameplay that much (and I honestly do not remember if I liked it or not). But I see that you are able to have the CPMA graphical user interface with VQ3 gameplay (VQ3 is what people refer to as ‘vanilla quake’ -  meaning normal gameplay with no alteriations to physics or jumping). 
Also, with CPMA, you are able to have all the typical game modes such as FFA, TDM, DUEL and you are also able to play a style of ‘Rocket Arena’ with a game mode CPMA calls Clan Arena. Really cool! More information can be found here; [https://esreality.com/post/1151879/re-cpl-chooses-cpma-vq3/](https://esreality.com/post/1151879/re-cpl-chooses-cpma-vq3/), [http://www.esreality.com/?a=post&id=1155489](http://www.esreality.com/?a=post&id=1155489) and [https://forum.lowyat.net/topic/332890/all](https://forum.lowyat.net/topic/332890/all).

### ioquake3 with Challenge ProMode mod
Follow this guide here [https://playmorepromode.com/guides/cpma-cnq3-installation](https://playmorepromode.com/guides/cpma-cnq3-installation) and download [https://cdn.playmorepromode.com/files/cpma-mappack-full.zip](https://cdn.playmorepromode.com/files/cpma-mappack-full.zip) and [https://playmorepromode.com/files/latest/cpma](https://playmorepromode.com/files/latest/cpma) and place those files in their respective folders with FileZilla. 

The folks over at CPMA has been kind enough to give us some examples of a proper server config: [https://playmorepromode.com/guides/cnq3-dedicated-server-guide](https://playmorepromode.com/guides/cnq3-dedicated-server-guide) (Yay! I finally found something which looks like legit configs), 

After you have downloaded the files nessecary for running the CPMA mod, we are going to make ourself a `start_server.sh` file to make ioquake3 start a server with the CPMA mod:
```bash
ioq3srv@quake3server:~$ nano start_CPMA_duelserver.sh
```
#### Duel server with VQ3 physics on CPMA with ioquake3

I am first setting up a duel server. The content of my `start_CPMA_duelserver.sh` file looks like this:
```bash
#!/bin/sh
ip="192.168.5.129"
port="27961"
name="CPMA duel server with VQ3"

echo running $name on $ip:$port
~/ioquake3/ioq3ded.x86_64 +set dedicated 1 \
        +set fs_game cpma \
        +set net_port 27961 \
        +set ttycon 1 \
        +set developer 0 \
        +exec CPMA_duelserver.cfg \
        +map cpm3a
```
Now we have to create our server config which is executed, `CPMA_duelserver.cfg`. The server configs for CPMA is loaded from `/ioquake3/cpma/` folder and not from the `.q3a/baseq3/` folder.
```bash
ioq3srv@quake3server:~$ cd ioquake3/cpma/
ioq3srv@quake3server:~/ioquake3/cpma$ nano CPMA_duelserver.cfg 
// make sure to update these!
sets .admin.        "Sarge"      // server browser info
sets .location      "PARIS"    // server browser info
set sv_hostname     "DUEL with VQ3"    // server browser info
set ref_password    "none"              // "none" means referee/admin access is disabled
set rconPassword    ""                  // "" means rcon access is disabled
set server_motd1    "Welcome to Fhloston Paradise!" // message of today
set server_motd2    "Mul-ti-pass"
set server_motd3    ""
// more useful settings:
set sv_pure                 "1"         // *always* set to 1 for online servers
set snaps                   "30"        // leave at 30
set sv_strictAuth           "0"         // enables CD-key checks
set server_record           "0"         // bitmask - forces players to record demos, take screenshots, etc
set server_chatfloodprotect "0"         // max. chat messages per second, 0 means no limit
set sv_maxrate              "30000"     // good range for players is 25k to 30k
set server_gameplay         "VQ3"       // change between VQ3, CPM, PMC, CQ3 or VQ3
set server_maxpacketsmin    "100"       // ideally cl_maxPackets 125, but allow a bit lower
set server_maxpacketsmax    "125"       // ideally cl_maxPackets 125
set server_ratemin          "25000"     // good range for players is 25k to 30k
set server_optimisebw       "1"         // reduces bandwidth a lot but can't see players through portals
set log_pergame             "0"         // opens a new timestamped log file for each game
set match_readypercent      "100"       // min. % of players that must be ready for a match to start
set g_gametype              "1"         // 1 is duel
set sv_maxclients           "8"         // max. player count
set mode_start              "1v1"       // game mode to start with
set sv_privateClients       "0"         // number of private slots reserved
set sv_privatePassword      ""          // password for the private slots
set server_availmodes       "1v1"       // restrict the server to only 1v1

// voting
set g_allowVote             "1"        // allows clients voting on server settings
set vote_percent            "70"       // specifiec the percentage of accepting clients needed for a vote to pass
set vote_allow_allcaptain   "0"       
set vote_allow_armor        "0"
set vote_allow_armorsystem  "0"
set vote_allow_dropitems    "0"
set vote_allow_fallingdamage "0"
set vote_allow_flood        "0"
set vote_allow_footsteps    "0"
set vote_allow_hook         "0"
set vote_allow_instagib     "1"
set vote_allow_kick         "0"
set vote_allow_map          "1"
set vote_allow_gameplay     "vq3" // only vq3 is allowed
```
We are going to display a custom MOTD (message of today) with `server_motdfile` and it has to be placed in our `/home/ioq3srv/.q3a/cpma/` folder
```bash
ioq3srv@quake3server:~$ nano .q3a/cpma/duelservermotd.txt 
Welcome to Dallas' (FR) DUEL (VQ3) server
This server runs on regular tourney settings 
Enjoy your stay here at Fhloston Paradise!
```
To start this server, you just write 
```bash
ioq3srv@quake3server:~$ sh start_CPMA_duelserver.sh
```
#### Team Death Match server with VQ3 physics in CPMA on ioquake3
```bash
ioq3srv@quake3server:~$ more start_CPMA_tdmserver.sh 
#!/bin/sh
ip="192.168.5.129"
port="27960"
name="CPMA tdm server with VQ3"
echo running $name on $ip:$port
~/ioquake3/ioq3ded.x86_64 +set dedicated 1 \
        +set fs_game cpma \
        +set net_port 27960 \
        +set ttycon 1 \
        +set developer 0 \
        +exec CPMA_tdmserver.cfg \
        +map pro-q3dm6
```

```bash
ioq3srv@quake3server:~$ more ioquake3/cpma/CPMA_tdmserver.cfg 
// make sure to update these!
sets .admin.        "SARGE"      // server browser info
sets .location      "FR"    // server browser info
set sv_hostname     "Dallas tdm (VQ3)"    // server browser info
set ref_password    "zxc"              // "none" means referee/admin access is disabled
set rconPassword    "qwe"                  // "" means rcon access is disabled
set g_motd          "Welcome to Fhloston Paradise!" // Message of today
// more useful settings:
set sv_pure                 "1"         // *always* set to 1 for online servers
set snaps                   "30"        // leave at 30
set sv_strictAuth           "0"         // enables CD-key checks
set server_record           "0"         // bitmask - forces players to record demos, take screenshots, etc
set server_chatfloodprotect "0"         // max. chat messages per second, 0 means no limit
set sv_maxrate              "30000"     // good range for players is 25k to 30k
set sv_allowDownload        "0"         // enables id's super slow download system
set server_gameplay         "VQ3"       // only change if you want your server to be lame
set server_maxpacketsmin    "100"       // ideally cl_maxPackets 125, but allow a bit lower
set server_maxpacketsmax    "125"       // ideally cl_maxPackets 125
set server_ratemin          "25000"     // good range for players is 25k to 30k
set server_optimisebw       "1"         // reduces bandwidth a lot but can't see players through portals
set log_pergame             "0"         // opens a new timestamped log file for each game
set match_readypercent      "100"       // min. % of players that must be ready for a match to start
set g_gametype              "1"         // 1 is duel
set sv_maxclients           "24"        // max. player count
set mode_start              "tdm"       // game mode to start with
set sv_privateClients       "1"         // number of private slots reserved
set sv_privatePassword      "asdf"          // password for the private slot
set server_availmodes       "tdm"       // restrict the server to only tdm
// voting
set g_allowVote             "1"        // allows clients voting on server settings
set vote_percent            "70"       // specifiec the percentage of accepting clients needed for a vote to pass
set vote_allow_allcaptain   "0"       
set vote_allow_armor        "0"
set vote_allow_armorsystem  "0"
set vote_allow_dropitems    "0"
set vote_allow_fallingdamage "0"
set vote_allow_flood        "0"
set vote_allow_footsteps    "0"
set vote_allow_hook         "0"
set vote_allow_instagib     "1"
set vote_allow_kick         "0"
set vote_allow_map          "1"
set vote_allow_gameplay     "tdm"
```
We are going to display a custom MOTD (message of today) with server_motdfile and it has to be placed in our /home/ioq3srv/.q3a/cpma/ folder
```bash
ioq3srv@quake3server:~$ nano .q3a/cpma/tdmservermotd.txt 
Welcome to Dallas' (FR) TDM (VQ3) server
This server runs on regular TDM settings 
Enjoy your stay here at Fhloston Paradise!
```
To start this CPMA TDM server, you just write 
```bash
ioq3srv@quake3server:~$ sh start_CPMA_tdmserver.sh
```
#### Clan Arena (Rocket Arena 3) with VQ3 physics on CPMA with ioquake3
In order to play the classic RA3 games in CPMA with gametype CA, you have to copy over the ra3map.pk3 to ioquake3/baseq3 folder. The latest version I found released of RA3 is 1.80; [http://www.esreality.com/?a=post&id=1651776](http://www.esreality.com/?a=post&id=1651776) / [https://www.quake3.fr/index.php?f_id_contenu=1156&f_id_type=13](https://www.quake3.fr/index.php?f_id_contenu=1156&f_id_type=13) / [https://games.square-r00t.net/downloads/quake/quakeIII/arena/readme.txt](https://games.square-r00t.net/downloads/quake/quakeIII/arena/readme.txt). 
Also, I recommend reading this post [http://levelsofdetail.net/2010/07/this-is-a-post-about-rocket-arena-3/](http://levelsofdetail.net/2010/07/this-is-a-post-about-rocket-arena-3/).

Download RA3 and in its folder `arena` you’ll find files called `ra3map1.pk3` up to `ra3map20.pk3`. Copy all those files over to `/home/ioq3srv/ioquake3/baseq3`. 
If you would like to set up a server as a pure RA3 mod, just copy the `arena` folder to `/home/ioq3srv/ioquake3/` (just as you did with cpma - and then it will be just to alter `+set fs_game arena`.. .. I’ll show you later. 
```bash
ioq3srv@quake3server:~$ more start_CPMA_caserver.sh 
#!/bin/sh
ip="192.168.5.129"
port="27962"
name="CPMA ca server with VQ3"
echo running $name on $ip:$port
~/ioquake3/ioq3ded.x86_64 +set dedicated 1 \
        +set fs_game cpma \
        +set net_port 27962 \
        +set ttycon 1 \
        +set developer 0 \
        +exec CPMA_caserver.cfg \
        +map ra3map1   
```
[https://r3dux.org/2010/05/how-to-fix-missing-bots-in-ioquake3-in-linux/](https://r3dux.org/2010/05/how-to-fix-missing-bots-in-ioquake3-in-linux/)

Reading this guide here, [https://playmorepromode.com/guides/cpma-custom-modes](https://playmorepromode.com/guides/cpma-custom-modes), to get a feel of the ‘old’ RA3 behavoir in CA - you should create a file called RA3.cfg and save it in this folder `~/ioquake3/cpma/modes/`, or copy `modes/sample/RA3.cfg` to the `modes/` folder.
```bash
ioq3srv@quake3server:~/home/ioq3srv/ioquake3/cpma/modes$ nano RA3.cfg
type 5                    # Clan Arena derivative
limit 5                    # first team to 5 rounds wins
ammo 200,100,20,50,150,50,100,0        # lots of GL/PG/RG ammo
armor 100                # 2 rails = YUO AR TEH WINNAR!
items -BFG
fallingdamage 1
startrespawn 0
selfdamage 2
teamdamage 0                # no team damage in spamfests
```
Looking through `server.cfg` from RA3 1.80, I found some inconsistency to the provided `RA3.cfg` from CPMA. For instance, `server.cfg` says RA3 does not use fraglimit, only timelimit.  And looking through demos of guys playing RA3 on youtube, I found out you start with 100 health and 100 armor, with no team damage or self damage and no falling damage. 
The ammos in the provided config was correct.

Our final custome mode aka RA3.cfg looks like this:
```bash
type 5
timelimit 30
ammo 200,100,20,50,150,50,100,0
armor 100
fallingdamage 1
startrespawn 0
selfdamage 0
teamdamage 0
items -BFG
```
In our `CPMAra3server.cfg` we point to RA3.cfg under `mode_start` and `server_availmodes`.
```bash
ioq3srv@quake3server:~/ioquake3/cpma$ nano CPMA_ra3server.cfg 
// make sure to update these!
sets .admin.        "SARGE"      // server browser info
sets .location      "PARIS"    // server browser info
set sv_hostname     "Dallas RA3 (VQ3)"    // server browser info
set ref_password    "zxc"              // "none" means referee/admin access is disabled
set rconPassword    "qwe"                  // "" means rcon access is disabled
set admin_log "ra3server.log" // log file located in .q3a/cpma/
set server_motdfile "ra3servermotd.txt" //MOTD located in .q3a/cpma/
// more useful settings:
set sv_pure                 "1"         // *always* set to 1 for online servers
set snaps                   "30"        // leave at 30
set sv_strictAuth           "0"         // enables CD-key checks
set server_record           "0"         // bitmask - forces players to record demos, take screenshots, etc
set server_chatfloodprotect "0"         // max. chat messages per second, 0 means no limit
set sv_maxrate              "30000"     // good range for players is 25k to 30k
set server_gameplay         "VQ3"       // only change if you want your server to be lame
set server_maxpacketsmin    "100"       // ideally cl_maxPackets 125, but allow a bit lower
set server_maxpacketsmax    "125"       // ideally cl_maxPackets 125
set server_ratemin          "25000"     // good range for players is 25k to 30k
set server_optimisebw       "1"         // reduces bandwidth a lot but can't see players through portals
set log_pergame             "0"         // opens a new timestamped log file for each game
set match_readypercent      "100"       // min. % of players that must be ready for a match to start
set sv_maxclients           "24"        // max. player count
set mode_start              "RA3"       // game mode to start with
set sv_privateClients       "1"         // number of private slots reserved
set sv_privatePassword      "asdf"          // password for the private slot
set server_availmodes       "RA3"       // restrict the server to only CA
// voting
set g_allowVote             "1"        // allows clients voting on server settings
set vote_percent            "70"       // specifiec the percentage of accepting clients needed for a vote to pass
set vote_allow_allcaptain   "0"
set vote_allow_armor        "0"
set vote_allow_armorsystem  "0"
set vote_allow_dropitems    "0"
set vote_allow_fallingdamage "0"
set vote_allow_flood        "0"
set vote_allow_footsteps    "0"
set vote_allow_hook         "0"
set vote_allow_instagib     "1"
```
To ensure that the specific RA3.cfg settings are loaded when you are starting the RA3, you have to alter the configuration files for the specific map that you are using. The configuration files for each specific map is located in `cpma/cfg-maps`. For instance, this is how the default `ra3map1.cfg` shipped from cpma looks like;
```bash
ioq3srv@quake3server:~/ioquake3/cpma/cfg-maps$ more ra3map1.cfg 
# make sure that browsers know the server is CA
showas  5
# Evolution
arena   1
mode    DA
# Thunderstruck
arena   2
mode    DA
# Canned Heat
arena   3
mode    DA
# Theatre Of Pain
arena   4
mode    CA
warmup  0
```
`ra3map1.cfg` is a MA map, Multi-Arena map (a map which consists of multiple arenas). We have to change the mode from `DA` and `CA`to `RA3` to envoke the specific settings from `RA3.cfg` in each arena:
```bash
ioq3srv@quake3server:~/ioquake3/cpma/cfg-maps$ nano ra3map1.cfg
# make sure that browsers know the server is CA
showas  5
# Evolution
arena   1
mode    RA3
# Thunderstruck
arena   2
mode    RA3
# Canned Heat
arena   3
mode    RA3
# Theatre Of Pain
arena   4
mode    RA3
warmup  0
```
We are going to display a custom MOTD (message of today) with `server_motdfile` and it has to be placed in our `/home/ioq3srv/.q3a/cpma/` folder
```bash
ioq3srv@quake3server:~$ nano .q3a/cpma/ra3servermotd.txt 
Welcome to Dallas' (FR) RA3 (VQ3) server
This server runs on regular RA3 settings 
Enjoy your stay here at Fhloston Paradise!
```
To start this server, you just write 
```bash
ioq3srv@quake3server:~$ sh start_CPMA_ra3server.sh
```
### Maps
If you are looking for special maps, such as ztn3tourney1 ([https://lvlworld.com/review/id:800](https://lvlworld.com/review/id:800)), you have to place those maps in the `ioquake3/baseq3/folder` on your server. A really cool map for TDM play is ospdm5 ([http://ws.q3df.org/map/ospdm5)](http://ws.q3df.org/map/ospdm5)) - just place the whole shebang (`ospmaps0.pk3` from previous link) in you baseq3 folder. 

If `sv_downloadAllow` is set to 1, clients connecting to the server will be allowed to download missing maps. 

PS: Remember that your client has to enable Automatic download (SETUP > GAME OPTIONS > Automatic Downloading “on”), or type `/cl_allowdownload 1` in Q3 console from the loaded _mod_  to fully allow downloads to the Q3 client. 

## Using screen to start the Q3 servers
Read this guide for a good practise on how to start and monitor your Q3 server: [https://playmorepromode.com/guides/cnq3-dedicated-server-guide](https://playmorepromode.com/guides/cnq3-dedicated-server-guide).

## Securing our server
With our sudo user, we will issue a command called netstat ([https://en.wikipedia.org/wiki/Netstat](https://en.wikipedia.org/wiki/Netstat)), which shows us our open ports and current connections. 
```bash
sarge@quake3server:~$ netstat -an | more
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 192.168.100.100:22      192.168.100.150:58566   ESTABLISHED
tcp6       0      0 :::22                   :::*                    LISTEN     
udp        0      0 0.0.0.0:68              0.0.0.0:*                          
udp        0      0 0.0.0.0:27960           0.0.0.0:*                          
udp        0      0 0.0.0.0:27961           0.0.0.0:*                          
udp        0      0 0.0.0.0:27962           0.0.0.0:*                          
```
In the excerpt above, you see that the localhost is open to anyone on port 22 (SSH) using protocol TCP with both IPv4 (tcp) and IPv6 (tcp6), listening from any address (0.0.0.0:*). There is an active connection from IP 192.168.1.150 to the ubuntu server (that line is the SSH Session from my macOS terminal). 

Port 68 is for issuing the DHCP protocol. 

27960, 27961 and 27962 is our Quake 3 servers. 

What I want to do, is that I want to use the default Firewall for Ubuntu, UFW (Uncomplicated Firewall) to deny all other access to port 22 except from my Local Area Network.  

The status of UFW:
```bash
sarge@quake3server:~$ sudo ufw status numbered
[sudo] password for Sarge: 
Status: inactive
```
A good strategy for all network security measurements is to deny all incoming and outgoing connections, and only open the ports that are essential for our running services (which is SSH and our Q3 servers, and the occasional exchange of DHCP Messages, NTP (Network Time Protocol and DNS). 
```bash
sarge@quake3server:~$ sudo ufw default deny incoming 
sarge@quake3server:~$ sudo ufw default allow outgoing
```
Add a rule for only our internal LAN to be able to access the SSH service:
```bash
sarge@quake3server:~$ sudo ufw allow from 192.168.100.0/24 to any port 22 proto tcp
Rules updated
```
Add allow rules for port 27960, 27961 and 27962 on UDP IPv4 with:
```bash
sarge@quake3server:~$ sudo ufw allow 27960:27962/udp
Rule added
Rule added (v6)
```
Remember to double check that you have written the statement above correct, because when you enable ufw, you might lock yourself out.
```bash
sarge@quake3server:~$ sudo ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```
Now show the status of UFW:
```bash
sarge@quake3server:~$ sudo ufw status numbered
Status: active
     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    192.168.100.0/24    
[ 2] 27960:27962/udp            ALLOW IN    Anywhere                  
[ 3] 27960:27962/udp (v6)       ALLOW IN    Anywhere (v6)  
```
I am not using IPV6, so I’ll just delete the rule number 3:
```
sarge@quake3server:~$ sudo ufw delete 3
```
## Getting players to play on your server

Read this .pdf [http://caia.swin.edu.au/reports/070730A/CAIA-TR-070730A.pdf](http://caia.swin.edu.au/reports/070730A/CAIA-TR-070730A.pdf) for a brief description on how the server-discovery protocol for Q3 works. 

```bash
seta sv_master1            "master3.idsoftware.com" //
seta sv_master2            "master0.gamespy.com" //
seta sv_master3            "q3master.barrysworld.com:27950" //Master Servers
seta sv_master4            "q3master.gamesinferno.com" //
seta sv_master5            "wfamaster.planetice.net" //
seta sv_master1 "master0.gamespy.com:28900"     // master servers where the server registers itself
seta sv_master2 "master.gamershut.de:27950" 	//   to be found by players.
seta sv_master3 "master.gnw.de:27950"		//   use +set dedicated x to tell the server whether or not
seta sv_master4 "master3.idsoftware.com:27950"  //   to register itself there, x = 2 : register x = 1 : don't
```
[https://ioquake3.org/2016/05/19/the-id-master-server-is-offline/](https://ioquake3.org/2016/05/19/the-id-master-server-is-offline/).

##  Acknowledgments
URLs that I found useful at some time when I was writing this howto: 

* [https://www.howtoinstall.co/en/ubuntu/xenial/quake3-server](https://www.howtoinstall.co/en/ubuntu/xenial/quake3-server)
* [http://it.rcmd.org/networks/q3_install/q3_linux_server_howto.php#step2](http://it.rcmd.org/networks/q3_install/q3_linux_server_howto.php#step2)
* [https://www.lucaswilliams.net/index.php/2017/01/15/quake3-arena-dedicated-server-on-ubuntu-16-04/](https://www.lucaswilliams.net/index.php/2017/01/15/quake3-arena-dedicated-server-on-ubuntu-16-04/)
* [https://launchpad.net/ubuntu/xenial/+package/quake3-server](https://launchpad.net/ubuntu/xenial/+package/quake3-server)
* [https://packages.ubuntu.com/xenial/games/quake3-server](https://packages.ubuntu.com/xenial/games/quake3-server)
* [https://www.howtoinstall.co/en/ubuntu/xenial/quake3-server?action=remove](https://www.howtoinstall.co/en/ubuntu/xenial/quake3-server?action=remove)
* [http://wiki.ioquake3.org/Sys_Admin_Guide#Linux](http://wiki.ioquake3.org/Sys_Admin_Guide#Linux)
* [https://kb.firedaemon.com/support/solutions/articles/4000086997-quake-iii](https://kb.firedaemon.com/support/solutions/articles/4000086997-quake-iii)
* [https://www.quake3world.com/q3guide/servers.html](https://www.quake3world.com/q3guide/servers.html)
* [https://diginc.us/linux/2009/my-linux-quake-3-dedicated-server-setup-notes-ubuntu-9-04-server/](https://diginc.us/linux/2009/my-linux-quake-3-dedicated-server-setup-notes-ubuntu-9-04-server/)
* [http://planetquake.gamespy.com/Viewdd90.html?view=Articles.Detail&id=151](http://planetquake.gamespy.com/Viewdd90.html?view=Articles.Detail&id=151)
* [https://quake.fandom.com/wiki/Quake_III_Version_History](https://quake.fandom.com/wiki/Quake_III_Version_History)
* [http://www.edawn-mod.org/binaries/quake3e-changes.txt](http://www.edawn-mod.org/binaries/quake3e-changes.txt)
* [http://www.esreality.com/?a=post&id=2036991#pid2036991](http://www.esreality.com/?a=post&id=2036991#pid2036991)
* [https://github.com/roguephysicist/q3a-server](https://github.com/roguephysicist/q3a-server)
* [http://www.esreality.com/post/2908690/cnq3-1-50-released/](http://www.esreality.com/post/2908690/cnq3-1-50-released/)
* [https://swissmacuser.ch/how-you-want-to-run-quake-iii-arena-in-2018-with-high-definition-graphics-120-fps-on-5k-resolution/#.XIjNby2ZN2Q](https://swissmacuser.ch/how-you-want-to-run-quake-iii-arena-in-2018-with-high-definition-graphics-120-fps-on-5k-resolution/#.XIjNby2ZN2Q)
* [http://www.esreality.com/?a=post&id=1153598](http://www.esreality.com/?a=post&id=1153598)
* [http://www.esreality.com/post/354034/n-a/](http://www.esreality.com/post/354034/n-a/)
* [http://www.esreality.com/post/2848596/new-q3-pro-osp-configs/](http://www.esreality.com/post/2848596/new-q3-pro-osp-configs/)
* [http://www.esreality.com/index.php?a=post&id=2036455](http://www.esreality.com/index.php?a=post&id=2036455)
* [https://www.dropbox.com/s/6z2iz4ugd0zket6/configs.zip?dl=0&file_subpath=%2Fquake3](https://www.dropbox.com/s/6z2iz4ugd0zket6/configs.zip?dl=0&file_subpath=%2Fquake3)
* [http://wiki.ioquake3.org/Sys_Admin_Guide#Linux](http://wiki.ioquake3.org/Sys_Admin_Guide#Linux)
* [https://www.giantbomb.com/quake-iii-arena/3030-3874/](https://www.giantbomb.com/quake-iii-arena/3030-3874/)
* [https://www.speedguide.net/port.php?port=27960](https://www.speedguide.net/port.php?port=27960)
* [https://github.com/m3fh4q/Quake3ArenaCPMADedicatedServerGuideLinux](https://github.com/m3fh4q/Quake3ArenaCPMADedicatedServerGuideLinux)
* [http://www.esreality.com/index.php?a=post&id=1180255](http://www.esreality.com/index.php?a=post&id=1180255)
* [https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-16-04)
* [http://esreality.com/?a=post&id=2210880](http://esreality.com/?a=post&id=2210880)
* [https://nohup.run/ioquake3-botaisetupclient-failed-error-on-64-bit-linux/](https://nohup.run/ioquake3-botaisetupclient-failed-error-on-64-bit-linux/)
* [https://www.katsbits.com/articles/quake-3/server-setup.php](https://www.katsbits.com/articles/quake-3/server-setup.php)
* [http://esreality.com/?a=post&id=2210880](http://esreality.com/?a=post&id=2210880)


*I know that M-M-M-MMULTIKILL/KILLING-SPREE is from Unreal Tournament. .. Hold your horses.
