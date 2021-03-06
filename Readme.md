# The Media Compose Stack

This is designed to be a fully dockerized Media downloading solution utilising Google Drive as an unlimited disk backend. 

This particular repo does not contain the media watching/distribution stack. Once I've cleaned that up for release, there will be a link to it here.

## Getting Started

This Readme is huge, so I made a smaller "Getting Started" page [here](docs/getting_started.md). This should be enough to get you going without having to read all of this crap.

## Motivation

There are several other attempts at this, such as [Cloudbox](https://github.com/Cloudbox/Cloudbox) and [PGBlitz](https://github.com/PGBlitz/PGBlitz.com), however I found these setups a little lacking in certain areas.

- They require an entire VM or machine dedicated to it
- They aren't _fully_ dockerized, it mangles the host machine and places config/installation all over the place, making it difficult to back up or fully utilise machine resources
- They regularly have rookie mistakes such as moving data between two Docker volumes, causing unneccessary high disk I/O
- They make too many assumptions about your current setup, and make it difficult to change (Well, so does mine, but it's __my__ setup so I'm allowed to be a hypocrite)

This setup has one gigantic shared folder named `shared`. It's set up with Rclone union mounts so that programs like Sonarr, Radarr, Medusa etc. all believe that the downloading and gdrive media directories are on the same filesystem (because they are!). This severely reduces I/O compared to moving things between volumes. There's then some clever volume mapping so all the programs have matched up directories even though they're technically pointed to different places.

## FAQ

**Q: Why are most of the containers on the host network?**

**A:** You'd be surprised how much CPU a bandwith-heavy container can use using the Docker proxy (especially for something like Sabnzbd). It just makes sense to allow the heavy stuff to bridge straight to the host, which also comes with its own set of connectivity challenges. Also, each open port would be a proxy process, so having a large range for torrenting would suck.

**Q: How do I disable some of the services I don't want/need?**

**A:** The easiest way would probably choose the services you want when you're starting the stack. Some of them have dependants, so if you choose `sonarr` you will get `rclone` by default as well, but you can customise which ones you want:
```bash
docker compose up -d sonarr radarr headphones transmission
```
Alternatively, you can also just comment out the services you don't want in the `docker-compose.yml` file. This may be a more convenient solution for some.

**Q: There are some extra settings in the service interface that you don't mention! What do I do?!**

**A:** That's intentional. The defaults for whatever I don't mention are usually fine. If I included all of the config options, this doc would be even longer than it already is.

**Q: Why is there a custom version of Traktarr/Bazarr?**

**A:** Traktarr/Bazarr times out listing movies/TV after 30/60 seconds. I edit it so it times out after 5 minutes instead. My library is big, and the Sonarr/Radarr APIs lag like crazy when they're doing _anything_.

**Q: Why Plex as opposed to Emby/Jellyfin/Serviio/Whatever?**

**A:** Jellyfin's Chromecast support is iffy at best, especially with subtitles (I watch anime on my Chromecast, deal with it). Emby has similar issues with casting and subtitles but is probably otherwise the least-worst offering. Serviio is a little feature bare. Plex, as much hacking as it requires to get to work, and as *absolutely freaking terrible* as its new interface is, does work once it's set up properly, and handles the abuse I throw at it fairly well.

**Q: Wait, you automatically switch team drives? Won't that cause duplicates?**

**A:** Nope, I fixed that in #4.

## Table of Contents
<!-- MarkdownTOC autolink="true" autoanchor="true" -->

- [Configuration and deployment](#configuration-and-deployment)
    - [Pre-Setup](#pre-setup)
    - [Rclone & Env Vars](#rclone--env-vars)
        - [Top-Level](#top-level)
            - [domain](#domain)
            - [plex_host](#plex_host)
        - [Rclone](#rclone)
            - [Multiple service credential files](#multiple-service-credential-files)
            - [Multiple Drives](#multiple-drives)
        - [Plex](#plex)
    - [Initial Starting of the Stacks](#initial-starting-of-the-stacks)
        - [Starting the Downloader Stack](#starting-the-downloader-stack)
        - [Starting the Watching Stack](#starting-the-watching-stack)
        - [Pre-Warming Rclone \(Optional\)](#pre-warming-rclone-optional)
    - [Service Configuration](#service-configuration)
        - [NZBHydra2](#nzbhydra2)
        - [Sabnzbd](#sabnzbd)
        - [Transmission](#transmission)
        - [Jackett](#jackett)
        - [Radarr](#radarr)
        - [Sonarr](#sonarr)
        - [Traktarr](#traktarr)
        - [Medusa](#medusa)
        - [Headphones](#headphones)
        - [LazyLibrarian](#lazylibrarian)
        - [Mylar](#mylar)
        - [Bazarr](#bazarr)
        - [Telegram Bots](#telegram-bots)
        - [Plex](#plex-1)
        - [Advanced Plex Modifications](#advanced-plex-modifications)
        - [Tautulli](#tautulli)
        - [Ombi](#ombi)
- [Backing up / Moving](#backing-up--moving)
- [Maintenance](#maintenance)
    - [Dedupe](#dedupe)
- [Service Port Matrix](#service-port-matrix)
- [Debugging](#debugging)
- [Todo](#todo)
    - [Services:](#services)
    - [Remote-Control:](#remote-control)
    - [Documentation:](#documentation)

<!-- /MarkdownTOC -->


<a id="configuration-and-deployment"></a>
## Configuration and deployment

<a id="pre-setup"></a>
### Pre-Setup

You will need:
- A Linux machine (I use Ubuntu 18.04 LTS)
- Docker (19.03.6+) + Docker Compose (1.17.1+)
- Some Rclone knowledge (Possibly a previous setup as well to copy files from)
- A Google Drive account with unlimited storage
- A [Google Drive service account](https://rclone.org/drive/#service-account-support)
- Encryption [set up already](https://rclone.org/crypt/) in Google Drive
- The compose file has labels compatible with my [Docker Nginx Conf Generator](https://github.com/Makeshift/docker-generate-nginx-conf), but it isn't neccessary

```bash
git clone github.com/Makeshift/Media-Compose-Stack
```

<a id="rclone--env-vars"></a>
### Rclone & Env Vars
<a id="top-level"></a>
#### Top-Level

```bash
cp .env.template .env
```

<a id="domain"></a>
##### domain
`domain=example.com`
If you intend to use my [Docker Nginx Conf Generator](https://github.com/Makeshift/docker-generate-nginx-conf), you can set your domain name here. Each container with a web UI will be given a label containing `<service>.<domain>`.

<a id="plex_host"></a>
##### plex_host
**Note:** Not yet fully implemented.
`plex_host=127.0.0.1`
What's the IP address/hostname of the machine hosting the Watching ![watch stack](./docs/images/watch.png) stack? If entered here, a script will be used to inform the Plex Rclone instance to clear the cache for newly uploaded media.

If you're running Watching ![watch stack](./docs/images/watch.png) on the same host as the Downloader ![download stack](./docs/images/download.png) stack, you can ignore this option.

<a id="rclone"></a>
#### Rclone

There are several env vars required to get Rclone to work. Here are the vars and where you can get them:

| Var                               | Source                                                                                | Notes                                                                                                                                                                      |
|--------------------------------   |-----------------------------------------------------------------------------------    |--------------------------------------------------------------------------------------------------------------------                                                        |
| rclone_encryption_password1       | [Rclone Crypt Config](https://rclone.org/crypt/)                                      | If you've previously set up Rclone, this will be in `~/.config/rclone/rclone.conf` under config param `password`                                                           |
| rclone_encryption_password2       | [Rclone Crypt Config](https://rclone.org/crypt/)                                      | If you've previously set up Rclone, this will be in `~/.config/rclone/rclone.conf` under config param `password2`                                                          |
| rclone_gdrive_mount_folder        | If your crypt directory is not the top level of Gdrive, this is the encrypted folder  | My GDrive is set up with Rclone encrypting the top level folder `encrypted`, so my mount folder is simply `encrypted`.                                                     |
| rclone_team_drive_ids             | [Rclone Gdrive Config](https://rclone.org/drive/)                                     | If you've previously set up Rclone, this will be in `~/.config/rclone/rclone.conf` under config param `team_drive`. If you want to combine multiple team drives, add them all in this variable delimited by a space. |

Remember that you *do not* need to escape the variables in env files.

This stack **only** supports team drives at the moment. I tried to support My Drive with sharing but I had trouble exceeding the 750GB/day limit with that setup.

```bash
cp rclone.env.template rclone.env
```

<a id="multiple-service-credential-files"></a>
##### Multiple service credential files

**TODO:** Provide instruction on having separate service creds for the mounts.

You can upload more than the 750GB/day limit by switching the service file that Rclone uses.

* Your files must be on a team drive. It's easy to migrate your files from "My Drive" to a team drive - just create a team drive, right click the folder and move it. It could take a little while to migrate. 80tb took about 2 days.
* I used [88lex's sa-gen](https://github.com/88lex/sa-gen) to generate my service accounts. You can then add them all to a group, and share the team drive with the group. There's a guide [here](https://github.com/88lex/sa-guide) on how to do this.
* Drop your service account json files in the `service_accounts` folder. `1.json` is used by default to mount the drive. The rest of the `.json`'s will be cycled when uploading
* The upload script will now automatically switch service accounts when the limit has been reached

<a id="multiple-drives"></a>
##### Multiple Drives

Team drives are limited to 400,000 files, meaning that if you have a large collection you'll likely need more than one drive. This is supported by the `rclone_team_drive_ids` env var in `rclone.env`.

Grab your team ids and delimit them by a space, and config will be generated on startup of the rclone container to combine them. This assumes your folder layout and encryption are identical on each team drive.

`rclone_team_drive_ids=0AHh2Szlzsxxxxxxxxx 0AHh2Szlzsyyyyyyyyy 0AHh2Szlzszzzzzzzzz`

The drives will then be cycled when `teamDriveFileLimitExceeded` is hit.

<a id="plex"></a>
#### Plex

You can optionally provide a `PLEX_CLAIM` env var in `plex.env` to speed up the process of claiming your server.

You can obtain the value [here](https://plex.tv/claim).

```bash
cp plex.env.template plex.env
```

<a id="initial-starting-of-the-stacks"></a>
### Initial Starting of the Stacks

<a id="starting-the-downloader-stack"></a>
#### ![download stack](./docs/images/download.png) Starting the Downloader Stack

The default `docker-compose.yml` file is the downloader stack. Starting the stack should be as simple as

```bash
docker-compose up -d ; docker-compose logs -f
```
I advise the above command as it will disconnect you from the containers after it finishes building them, but still allows you to follow logs to watch for errors. `Ctrl+C` will not kill the running containers, just kill the log follow.

In case of a problem, you can run
```bash
docker-compose down
```
to clean up the stack. This will not delete any configuration.

<a id="starting-the-watching-stack"></a>
#### ![watch stack](./docs/images/watch.png) Starting the Watching Stack

The watcher stack `plex-compose.yml` is used to deploy the Plex stack with its own Rclone. This is so they can be deployed on different machines and reduce the amount of competing the stacks have to do. 

However, it's still perfectly possible to run both stacks on the same machine, using the same config. I did this to do the initial scanning and prepopulating of databases on a cloud server prior to moving it to my local server.

```bash
docker-compose -f plex-compose.yml up -d
```

<a id="pre-warming-rclone-optional"></a>
#### ![download stack](./docs/images/download.png)![watch stack](./docs/images/watch.png) Pre-Warming Rclone (Optional)

It may be worth pre-warming the Rclone caches with data.
Go to `shared/merged/Media` (or whichever folder your Media lies in) and run:
```bash
ls -R . > /dev/null &
disown
```
You can continue using the remote while this happens, it will simply `ls` through every folder in your remote to warm the caches (and will disown it so it keeps going even if you kill your session).

If you really want to see some sembelance of progress of your warming, grab the `rsync` package for your distribution, go to `shared/merged/Media` (or whichever folder your Media lies in) and run:
```
rsync -rhn --info=progress2 . $(mktemp -d)
```
The progress bar is a bit fake, but if you know approximately how large your collection is, you can work out how far along you are.

<a id="service-configuration"></a>
### Service Configuration
Services with the ![download stack](./docs/images/download.png) icon are part of the 'Downloader' stack.

Services with the ![watch stack](./docs/images/watch.png) icon are part of the 'Watcher' stack.

<a id="nzbhydra2"></a>
#### ![download stack](./docs/images/download.png) NZBHydra2 
NZBHydra2 is a searching/caching/indexing tool for Newznab and Torznab indexers. It acts as a proxy in between sources of NZBs and your services, which means less configuration down the line.

You'll only need this if you plan to use this stack with Usenet.

- Connect to the Web UI on port `5076`.
- Follow the Web UI guide to add your indexers
- Click the `Config` button
- Click the `API?` button on the right hand side. Note down your API key.

<a id="sabnzbd"></a>
#### ![download stack](./docs/images/download.png) Sabnzbd 
Sabnzbd is used to download from Usenet. You'll need Usenet account(s).

- On your host, run `chmod -R 755 shared/separate/`
- Connect to the Web UI on port `8080`.
- Follow the quick-start wizard
- Click the cog at the top right

**In the `General` tab**

- Note down your API Key

**In the `Folders` tab**

| Setting Name                                       | Value                                                                                          |  
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------- |  
| Temporary Download Folder                          | `/shared/merged/downloads/sabnzbd/incomplete`                                                |  
| Minimum Free Space for Temporary Download Folder   | A reasonable number, I chose 300GB on a 2TB disk                                               |  
| Completed Download Folder                          | `/shared/merged/downloads/sabnzbd/default/` (We'll be overriding this with categories later) |  
| Permissions for completed downloads                | 777                                                                                            |  

**In the `Categories` tab**
Copy the below table:

| Category        | Priority   | Processing   | Script                                           | Folder/Path          | Indexer Categories/Groups   |  
| --------------- | ---------- | ------------ | --------                                         | -------------------- | --------------------------- |  
| Default         | Normal     | +Delete      | `sabnzbd_post_process_clear_rclone_vfs_cache.sh` |                      |                             |  
| sonarr          | Default    | Default      |                                                  | `../sonarr`          |                             |  
| headphones      | Default    | Default      |                                                  | `../headphones`      |                             |  
| radarr          | Default    | Default      |                                                  | `../radarr`          |                             |  
| lazylibrarian   | Default    | Default      |                                                  | `../lazylibrarian`   |                             |  
| medusa          | Default    | Default      |                                                  | `../medusa`          |                             |  
| mylar           | Default    | Default      |                                                  | `../mylar`           |                             |  

The script in the `Default` category is **extremely** important. It forces Rclone to update its VFS cache when a download finishes, which allows the clients (Radarr, Sonarr etc.) to actually see the completed download. If you don't have this, auto processing won't work!

<a id="transmission"></a>
#### ![download stack](./docs/images/download.png) Transmission 
Todo

<a id="jackett"></a>
#### ![download stack](./docs/images/download.png) Jackett 
Todo

<a id="radarr"></a>
#### ![download stack](./docs/images/download.png) Radarr 

- Navigate to the web UI on port `7878`
- Navigate to the `Settings` page

**In the `Media Management` tab**

- Click the 'Advanced Settings' toggle at the top right to 'Shown'.

The movie and folder formats were chosen to make importing easier in the case of losing the Radarr database. However, they are fairly optional.

| Setting Name                    | Value                                                                                                                                                    |  
| ------------------------------- | ---------------------------------                                                                                                                        |  
| **Movie Naming**                |                                                                                                                                                          |  
| Rename Movies                   | Yes                                                                                                                                                      |  
| Replace Illegal Characters      | Yes                                                                                                                                                      |  
| Colon Replacement Format        | Replace with Space Dash Space                                                                                                                            |  
| Standard Movie Format           | `{Movie Title} ({Release Year}) ({IMDb Id}) {{Quality Full}}`                                                                                            |  
| Movie Folder Format             | `{Movie TitleThe} ({Release Year}) ({IMDb Id})`                                                                                                          |  
| **Folders**                     |                                                                                                                                                          |  
| Automatically Rename Folders    | (Optional - May be a bad idea if you have a large collection that doesn't currently follow the above folder format) Yes                                  |  
| Movie Paths Default to Static   | (Optional - If you don't tick the above, this won't do much) No                                                                                          |  
| **Importing**                   |                                                                                                                                                          |  
| Skip Free Space Check           | Yes                                                                                                                                                      |  
| Use Hardlinks instead of Copy   | No                                                                                                                                                       |  
| Import Extra Files              | Yes                                                                                                                                                      |  
| Extra File Extensions           | `srt,nfo`                                                                                                                                                |  
| **File Management**             |                                                                                                                                                          |  
| Analyse video files             | No ||

**In the `Indexers` tab**
- Press the `+` to add a new indexer
- Click `Newznab`

| Setting Name  | Value                                             |
|-------------- |------------------------------------------------   |
| Name          | NZBHydra2                                         |
| URL           | `http://localhost:5076`                           |
| API Key       | The key you noted down in the NZBHydra section    |

**In the `Download Client` tab**
- Press the `+` to add a new download client
- Click 'SABnzbd'

| Setting Name  | Value                                                 |
|-------------- |-----------------------------------------------------  |
| Name          | SABnzbd                                               |
| Host          | localhost                                             |
| Port          | 8080                                                  |
| API Key       | The API key you noted down from the SABnzbd section   |
| Category      | radarr                                                |

**Add movie library**
- Click 'Add Movies' at the top left
- Either bulk import your current library, or add a new movie to configure the default library. It should be under `/shared/merged`, eg `/sharged/merged/Media/Movies`.

<a id="sonarr"></a>
#### ![download stack](./docs/images/download.png) Sonarr 

- Navigate to the web UI on port `8989`
- Navigate to the `Settings` page

**In the `Media Management` tab**

- Click "Show Advanced" at the top

| Setting Name                    | Value                                                                                                                                                       |  
| ------------------------------- | ---------------------------------                                                                                                                           |  
| **Episode Naming**              |                                                                                                                                                             |  
| Rename Episodes                 | Yes                                                                                                                                                         |  
| Replace Illegal Characters      | Yes |
| Standard Episode Format         | `{Series TitleTheYear} - S{season:00}E{episode:00} - {Episode Title} ({Quality Full})`                                                                      |  
| Daily Episode Format            | `{Series TitleTheYear} - {Air-Date} - {Episode Title} ({Quality Full})`                                                                                     |  
| Anime Episode Format            | `{Series TitleTheYear} - S{season:00}E{episode:00} - {Episode Title} ({Quality Full})`                                                                      |  
| Series Folder Format            | `{Series TitleTheYear} ({ImdbId})`                                                                                                                          |  
| **Folders** ||
| Delete empty folders | Yes |
| **Importing**                   |                                                                                                                                                             |  
| Skip Free Space Check           | Yes                                                                                                                                                         |  
| Use Hardlinks instead of Copy   | No                                                                                                                                                          |  
| Import Extra Files              | Yes                                                                                                                                                         |  
| Extra File Extensions           | `srt,nfo`                                                                                                                                                   |  
| **File Management**             |                                                                                                                                                             |  
| Analyse video files             | No |
| **Root Folders**                | ~~`/shared/merged/Media/TV`~~ **Warning:** With the current version of Sonarr V3 I've found setting this will cause Sonarr to hang in D state indefinitely. |  


**In the `Indexers` tab**
- Press the `+` to add a new indexer
- Click `Newznab`

| Setting Name  | Value                                             |
|-------------- |------------------------------------------------   |
| Name          | NZBHydra2                                         |
| URL           | `http://localhost:5076`                           |
| API Key       | The key you noted down in the NZBHydra section    |

**In the `Download Client` tab**
- Press the `+` to add a new download client
- Click 'SABnzbd'

| Setting Name  | Value                                                 |
|-------------- |-----------------------------------------------------  |
| Name          | SABnzbd                                               |
| Host          | localhost                                             |
| Port          | 8080                                                  |
| API Key       | The API key you noted down from the SABnzbd section   |
| Category      | sonarr                                                |

- In the settings for download clients

| Setting Name                    | Value                                                 |  
| --------------                  | ----------------------------------------------------- |  
| **Completed Download Handling** |                                                       |  
| Remove                          | Yes                                                   |  
| **Failed Download Handling**    |                                                       |  
| Remove                          | Yes                                                   |  

**In the `Series` Menu**
If you have existing series, click 'Import'. If not, click 'Add New' and follow the instructions for adding the root directory `/shared/merged/Media/TV`.

<a id="traktarr"></a>
#### ![download stack](./docs/images/download.png) Traktarr 
Traktarr can automatically add new TV series and movies to Sonarr & Radarr based on Trakt lists.

- First, copy the file `traktarr.json.template` to `traktarr.json`. 
 
**Note:** My Traktarr settings are probably *very* extreme for most people. You will likely need to edit these.

- Update the following fields in the JSON object:

| Setting Key   | Value                                                 |
|-------------- |-----------------------------------------------------  |
| radarr.api_key          |     Found in Radarr Settings -> General -> Security -> API Key |
| sonarr.api_key | Found in Sonarr Settings -> General -> Security -> API Key |
| omdb.api_key | You can generate a key [here](http://www.omdbapi.com/apikey.aspx) |
| trakt.client_id <br> trakt.client_secret | You can generate these [here](https://trakt.tv/oauth/applications/new) |

- You will need to restart the container for it to pick up the changes: `docker-compose restart traktarr`

<a id="medusa"></a>
#### ![download stack](./docs/images/download.png) Medusa 
Medusa is another TV series downloader, but happens to be slightly better at anime, so it's set up specifically for anime.

- Navigate to the web UI on port `8081`
- Navigate to the `Settings` page (Cog at the top right)

**In the `General` Menu**

| Setting Name          | Value                                                 |  
| --------------        | ----------------------------------------------------- |  
| **Misc**              |                                                       |  
| Show root directories | `/shared/merged/Media/Anime`                          |  

- Press 'Save Changes' at the bottom left

**In the `Search Settings` Menu**

| Setting Name                            | Value                                                 |  
| --------------                          | ----------------------------------------------------- | 
| **Episode Search**                      |                                                       |
| Forced Backlog Search day(s)            | 7300                                                  |
| Backlog Search Interval                 | 10080
| **NZB Search**                          |                                                       |  
| Search NZBs                             | True                                                  |  
| Send .nzb files to                      | SABnzbd                                               |  
| SABnzbd server URL                      | `localhost:8080`                                      |  
| Sabnzbd API key                         | The API key you saved from the SABnzbd section        |  
| Use SABnzbd category                    | medusa                                                |  
| Use sabnzbd category (backlog episodes) | medusa                                                |  

_In the `Torrent Search` tab_

| Setting Name       | Value                                                 |  
| --------------     | ----------------------------------------------------- |  
| **Torrent Search** |                                                       |  
| Search Torrents    | False (For now)                                       |  
- Press 'Save Changes' at the bottom left

**In the `Search Providers` Menu**

_In the `Configure Custom Newsnab Providers` tab_

| Setting Name                           | Value                                                 |  
| --------------                         | ----------------------------------------------------- |  
| **Configure Custom Newznab Providers** |                                                       |  
| Select Provider                        | -- add new provider --                                |  
| Provider Name                          | NZBHydra2                                             |  
| Site URL                               | `http://localhost:5076`                               |  
| API Key                                | The key you noted down in the NZBHydra section        |  

- Ctrl+Click on all of the Newznab search categories, then click the 'Select Categories' button.
- Press the 'Add' button.
- Press 'Save Changes' at the bottom left

_In the `Provider Options` tab_
 
 | Setting Name                           | Value                                                 |  
| --------------                         | ----------------------------------------------------- |  
| Configure Provider | NZBHydra2                                                       |  
| Enable daily searches                        | Ticked               |  
| Enable for 'Manual search' feature                          | Ticked                                             |  
| Enable backlog searches                               | Ticked                               |  
| Backlog search mode                                | season packs only        |  
| Enable fallback | Ticked |

_In the `Provider Priorities` tab_

- Ensure that the NZBHydra2 provider is ticked and dragged to the top.
- Press 'Save Changes' at the bottom left

**In the `Subtitles Settings` Menu**

_In the `Subtitles Search` tab_

| Setting Name         | Value                                                 |  
| --------------       | ----------------------------------------------------- |  
| **Subtitles Search** |                                                       |  
| Search Subtitles     | Yes                                                   |  
| Subtitle Languages   | English (Or something else, if you like)              |  

_In the `Subtitles Plugin` tab_

- You will need to configure these to your liking. Alternatively, you can let Bazarr do all the subtitle work for anime as well.

** In the `Post Processing` Menu **

_In the `Post Processing` tab_

| Setting Name                  | Value                                                 |  
| --------------                | ----------------------------------------------------- |  
| **Scheduled Post-Processing** |                                                       |  
| Scheduled Postprocessor       | Yes                                                   |  
| Post Processing Dir           | `/shared/merged/downloads/sabnzbd/medusa`             |  
| Processing Method             | Move                                                  |  

_In the `Episode Naming` tab_

| Setting Name       | Value                                                 |  
| --------------     | ----------------------------------------------------- |  
| **Episode Naming** |                                                       |  
| Name Pattern       | `Season %0S/%SN - S%0SE%0E - %EN (%QN)`               |  

- Press 'Save Changes' at the bottom left

**In the `Anime` Menu**

_In the `AnimeDB Settings` tab_

| Setting Name   | Value                                                 |  
| -------------- | ----------------------------------------------------- |  
| **AniDB**      |                                                       |  
| Enable         | Yes                                                   |  
| AniDB Username | Your AniDB Username                                   |  
| AniDB Password | Your AniDB Password                                   |  

- Press 'Save Changes' at the bottom left


** Under the `Shows` menu, click `Add Shows` **

- If you have existing shows, click 'Add Existing Shows' and follow the prompts.
- My recommended settings for the `Customize Options` tab are as follows:

| Setting Name                        | Value                                                 |  
| --------------                      | ----------------------------------------------------- |  
| Quality                             | Any                                                   |  
| Subtitles                           | Yes                                                   |  
| Status for previously aied episodes | Wanted                                                |  
| Status for all future episodes      | Wanted                                                |  
| Season Folders                      | Yes                                                   |  
| Anime                               | Yes                                                   |  

<a id="headphones"></a>
#### ![download stack](./docs/images/download.png) Headphones 
Headphones is an automatic music downloader.

- Navigate to the web UI on port `8181`
- Navigate to the `Settings` page (Cog at the top right)

**In the `Download Settings` Tab**

| Setting Name          | Value                                                 |  
| --------------        | ----------------------------------------------------- |  
| **Usenet**              |                                                       |  
| SABnzbd Host                      | `localhost:8080`                                      |  
| Sabnzbd API key                         | The API key you saved from the SABnzbd section        |  
| SABnzbd category                    | headphones                                                 |  
| Music Download Directory | `/shared/merged/downloads/sabnzbd/headphones` |

- Click 'Save Changes' at the bottom left

**In the `Search Providers` tab**

| Setting Name             | Value                                                 |  
| --------------           | ----------------------------------------------------- |  
| **NZBs**                 |                                                       |  
| Custom Newznab Providers | Ticked                                                |  
| Newznab Host             | `http://localhost:5076`                               |  
| Newznab API              | The key you noted down in the NZBHydra section        |  

**In the `Quality & Post Processing` tab**

| Setting Name                         | Value                                                 |  
| --------------                       | ----------------------------------------------------- |  
| **Quality**                          |                                                       |  
| Highest Quality including lossless   | Ticked (Optional)                                     |  
| **Post-Processing**                  |                                                       |  
| Move Downloads to Destination Folder | Ticked                                                |  
| Replace existing folders?            | Ticked (Optional)                                     |  
| Keep original folder (i.e copy)      | Ticked (Optional)                                     |  
| Rename Files                         | Ticked (Optional)                                     |  
| Correct metadata                     | Ticked (Optional)                                     |  
| Delete leftover files                | Ticked (Optional)                                     |  
| Keep original nfo                    | Ticked (Optional)                                     |  
| Embed lyrics                         | Ticked (Optional)                                     |  
| Embed album art in each file         | Ticked (Optional)                                     |  
| Add album art jpeg to album folder   | Ticked (Optional)                                     |  
| Destination Directory                | `/shared/merged/Media/Music`                          |  

- Click 'Save Changes' at the bottom left

**In the `Advanced Settings` tab**

| Setting Name                                       | Value                                                 |  
| --------------                                     | ----------------------------------------------------- |  
| **Renaming Options**                               |                                                       |  
| Folder Format                                      | `$Artist - $Album ($Year)`                            |  
| File Format                                        | `$Track - $Title`                                     |  
| **Miscellaneous**                                  |                                                       |  
| Automatically include extras when adding an artist | Single, Ep, Compilation                               |  
| Only include 'official' extras                     | Ticked                                                |  
| Automatically mark all albums as wanted            | Ticked                                                |  

- Click 'Save Changes' at the bottom left

**In the `Manage` Menu (At the top)**
- You can scan your current collection here.

<a id="lazylibrarian"></a>
#### ![download stack](./docs/images/download.png) LazyLibrarian 
LazyLibrarian is an automatic ebook downloader.

- Navigate to the web UI on port `5299`
- Navigate to the `Config` page

**In the `Downloaders` tab**

| Setting Name         | Value                                                 |  
| --------------       | ----------------------------------------------------- |  
| **Usenet**           |                                                       |  
| Use Sabnzbd+         | Ticked                                                |  
| SABnzbd Host         | `localhost`                                           |  
| SABnzbd Port         | `8080`                                                |  
| Sabnzbd API key      | The API key you saved from the SABnzbd section        |  
| Use SABnzbd category | lazylibrarian                                         |  

**In the `Providers` tab**

| Setting Name          | Value                                                 |  
| --------------        | ----------------------------------------------------- |  
| **Newznab Providers** |                                                       |  
| Name                  | NZBHydra2                                             |  
| Newznab URL #0        | `http://localhost:5076`                               |  
| Newznab API #0        | The key you noted down in the NZBHydra section        |  

- Make sure you tick the tiny tickbox at the top right of the NZBHydra2 box to enable it.

**In the `Processing` tab**

| Setting Name                         | Value                                                 |  
| --------------                       | ----------------------------------------------------- |  
| **Status**                           |                                                       |  
| Missing Book Status                  | Wanted                                                |  
| New Book Status                      | Wanted                                                |  
| New AudioBook Status                 | Skipped (Optional)                                    |  
| New Authors eBook Status             | Skipped                                               |  
| New Authors AudioBook Status         | Skipped                                               |  
| New Series Status                    | Wanted                                                |  
| **Folders**                          |                                                       |  
| Download Directories                 | `/shared/merged/downloads/sabnzbd/lazylibrarian`    |  
| eBook Library Folder                 | `/shared/merged/Media/Books/`                         |  
| AudioBook Library Folder             | `/shared/merged/Media/Audiobooks`                     |  
| **Miscellaneous**                    |                                                       |  
| Rename existing books on libraryscan | Ticked                                                |  

- Click 'Save Changes' at the bottom left

<a id="mylar"></a>
#### ![download stack](./docs/images/download.png) Mylar 
Mylar is an automatic comic book downloader.

- Navigate to the web UI on port `8090`
- Navigate to the `Config` page (Cog at the top right)

**In the `Web Interface` tab**

- Note that Mylar requires a ComicVine account and [API key](https://comicvine.gamespot.com/api/) for searching to work.

| Setting Name        | Value                                                       |  
| --------------      | -----------------------------------------------------       |  
| **API**             |                                                             |  
| ComicVine API Key   | Can be obtained [here](https://comicvine.gamespot.com/api/) |  
| **Comic Location**  |                                                             |  
| Comic Location Path | `/shared/merged/Media/Comics`                               |  

**In the `Download settings` tab**

| Setting Name                           | Value                                                 |  
| --------------                         | ----------------------------------------------------- |  
| **Usenet**                             |                                                       |  
| Sabnzbd                                | Ticked                                                |  
| SABnzbd Host                           | `localhost:8080`                                      |  
| Sabnzbd API key                        | The API key you saved from the SABnzbd section        |  
| SABnzbd category                       | mylar                                                 |  
| Are Mylar/Sabnzbd on separate machines | Ticked                                                |  
| SABnzbd Download Directory             | `/shared/separate/downloads/sabnzbd/mylar`            |  
| Enable Completed Download Handling     | Ticked                                                |  
 



**In the `Search providers` tab**

| Setting Name   | Value                                                 |  
| -------------- | ----------------------------------------------------- |  
| **Newznab**    |                                                       |  
| Use Newznab    | Ticked (Click 'Add Newznab')                          |  
| Newznab Name   | NZBHydra2                                             |  
| Newznab Host   | `http://localhost:5076`                               |  
| Newznab API    | The key you noted down in the NZBHydra section        |  

**In the `Quality & Post Processing` tab**

| Setting Name                                  | Value                                                 |  
| --------------                                | ----------------------------------------------------- |  
| **Post-Processing**                           |                                                       |  
| Enable Post-Processing                        | Ticked                                                |  
| **Metadata Tagging**                          |                                                       |  
| Enable Metadata Tagging                       | Ticked                                                |  
| Write ComicRack (cr) tags (ComicInfo.xml)     | Ticked                                                |  
| Write ComicBookLover (Cbl) tags (zip comment) | Ticked                                                |  
| Overwrite existing cbz tags (if they exist)   | Ticked                                                |  
| **Failed Download Handling**                  |                                                       |  
| Enabled Failed Download Handling              | Ticked                                                |  
| Enable Automatic-Retry for Failed Downloads   | Ticked                                                |  

**In the `Advanced Settings` tab**

| Setting Name                                        | Value                                                 |  
| --------------                                      | ----------------------------------------------------- |  
| **Miscellaneous**                                   |                                                       |  
| Automatically Mark All Issues as Wanted             | Ticked                                                |  
| Place cover.jpg into Comic Directory for each comic | Ticked                                                |  
| Write cvinfo into each comic directory              | Ticked                                                |  

- Click 'Save Changes' at the bottom left

<a id="bazarr"></a>
#### ![download stack](./docs/images/download.png) Bazarr 
Bazarr is an automatic subtitle downloader that is compatible with some of the other services here.

- Navigate to the web UI on port `6767`
- Follow the wizard to set up using the below settings

**In the `General` tab**

| Setting Name                   | Value                                                 |  
| --------------                 | ----------------------------------------------------- |  
| **Path Mappings For TV Shows** |                                                       |  
| **Path for Sonarr**            | **Path for Bazarr**                                   |  
| `/shared/merged/Media/TV`      | `/shared/merged/Media/TV`                             |  
| **Path Mappings for Movies**   |                                                       |  
| **Path for Radarr**            | **Path for Bazarr**                                   |  
| `/shared/merged/Media/Movies`  | `/shared/merged/Media/Movies`                         |  

- Click 'Next'

**In the `Subtitles` tab**

| Setting Name                | Value                                                 |  
| --------------              | ----------------------------------------------------- |  
| **Subtitles Providers**     | This section will have to be configured by you.       |  
| **Subtitles Languages**     |                                                       |  
| Enabled Languages           | Optional                                              |  
| **Series default settings** |                                                       |  
| Default Enabled             | Yes                                                   |  
| Languages                   | Optional                                              |  
| **Movie Default Settings**  |                                                       |  
| Default Enabled             | Yes                                                   |  
| Languages                   | Optional                                              |  

- Click 'Next'

**In the `Sonarr` tab**

| Setting Name            | Value                                                      |  
| --------------          | -----------------------------------------------------      |  
| **Connection Settings** |                                                            |  
| Use Sonarr              | Yes                                                        |  
| Hostname or IP Address  | localhost                                                  |  
| Listening Port          | 8989                                                       |  
| API Key                 | Found in Sonarr Settings -> General -> Security -> API Key |  

- Click 'Test'
- Click 'Next'

**In the `Radarr` tab**

| Setting Name            | Value                                                      |  
| --------------          | -----------------------------------------------------      |  
| **Connection Settings** |                                                            |  
| Use Radarr              | Yes                                                        |  
| Hostname or IP Address  | localhost                                                  |  
| Listening Port          | 7878                                                       |  
| API Key                 | Found in Radarr Settings -> General -> Security -> API Key |  

- Click 'Test'
- Click 'Save'
- Click the 'Here' restart prompt

<a id="telegram-bots"></a>
#### ![download stack](./docs/images/download.png) Telegram Bots 
This stack will eventually include my [Telegram bot](https://github.com/Makeshift/telegram-sonarr-radarr-bot), that will let you add new wanted items to any of the above services. However, at the moment development for it is paused. I do intend to continue it at some point.

Todo

<a id="plex-1"></a>
#### ![watch stack](./docs/images/watch.png) Plex 

- Navigate to the web UI on port `32400`

<a id="advanced-plex-modifications"></a>
#### ![watch stack](./docs/images/watch.png) Advanced Plex Modifications 

You can do horrendous things to the Plex database to get it to act JUST how you like it. I'll be going through some of those here.

Todo

<a id="tautulli"></a>
#### ![watch stack](./docs/images/watch.png) Tautulli 

Todo

<a id="ombi"></a>
#### ![watch stack](./docs/images/watch.png) Ombi 

Todo


<a id="backing-up--moving"></a>
## Backing up / Moving

While it may be easier to just `tar` the entire thing to move it (make sure you `docker-compose down` first to unmount gdrive!), sometimes you only want to back up runtime config. Here's an idea of where things are stored and what you should back up.

| File                   | Description                                                                                                            | Back up?    |       
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------- | ----------  |       
| `.env`                 | Contains env vars used in the compose file itself                                                                      | Yes         |       
| `rclone.env`           | rclone required for Rclone to run                                                                                      | Yes         |       
| `rclone.conf`          | If you've made any tweaks to the mount settings, you might want to back this up                                        | Maybe       |       
| `runtime_conf/`        | Contains all the service-specific config generated after first startup and during use                                  | Yes         |       
| `shared/separate`      | Individual mounts for downloaders (You can back these up if you care about losing unsorted or in-progress downloads) and in-progress uploads   | Maybe       |       
| `shared/caches`        | Contains Rclone's disk caches                                                                       | Maybe       |
| `shared/merged`        | Union mount containing Google Drive, download and merged upload directories                                                                 | No          |       
| `shared/plex`          | Mount containing Google Drive for Plex specifically                                                                    | No          |       

<a id="maintenance"></a>
## Maintenance

There are a couple of sometimes-useful maintenance scripts provided, especially for Rclone.

<a id="dedupe"></a>
### Dedupe

There is a script that will iterate over all provided team drives and dedupe them using [Rclone's "newest" dedupe mode](https://rclone.org/commands/rclone_dedupe/). It can be ran using the following command:

```bash
docker-compose exec rclone dedupe
```

<a id="service-port-matrix"></a>
## Service Port Matrix

There's a lot of services included in these stacks. If you need to configure firewalls, here's what you need to know:

| Service           | Description        | Filename             | Stack Name                                             | Port  |  
| --------------    | ------------------ | -------------------- | ------------------------------------------------------ | ----- |  
| rclone            | Remote Control API | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 5572  |  
| nzbhydra          | Web UI             | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 5076  |  
| radarr            | Web UI             | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 7878  |  
| sonarr            | Web UI             | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 8989  |  
| sabnzbd           | Web UI             | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 8080  |  
| medusa            | Web UI             | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 8081  |  
| headphones        | Web UI             | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 8181  |  
| lazylibrarian     | Web UI             | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 5299  |  
| mylar             | Web UI             | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 8090  |  
| bazarr            | Web UI             | `docker-compose.yml` | ![download stack](./docs/images/download.png) Download | 6767  |  
| rclone            | Remote Control API | `plex-compose.yml`   | ![watch stack](./docs/images/watch.png) Watch          | 5573  |  
| plex              | Web UI             | `plex-compose.yml`   | ![watch stack](./docs/images/watch.png) Watch          | 32400 |  
| sqlite-web_radarr | Web UI             | `edit-dbs.yml`       | Debug                                                  | 8082  |  
| sqlite-web_sonarr | Web UI             | `edit-dbs.yml`       | Debug                                                  | 8083  |  

<a id="debugging"></a>
## Debugging

Some debug tools have been included, like [sqlite-web](https://github.com/coleifer/sqlite-web) which can be used to edit many of the sqlite DBs used by some of the services.

Generally, unless you know what you're doing (and are okay with voiding your support by the service authors), you shouldn't need to touch these. I generally use it when I break Sonarr in some horrific way and need to fix it manually ;).

You should **not** start these while the other services are running. You should `docker-compose down` first.

Launch all the editors with: `docker-compose -f edit-dbs.yml up -d`, or individual editors with `docker-compose -f edit-dbs.ym up -d radarr`.

The following editors are available:

| Service   | Editor        | File                                  | Port  |
|---------  |------------   |-------------------------------------  |------ |
| radarr    | sqlite-web    | `./runtime_conf/radarr/nzbdrone.db`   | 8082  |
| sonarr    | sqlite-web    | `./runtime_conf/sonarr/sonarr.db`     | 8083  |

<a id="todo"></a>
## Todo
<a id="services"></a>
### Services:

- [x] Rclone
- [x] NZBHydra2
- [x] Radarr
- [x] Sonarr
- [x] Sabnzbd
- [x] Traktarr
- [x] Medusa
- [x] Headphones
- [x] LazyLibrarian
- [x] Mylar
- [x] Bazarr
- [ ] Transmission
- [ ] Jackett
- [ ] Plex
- [ ] Tautulli
- [ ] Ombi

<a id="remote-control"></a>
### Remote-Control:

- [ ] Radarr Telegram Bot
- [ ] Sonarr Telegram Bot
- [ ] Ombi

<a id="documentation"></a>
### Documentation:

- [x] Readme
- [x] Env Setup
- [x] rclone
- [x] NZBHydra2
- [x] Radarr
- [x] Sonarr
- [x] Sabnzbd
- [x] Traktarr
- [x] Medusa
- [x] Headphones
- [x] LazyLibrarian
- [x] Mylar
- [x] Bazarr
- [ ] Radarr Telegram Bot
- [ ] Sonarr Telegram Bot
- [x] Backing Up
- [ ] Transmission
- [ ] Jackett
- [ ] Plex
- [ ] Advanced Plex
- [ ] Tautulli
- [ ] Ombi
