Upgrading Mastodon Using Docker
========

**NOTE: It is important to not update your instance to `master`. As of right now it is not guaranteed to be stable at any given moment. You can track [this issue](https://github.com/tootsuite/mastodon/issues/2046) to see if that has changed.**

This guide will cover how to update a Docker instance although some parts of this guide may be similar for non-Docker users.

1. Update tag references for ~/production with `git fetch --tags`
2. Find out what the latest tagged release is called. You can do this by going to [Mastodon's GitHub Releases](https://github.com/tootsuite/mastodon/releases) or check via command line by running the below snippet after step 1.
```
git tag | tail -n 1
```
3. Pull down the new release using `git checkout [release-number]` where `[release-number]` is what the latest release is tagged as without including the square brackets (ex. `v1.2.2`).

A more automated way to do all of this is:
```
git fetch
git checkout $(git tag | tail -n 1)
```

## Not Done Yet
Now that you have the updated changes I believe you have to run `docker-compose build`. I've been trying to stop my instance while doing this with `docker stop $(docker ps -a -q)`, but I believe I was able to do build the containers with the instance running.

Now that they're rebuilt, startup the containers with `docker-compose up -d`. You should always check the [release notes](https://github.com/tootsuite/mastodon/releases) for the tag you're upgrading to as there may be special requirements. This usually involves running two things

```
docker-compose run --rm web rails db:migrate
docker-compose run --rm web rails assets:precompile
```

The first one will upgrade your database if the schema has changed at all and the second one will compile JS, CSS, and other assets for the web. Finally, restart your instance to show these changes with

```
docker stop $(docker ps -a -q) && docker-compose up -d
```
