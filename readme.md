# SteamCMD downloading + automation for Synology NAS
10/07/20

Dev

# Basics

## Why?

If you want to download large game files in the background, on your NAS. And if you like playing messing around. I couldn't find much info on this recently, so here's a quick guide to the essentials.

## Overview

For this, you can use [SteamCMD, Steam's command-line client](https://developer.valvesoftware.com/wiki/SteamCMD). At a basic level, just use it manually. For more advanced usage, automate downloads from your Steam library. For bonus points, run a cache too.


## Set-up

1. Install [Docker SteamCMD container (I used the **cm2network/steamcmd** one, others are available)](https://registry.hub.docker.com/r/cm2network/steamcmd) (direct SteamCMD installation on Synology DSM should also be possible - not yet tested). Use the Docker DSM GUI (Image > Add, from URL or downloaded file), or shell commands, to install.

2. Mount external storage for the container. In the Docker DSM GUI this is under Container > Edit > Mount settings when container is NOT running. This is where you'll put the files on your NAS, e.g. the share `/docker/steam`. Without this everything will stay in the container (and may not be persistent with container restart), defaulting to the path `/home/steam/Steam/steamapps`.

## Basic use

Via Docker DSM GUI

1. Run Docker container.
2. Navigate to terminal tab.
3. Use this to run SteamCMD commands.

Via terminal

1. Login to your NAS as usual with SSH, `ssh user@NAS`
2. Connect to the container `sudo docker exec -it cm2network-steamcmd1 /bin/bash`
3. Use this to run SteamCMD commands.

For a basic SteamCMD test, and to establish Steam account connection, just run `./steamcmd.sh +login <username>`

At the first go this will require a password, and two-factor authorisation (e.g. a Steam Guard code in your email). After that it should only need the username.

## Getting game info

As a quick test, try getting some game info from an appID.

`./steamcmd.sh +app_info_print 70 +quit`

TODO - it'd be good to parse this info, esp. for download size.



# Downloads

This can be [done interactively at the `STEAM>` prompt](https://developer.valvesoftware.com/wiki/SteamCMD#Downloading_an_app) after running the `steamcmd.sh` script. The necessary commands can also be passed directly to the script... At the most basic level this just requires a command line of the form:

```
<steamCMD>/steamcmd.sh +login <user> +force_install_dir <gameDIR> +@sSteamCmdForcePlatformType <OS> +app_update <appID> validate quit
```

This requires a few things:

* <steamCMD> is the SteamCMD root directory, e.g. `~/steamcmd` for the Docker container used here.
* <user> is the account name. (This assumes you've already logged in once, as above).
* <gameDIR> is where the download will be stored, probably of the form `<mount>/<game>`, e.g. `/downloads/Half-Life` if you mounted your container's storage to `/downloads`. See more notes on this below!
* <OS> is the operating system for which you want the game: [windows | macos | linux]
* <appID> is the ID of the game you want to download, you can find this from the Steam Store page link, or [via SteamDB](https://steamdb.info).

For a full list of command line options, see [the Valve wiki](https://developer.valvesoftware.com/wiki/Command_Line_Options#SteamCMD).

To set download throttling, add a command: `+set_download_throttle <kbps>`. Note the units here! For me, on a typical countryside 5Mbit ADSL connection, something like 2500 kbps works well here, and shows average usage ~300 KB/s on my router's traffic monitor, with variation from ~200-400 KB/s (i.e. uses ~40-80% of my bandwidth, and leaves overhead to do other things!). Yes, my router has traffic management/shaping as an alternative to setting throttling here; no, it doesn't work all that well. Yes, it takes a long time to download large games... that's why I'm using my NAS!

To check the progress of your download you can do one/all of the following:

* Look at the Log or Terminal tabs in the DSM Docker window. This shows the messages output by SteamCMD.
* Run the usual Steam GUI on another machine - if the **cm2network/steamcmd** host is picked up (you'll see the usual Steam message at the bottom right of your screen regarding available hosts), then the download progress should also show up in your game library list in the usual way (this was a little intermittent for me however).
* Check the file size in the target directory.

Note: in quick testing it's not clear if SteamCMD picks up partially downloaded content if interrupted. One would hope so, but it may not - TBC.


# Options & additions

A few more sophisticated shell scripts for automating some of the process.

## SteamCMD-wrapper
Shell script for downloads (and some other things), from mdeguzis/ProfessorKaos64 including correcting install dir: https://github.com/mdeguzis/steamcmd-wrapper

- Basic game info function (no JSON parsing)

- Install dir name routine. This grabs the correct name from SteamCMD game info, as so:
`./steamcmd.sh +app_info_print ${GAME_APP_ID} +quit | awk -F'"' '/installdir/ {print $4}'`

There's a quick-hack -syno version of this script in this repo, with the paths changed slightly for use with the Docker container.

To run:

* Set options (paths & download speed) at file head as required.
* Copy to wherever it can be access on your NAS, e.g. the Docker shared storage location.
* Modify permissions if required, `chmod a+x steamcmd-wrapper-syno.sh` will usually do it.
* Run from within your Docker container.
  * Game info: `steamcmd-wrapper-syno.sh -i <appID>`
  * Download a game: `steamcmd-wrapper-syno.sh -g <appID> -p <platform>`




## SteamCMDHelper
Another shell script for wrapping game download functionality, includes some notes on the game naming and directory structure, ahem, issue. This one doesn't have many of the bells and whistles of SteamCMD-wrapper, but can handle batch downloading with a list of AppIDs.
https://github.com/tunbridgep/SteamCMDHelper

## Generate install list for your game library

There's a script [for this from amildahl](https://github.com/amildahl/steamcmd_scripts). This uses SteamDB to grab a list of your games via http://steamdb.info/calculator/?player=<steamID>, which can then be used to automate downloads. This requires your games to be publically shared (i.e. they appear on SteamDB), BUT... it doesn't seem to work for me.

## Full Steam AppID and Command lists

See dgibbs64's repos [SteamCMD-AppID-List](https://github.com/dgibbs64/SteamCMD-AppID-List) and [SteamCMD-Commands-List](https://github.com/dgibbs64/SteamCMD-Commands-List). These contain the lists, and scripts to gereate the lists. If one were so inclined, the AppID list could be used for local game ID searches.

## Run updates on installed games

There's a short script for this at https://github.com/JuanMadness/VaporScript, which basically parses your `steamapps` folder for the AppIDs of installed games.

# Advanced

## Set up a cache server

A very cool thing to do, if one is so inclined, and quite easy with Docker. There's a good [Ars article on this](https://arstechnica.com/gaming/2017/01/building-a-local-steam-caching-server-to-ease-the-bandwidth-blues/), and it looks like the [Monolithic container](http://lancache.net/docs/containers/monolithic/) is probably the thing to try.

## Use python

I've stuck to shell scripts above, since they're pretty robust and work in the minimal Docker SteamCMD container. But it'd be nice to use python, since it's lovely. There's a [basic wrapper at pysteamcmd](https://github.com/f0rkz/pysteamcmd), which should be a good place to start.
