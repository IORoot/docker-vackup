# Vump: Backup and Restore Docker Volumes and Database Dumps.

Vump: (contraction of "volume dump")

## What

The `vump` command makes it convienient to backup, migrate and restore websites based on docker containers.

It is a command-line tool that is also the basis for the `vump_site` and `vump_menu` scripts too.

`vump_site` is a wrapper for `vump` that will backup and restore an entire website, including a MySQLDump file and website volume data, to a container registry.

The `vump_menu` script is also a wrapper to `vump` that gives you a nicer menu for all options.

## Usage

```
🚚 Volume Dump

    Docker Volume backup/recovery, import and export tool that includes MySQL databases.

    export/import copies files between a host tarball and a volume. For making volume backups and restores.

    save/load copies files from the volume into a container which then is made into a new image for commiting to registry.

    dbsave/dbload creates a Mysqldump file from the database into a container and is then made into an image for the registry.

    ┌────────┐                 ┌────────────────────────┐ 
    │        │◀──── export ────│                        │ 
    │   ./   │                 │         volume         │ 
    │        │───── import ───▶│                        │ 
    └────────┘                 └────────────────────────┘ 
                                │     │       ▲     ▲   
                                save dbsave    │     │   
                                │     │    dbload  load  
                                ▼     ▼       │     │   
                            ┌────────────────────────┐ 
                            │   busybox container    │ 
                            │      /volume-data      │ 
                            └────────────────────────┘ 
                                    │             ▲      
                                    ▼             │      
                            ┌────────────────────────┐ 
                            │  Image  of container   │ 
                            └────────────────────────┘ 

    EXPORT
    ------
    Used to copy all data inside a volume to a .tar.gz file on the host in the current directory.

    ./vump --export  --volume [volumename] --filename [filename.tar.gz]


    IMPORT
    ------
    Used to copy all data inside a .tar.gz file on the host into a specific volume.
    
    ./vump --import --filename [filename.tar.gz] --volume [volumename]


    SAVE
    ----
    Used to copy all data inside a volume onto a new container image. Used to backup to a container repository.
    Use on volumes with web-files / assets / etc... 
    
    ./vump --save --volume [volumename] --image [name_of_image] 


    LOAD
    ----
    Used to recover all data inside a backup container image onto a volume.
    
    ./vump --load --image [name_of_image] --volume [volumename] 


    DBSAVE
    ------
    Used on mysql containers. Creates a mysqldump and puts the file into a backup container image.
    Use on containers with mysql installed and has a database you wish to backup.

    ./vump --dbsave --container [container] --image [name_of_image] --database [database_name] --username [database_username] --password [database_password]

    DBFILE
    ------
    Used on mysql containers. Save the contents of the SQL file in a simple tar.gz file on the host.
    
    ./vump --dbfile --container [container] --database [database_name] --username [database_username] --password [database_password]

    DBLOAD
    ------
    Used on mysql containers. Loads the contents of the SQL file in a database backup image into the container MySQL.
    
    ./vump --dbload --image [name_of_image] --container [container] --database [database_name] --username [database_username] --password [database_password]

```

## How

### Export

```bash
# ╭──────────────────────────────────────────────────────────╮
# │                          EXPORT                          │
# ╰──────────────────────────────────────────────────────────╯
#
# 1. Create a BUSYBOX container
# 2. mount volume to    /vackup-volume
# 3. mount $pwd to      /vackup
# 4. create tar.gz of volume into $pwd
#                                ┌──────────────────────┐
# ┌────────────────┐             │ ┌────────────────┐   │
# │       ./       │────mount────┼▶│    /vackup     │   │
# └────────────────┘             │ └──────────────▲─┘   │
#                                │                │     │
#                                │ busybox       tar -c │
#                                │                │     │
# ┌────────────────┐             │ ┌──────────────┴─┐   │
# │     volume     │────mount────┼▶│ /vackup-volume │   │
# └────────────────┘             │ └────────────────┘   │
#                                └──────────────────────┘
```


### Import

```bash
# ╭──────────────────────────────────────────────────────────╮
# │                          IMPORT                          │
# ╰──────────────────────────────────────────────────────────╯
#
# 1. Create a BUSYBOX container
# 2. mount volume to    /vackup-volume
# 3. mount $pwd to      /vackup
# 4. extract contents of tar.gz into volume
#
#                                ┌──────────────────────┐
# ┌────────────────┐             │ ┌────────────────┐   │
# │       ./       │────mount────┼▶│    /vackup     │   │
# └────────────────┘             │ └──────────────┬─┘   │
#                                │                │     │
#                                │ busybox       tar -x │
#                                │                │     │
# ┌────────────────┐             │ ┌──────────────▼─┐   │
# │     volume     │────mount────┼▶│ /vackup-volume │   │
# └────────────────┘             │ └────────────────┘   │
#                                └──────────────────────┘
#
```

### Save

```bash
# ╭──────────────────────────────────────────────────────────╮
# │                           SAVE                           │
# ╰──────────────────────────────────────────────────────────╯
#
# 1. Mount volume to busybox
# 2. Copy contents to /volume-data/
# 3. Make an image of the container
# 4. Delete container
#
#                               ┌──────────────────────┐
# ┌────────────────┐            │  ┌────────────────┐  │
# │     volume     │────mount───┼─▶│ /vackup-volume │  │
# └────────────────┘            │  └──────────────┬─┘  │
#                               │                 │    │
#                               │ busybox      cp -Rp  │
#                               │                 │    │
#                               │ ┌───────────────▼┐   │
#                               │ │ /volume-data/  │   │
#                               │ └────────────────┘   │
#                               └──────────────────────┘
#                                           │           
#                                           ▼           
#                               ┌──────────────────────┐
#                               │  Image of Container  │
#                               └──────────────────────┘
#
```

### Load

```bash
# ╭──────────────────────────────────────────────────────────╮
# │                           LOAD                           │
# ╰──────────────────────────────────────────────────────────╯
#
# 1. Create container of image
# 2. Mount volume to /mount-volume
# 3. Copy everything in /volume-data to /mount-volume
#
#                               ┌──────────────────────┐
# ┌────────────────┐            │  ┌────────────────┐  │
# │     volume     │────mount───┼─▶│ /mount-volume  │  │
# └────────────────┘            │  └──────────────▲─┘  │
#                               │                 │    │
#                               │ busybox      cp -Rp  │
#                               │                 │    │
#                               │ ┌───────────────┴┐   │
#                               │ │ /volume-data/  │   │
#                               │ └────────────────┘   │
#                               └──────────────────────┘
#                                           ▲           
#                                           │           
#                               ┌──────────────────────┐
#                               │  Image of Container  │
#                               └──────────────────────┘
#
```

### DbSave

```bash

# ╭──────────────────────────────────────────────────────────╮
# │                          DBSAVE                          │
# ╰──────────────────────────────────────────────────────────╯
#
# 1. Run DB Container (with DB volume mounted)
# 2. MySQLDump database to current folder
# 3. Create new busybox container
# 4. Mount current dir to container
# 5. Copy dump file into container
# 6. Make image of container
#
#                                 ┌─────────────────────────┐
#                                 │     MySQL Container     │
# ┌──────────────────┐            │                         │
# │     DB Volume    │── mount ───┼───────▶ mysqldump       │
# └──────────────────┘            │             │           │
# ┌──────────────────┐            │ ┌───────────▼─────────┐ │
# │   ./backup.sql   │◀─── cp ────┼─│   /tmp/backup.sql   │ │
# └──────────────────┘            │ └─────────────────────┘ │
#           │                     └─────────────────────────┘
#           │                                                
#           │                     ┌─────────────────────────┐
#           │                     │         busybox         │
#           │                     │                         │
#           │                     │ ┌─────────────────────┐ │
#           └─────── mount ───────┼▶│ /vackup/backup.sql  │ │
#                                 │ └──────────┬──────────┘ │
#                                 │            │ cp -p      │
#                                 │            ▼            │
#                                 │ ┌─────────────────────┐ │
#                                 │ │ /db-data/backup.sql │ │
#                                 │ └─────────────────────┘ │
#                                 └─────────────────────────┘
#                                              │ commit      
#                                              ▼             
#                                  ┌──────────────────────┐  
#                                  │  Image of Container  │  
#                                  └──────────────────────┘  
#
```

### DbFile

```bash
# ╭──────────────────────────────────────────────────────────╮
# │                          DBFILE                          │
# ╰──────────────────────────────────────────────────────────╯
#
# 1. Run DB Container (with DB volume mounted)
# 2. MySQLDump database to current folder
#
#                                 ┌─────────────────────────┐
#                                 │     MySQL Container     │
# ┌──────────────────┐            │                         │
# │     DB Volume    │── mount ───┼───────▶ mysqldump       │
# └──────────────────┘            │             │           │
# ┌──────────────────┐            │ ┌───────────▼─────────┐ │
# │   ./backup.sql   │◀─── cp ────┼─│   /tmp/backup.sql   │ │
# └──────────────────┘            │ └─────────────────────┘ │
#                                 └─────────────────────────┘
#
```


### DbLoad

```bash
# ╭──────────────────────────────────────────────────────────╮
# │                          DBLOAD                          │
# ╰──────────────────────────────────────────────────────────╯
#
# 1. Download DB data image
# 2. Copy the backup file out of image
# 3. Copy backup file into target container
# 4. Load backup into MySQL.
#
# ┌─────────────────────┐         ┌─────────────────────────┐
# │  Image with Backup  │────────▶│        Container        │
# └─────────────────────┘         │                         │
#                                 │ ┌─────────────────────┐ │
#                                 │ │ /db-data/backup.sql │ │
#                                 │ └─────────────────────┘ │
#                                 │            │            │
#                                 │            │  cp -p     │
#                                 │            ▼            │
# ┌───────────────────┐           │ ┌─────────────────────┐ │
# │        ./         │◀── mount ─┼─│/opt/mount/backup.sql│ │
# └───────────────────┘           │ └─────────────────────┘ │
#           │                     └─────────────────────────┘
#           │                                                
#           │                                                
#           │                     ┌─────────────────────────┐
#           │                     │     MySQL Container     │
#           │                     │ ┌─────────────────────┐ │
#           └───────cp────────────┼▶│   /tmp/backup.sql   │ │
#                                 │ └─────────────────────┘ │
#                                 │            │            │
#                                 │            │ mysql      │
#                                 │            ▼            │
# ┌──────────────────┐            │ ┌─────────────────────┐ │
# │     DB Volume    │◀──mounted──┼─│      DATABASE       │ │
# └──────────────────┘            │ └─────────────────────┘ │
#                                 └─────────────────────────┘
#
```


## Build from SRC file

Within the ./src folder is the source files to the script split across multiple files. 

You can use the [Standalone](https://github.com/IORoot/standalone) tool to replace any `source ./file` lines with the actual contents of that file.

```bash

cd docker-vump/src
standalone --input vump_site.src --output  ../vump_site

```