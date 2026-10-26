# Building Prowlarr Container

The first application we are going to install. Prowlarr is an index manager — it manages the search sources (indexers) that the other apps query.

Before installing, I need to find the image name and necessary variables required to run this container. 
    -Search for the image name
    -Pull the contents of the Image (use full image name)
    -Inspect the contents of the image

```bash
$ podman search --format "{{.Name}}\t{{.Stars}}\t{{.Official}}" prowlarr
$ podman pull docker.io/linuxserver/prowlarr
$ podman inspect prowlarr
```
## ----------- Preset Container Variable -------------
These are defaults to set for my container
```json
    -ExposedPorts: {
        "9696/tcp": {}
}
    -Volume: {
        "/config": {}
}
    -Environment:{
        **values found in permission section**
} 
```
## ---------- Permissions -------------
Setting file ownership inside the container in order to avoid any permission errors. This should also reduce the need to use 'sudo'. 
I will use these enviroment variables 
    - PUID <=== sets the containers user 
    - PGID <=== sets the containers group
    - TZ <=== container will run in this timezone

In order to find PGUID & PGID
```bash
$ id [user]
```
Since I set the internal container ownership to my primary user, I want to make sure files created by the container — sitting on the local host — are owned by that same user (matching the PUID/PGID I set). If UserNS isnt set, any files the container creates with a permenant location on local host will be given a random number. Rootless containers do not automatically use the same numbers as their set internal PUID & PGID 
    - UserNS=keep-id

## ------------ Version Installation -----------

```bash
$ podman inspect prowlarr | grep 'build_version'
```
Pinned build version tag to the image section. Applications are constantly updating and changing. In order to make sure we dont lose track of anything within our application, I set the image version to keep a constant state 

## ----------- DATA ------------
Ive chosent to keep the repository data in two seperate locations

The configuration of this container can be found in [prowlerrcontainer](/home/don/Github/ampchor-stack/quadlets/prowlerr.container)
The contents data will be in a seperate folder under primary users home dir


# -------- DEPLOYMENT ----------
Once everything has been set within the container, my next step is to deploy sucessfully

I need to make sure all folders/files are created with the correct ownership folders/files inside recieve the same permissions
```bash
$ mkdir ~/<folder_name>/Prowlarr
$ chmod 700 ~/<folder_name>/Prowlarr
```

Now I need to copy everything ive created into ~/<folder_name>/Prowlarr . This is because our container was created rootless. In order for our application to work, it needs to be in a folder our systemd knows to read so it will start up.
```bash
$ cp -r quadlets/. ~/.config/containers/systemd/

Now in order to find the container ive created for myself, I will run this command. Important to note that because this is rootless, we want to make sure the command is run for our user profile only 
```bash
$ systemctl --user daemon-reload
```
Once the container can be found, I will start the container for my user account
```bash
$ systemctl --user start prowlarr
```
