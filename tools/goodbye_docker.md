---
title: Goodbye, Docker
parent: tools
tags: [open-source, containers, cloud-native]
categories: [containers, cloud-native, open-source]
thumbnail: assets/containers.jpg
date: 2021-11-01 11:14
description: One way to move from Docker to Podman on MacOS.
toc: true
author: tim-clegg
---
{% img aligncenter assets/containers.jpg 400 266 "Containers" "Containers" %}
*(Photo by [Tom Fisk](https://www.pexels.com/@tomfisk?utm_content=attributionCopyText&utm_medium=referral&utm_source=pexels) from [Pexels](https://www.pexels.com/photo/aerial-photography-of-container-van-lot-3063470/?utm_content=attributionCopyText&utm_medium=referral&utm_source=pexels))*

> This article expresses the views and opinions of the author only, and does not represent Oracle or anyone else.
{:alert}

## Hello containers, goodbye Docker?
It's a sad day.  For years I've used Docker Desktop both personally and professionally.  While not perfect, it's been a great tool for building and running containers.  Docker Compose was there as a "poor man's orchestration solution", allowing me to cobble together minimalistic environments used for testing and experimentation.  Those were the good old days.

Apparently I've had my head in the sand.  Well, not really - it's a byproduct of being extremely busy and very laser-focused on specific objectives.  It happens to us all.  Regardless, yesterday I was alerted by Docker Desktop that there were changes made to the license.  At first I wasn't too concerned, after all, so many know and love Docker - there's no way they'd alienate their user base, right?!  Wrong.

In case you too have been in the dark, check out the [Docker FAQs](https://www.docker.com/pricing/faq) for info on how and why they've changed their licensing.  Suffice it to say that I don't appreciate bait-and-switch tactics from *any* vendor, so I am saying "goodbye" to Docker.  It's been a good run.  A long run.  But it's time we part ways.

## Making the separation
It's hard to get rid of a tool that I've loved so much over the years.  Uninstalling it on MacOS was easy, following the [official uninstallation directions](https://docs.docker.com/desktop/mac/install/#uninstall-docker-desktop).  Following the directions, I patiently waited.  Maybe I had a lot of images... it took a short eternity (at least felt like it).  Had to tell uninstall it twice (the first time it didn't complete), then thankfully there were no issues in uninstalling it.

## The next tool
As part of any migration, you must choose your poison (the next tool, platform, etc.).  I decided to go with [Podman](https://podman.io), as it seemed to be a drop-in replacement for Docker.  This makes my life easier, in that I don't have to learn a new tool or CLI, but rather stick with my traditional Docker CLI commands.

Podman was easily installed with `brew install podman`.  This was what the [Podman installation instructions](https://podman.io/getting-started/installation) told me to do.  Easy enough!

The last thing I did was to create an alias, to make my life easier (hard to break old habits):

```
alias docker=podman
```

In actuality, I added this to my `~/.zshrc` file (so I didn't have to run this every time I opened a shell).  If you use bash, this might be added to your `~/.bash_profile` file.

I proceeded with the [installation instructions](https://podman.io/getting-started/installation):

```
podman machine init
podman machine start
```

At which point I was ready to get working with containers, using Podman!

## The beginning of the end
Happily going down my merry way, knowing that I was no longer dependent upon "evil docker" (I'm exagerating horribly and being overly dramatic here), I went to run a container that involved mounting a location.  Most of the containers I work with involve some sort of mounting/sharing of data between the host and the container.  Something I've taken for granted, apparently.  I hit a brick wall.

Podman, on MacOS, using qemu, does not, out of the box, support mounting local file systems to the container.  This is a by-product of how it's working, really.  At first I was incredulous and a bit upset by this inconvenient detour.  But then, the silver lining appeared: let's not use qemu, but instead take a different approach... using VirtualBox.

## The next part of the saga
First Vagrant needed to be installed:

```
brew install vagrant
```

Took the config from [this blog](https://www.redhat.com/sysadmin/replace-docker-podman-macos) and modified it a bit, to use OL8.  Here's the `Vagrantfile` I used (placed this in `~/tools/podman/Vagrantfile` as suggested in [Dave's article](https://www.redhat.com/sysadmin/replace-docker-podman-macos)):

```
Vagrant.configure("2") do |config|
  config.vm.box = "generic/oracle8"
  config.vm.box_url = "https://oracle.github.io/vagrant-projects/boxes/oraclelinux/8.json"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
  end

  config.vm.provision "shell", inline: <<-SHELL
    yum install -y podman

    groupadd -f -r podman

    #systemctl edit podman.socket
    mkdir -p /etc/systemd/system/podman.socket.d
    cat >/etc/systemd/system/podman.socket.d/override.conf <<EOF
[Socket]
SocketMode=0660
SocketUser=root
SocketGroup=podman
EOF
    systemctl daemon-reload
    echo "d /run/podman 0770 root podman" > /etc/tmpfiles.d/podman.conf
    sudo systemd-tmpfiles --create

    systemctl enable podman.socket
    systemctl start podman.socket

    usermod -aG podman $SUDO_USER
  SHELL
  
  config.vm.synced_folder "/Users", "/Users"
end
```

This is great, but it is missing the mount(s), which we really need/want if we want to share files between the host and containers.  See [this blog](https://www.danielstechblog.io/running-podman-on-macos-with-multipass/) for what one person did, using [multipass](https://multipass.run).  Since we're using Vagrant, VirtualBox and Oracle Linux, we need to do this a bit differently.  [Vagrant sync'd folders](https://www.vagrantup.com/docs/synced-folders/basic_usage) is here to save the day!  Make sure to modify as you need...

From within the `~/tools/podman` directory, I issued `vagrant up` and waited for it to download and provision.  For grins, I wanted to make sure that my mount was successful, so I connected to the newly minted VM to check it out:

```
vagrant ssh -c "ls /Users"
```

Sure enough, it was there!  All of the directories that I expected... yay.  Now let's plumb-up podman to use this VM for running containers in.

```
podman system connection add vagrant --identity ~/tools/podman/.vagrant/machines/default/virtualbox/private_key ssh://vagrant@127.0.0.1:2222/run/podman/podman.sock
podman system connection default vagrant
```

I followed steps 6 and onward in [Dave's article](https://www.redhat.com/sysadmin/replace-docker-podman-macos), yielding success to my efforts!

From there, I created the `pman` script (mentioned in this [blog](https://www.redhat.com/sysadmin/replace-docker-podman-macos)), made it executable (`chmod +x ~/tools/podman/pman`) and then added the `~/tools/podman` directory to my path (in my `~/.zshrc` file).

At this point, I can run `pman up` to get Podman running (and `pman down` to shut it down), then use `podman` (or `docker`, thanks to the alias) commands as I'm used to.

## Conclusion
Was this an exercise in futility?  For the limited functionality I needed in Docker Desktop, no.  Podman is not a one-size-fits-all "silver bullet" solution for everyone.  It doesn't do everything that Docker Desktop does (by any means), but meets my meager demands.  Your mileage may vary.

I'm able to move along my merry way, working with containers as I was previously used to.  No GUI, but I've got my trusty CLI available.  Hurrah for free, open-source software!

## Resources
* https://docs.docker.com/desktop/mac/install/#uninstall-docker-desktop
* https://podman.io/getting-started/installation
* https://devopstales.github.io/home/docker-desktop-alternatives/
* https://matt-rickard.com/docker-desktop-alternatives/
* https://medium.com/nttlabs/containerd-and-lima-39e0b64d2a59
* https://macoscontainers.org
* https://www.redhat.com/sysadmin/replace-docker-podman-macos
* https://www.danielstechblog.io/running-podman-on-macos-with-multipass/
* http://yum.oracle.com/boxes/