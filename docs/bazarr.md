# Building Bazarr Container

The application we are going to install is Bazarr. Bazarr gets subtitles for the movies and shows Sonarr/Radarr manage.


Before installing, I need to find the image name and necessary variables required to run this container. 
    -Search for the image name
    -Pull the contents of the Image (use full image name)
    -Inspect the contents of the image

```bash
$ podman search --format "{{.Name}}\t{{.Stars}}\t{{.Official}}" bazarr

$ podman pull docker.io/linuxserver/bazarr
$ podman inspect bazarr
```
## ----------- Preset Container Variable -------------
These are defaults to set for my container
```json
    -ExposedPorts: {
        "6767/tcp": {}
}
    -Volume: {
        "/config": {}
        "/data": {}
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
Since I set the internal container ownership to my primary user, I want to make sure files created by the container — sitting on the local host — are owned by that same user (matching the PUID/PGID I set). If UserNS isnt set, any files the container creates with a permanent location on local host will be given a random number. Rootless containers do not automatically use the same numbers as their set internal PUID & PGID 
    - UserNS=keep-id

## ------------ Version Installation -----------

```bash
$ podman inspect bazarr | grep 'build_version'
```
Pinned build version tag to the image section. Applications are constantly updating and changing. In order to make sure I dont lose track of anything within our application, I set the image version to keep a constant state 

## ----------- DATA ------------
Ive chosen to keep the repository data in two separate locations

The configuration of this container can be found in [bazarrcontainer](../quadlets/bazarr.container)
The contents data will be in a separate folder under primary users home dir


# -------- DEPLOYMENT ----------
Once everything has been set within the container, my next step is to deploy successfully

I make sure all folders/files are created with the correct ownership folders/files inside receive the same permissions
```bash
$ mkdir ~/<folder_name>/bazarr
$ chmod 700 ~/<folder_name>/bazarr
```

Now I need to copy everything ive created into ~/.config/container/systemd/bazarr because my container is rootless. Want to make sure systemd is able to read container and start process.

```bash
$ cp quadlets/bazarr.container ~/.config/containers/systemd/
```

Since ive added a new daemon to my system, I need to load them in. Want to make sure, because this container is applicable to the current user who made it, the reload applies to the current user only. 
```bash
$ systemctl --user daemon-reload
```
Starting the container for my user account
```bash
$ systemctl --user start bazarr
```

# ---------- Firewall ---------------
giving port access to communicate with the internet 

```bash
$ sudo firewall-cmd --add-port=6767/tcp --permanent 
