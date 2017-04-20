Migrating Droplets
========

I decided to move my Mastodon instance to a new droplet because there were a few things not working properly and I wanted to see if that would eliminate the problem. I also wanted postgres to persist as well as redis (I think).

This workflow put my instance down for longer than a multi-user instance would probably want to go down for, but hopefully some help is better than nothing.

## Data Unique To Your Instance

- /path/to/mastodon/ _(Not the entire folder, just the listed files and folders)_
  - **.env.production** _(Changes come from you only)_
  - **docker-compose.yml** _(Changes come from you and git)_
  - **config/settings.yml** _(Changes come from you and git)_
  - **public/system** _(Changes come from users)_
- **PostgreSQL DB** _(Changes come from users)_
- **Redis `dump.rdb`** _(???)_

## On Your New Droplet (Part 1)

### Create a New Droplet

Choose a Docker 17.04.0-ce on 16.04 (or similar) image. Check the IPv6 additional option for later if you want to support that connectivity.

Once the droplet is done setting up, ssh in `ssh root@[droplet-ip]`, create a new user `adduser mastodon`, and add the user to sudoers `gpasswd -a mastodon sudo`. **NOTE: If you added an SSH key when creating the droplet, you will need to `cp /.ssh/authorized_users /home/mastodon/.ssh/` and `chown mastodon:mastodon /home/mastodon/.ssh/authorized_users`.** You should now be able to `ssh mastodon@[droplet-ip]`.

### Setup mastodon User

Make sure your new droplet is up to date by running `sudo apt-get update && sudo apt-get upgrade` and install nginx with `sudo apt-get install nginx`.

To avoid having to `sudo` every `docker` command, we're going to add our mastodon user to the docker group with `sudo usermod -aG docker $USER`. You need to type `logout` after this command for it to take effect. Log back in with `ssh mastodon@[droplet-ip]` and restart docker `sudo service docker restart`.

### Clone Mastodon Repository

At the time of writing this, `master` should not be used for a production environment. Find the [current release for the project](https://github.com/tootsuite/mastodon/releases) and then enter `git clone --branch v1.2.2 https://github.com/tootsuite/mastodon production` where `v1.2.2` is the tagged release you find.

### Transfer .env.production

I used SSH keys from the start of my droplets and didn't feel like tinkering with them to get them talking to each other, so I transferred everything to my computer first and then to the new instance (this may not work with larger datasets).
```
scp mastodon@[old-droplet-ip]:/home/mastodon/production/.env.production ~/Downloads/
```
```
scp ~/Downloads/.env.production mastodon@[new-droplet-ip]:/home/mastodon/production/
```
This file is also small enough to where you could copy and paste between ssh terminal text editors.

### Transfer settings.yml

This can be copied over similarly to `.env.production`.
```
scp mastodon@[old-droplet-ip]:/home/mastodon/production/config/settings.yml ~/Downloads/
```
```
scp ~/Downloads/settings.yml mastodon@[new-droplet-ip]:/home/mastodon/production/config/
```
This file is also small enough to where you could copy and paste between ssh terminal text editors.

### Build Containers

My instance has 512MB of RAM. This is not enough to do what we need to do (I don't even think 1GB is enough). So we need to create a swap. **NOTE: This only creates a swap for the session which will be removed upon reboot. This [can be made permanent](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04#make-the-swap-file-permanent).**
```
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```
This means we can now run `docker-compose build` and switch to our old server.

## On Your Old Droplet

Being unclear of when image assets start populating into /production/public/system/ and also reading that data can be lost with postgres, at this point you can now shut down your Mastodon instance using `docker stop $(docker ps -a -q)`.

### Postgres DB

I didn't modify my `docker-file.yml` so the lines about persisting data were commented out. All this seems to have meant is that my database was being stored at `/var/lib/docker/volumes` and had I run `docker-compose down` the container would have been deleted or detached from my instance.

#### Backup
You can't run commands on containers while they're stopped, so I started them individually using their names. You can find the names using `docker ps -a` and then `docker start production_db_1`.

```
docker exec -t production_db_1 pg_dumpall -c -U postgres > dump_`date +%d-%m-%Y"_"%H_%M_%S`.sql
```

This should then give you a file you can `scp`. Run `docker stop production_db_1` to stop the database container.

### Redis

I could use more information on the role Redis plays in Mastodon's architecture, but redis was also not persisting.

#### Backup

Start the container using `docker start production_redis_1`.

```
docker cp production_redis_1:/data/dump.rdb ~/
```

This should then give you a file you can `scp`. Run `docker stop production_redis_1` to stop the redis container.

### Transfer /production/public/system/

Some things in this folder may seem unnecessary to back-up such as other instance's avatars, headers, and media_attachments, but I don't know how to force an instance to download them if they're missing.

```
scp -r mastodon@[old-droplet-ip]:/home/mastodon/production/public/system/ ~/Downloads
```
When the transfer completes, you can power off the old droplet using `sudo poweroff`.

## On Your New Droplet (Part 2)

Ensure you have everything on your new droplet from the list at the top of this document. This part is only going to cover restoring redis and postgres. The other files and folders are just file management.

<!-- ### Start the instance

With everything except redis and postgres in place we can run `docker-compose up -d` and get to work. -->
### Postgres DB

We have to start the container to work with it `docker start production_db_1`. Verify your container names with `docker ps -a` if need be.

#### Restore
This restore includes postgres databases, usernames, and passwords for interfacing with the database. No need to create databases or users to access them first.

```
cat dump_date.sql | docker exec -i production_db_1 psql -U postgres
```

Provided you don't get any logs that look bad, `docker stop production_db_1`.

### Redis

You might not need to do this, but `docker start production_redis_1`. Persisting redis makes a mount point inside the project folder and you can just copy your `dump.rdb` into it.

#### Restore
`cp /path/to/dump.rdb /path/to/mastodon/production/redis/` and `docker stop production_redis_1` if it was started.

### Finally

`docker-compose run --rm web rails db:migrate` and `docker-compose run --rm web rails assets:precompile` in case there were update during cloning to the new droplet.

```
docker-compose up -d
```

You still need to configure nginx again, but I'll talk more about that in [IPv6-CryptCheck-MozObservatory.md](IPv6-CryptCheck-MozObservatory.md)

## Sources
- [Deploying Mastodon on Digital Ocean](http://digitalmind.io/post/deploying-mastodon-on-digital-ocean)
- [Backup/Restore a dockerized PostgreSQL database](http://stackoverflow.com/questions/24718706/backup-restore-a-dockerized-postgresql-database/29913462#29913462)
