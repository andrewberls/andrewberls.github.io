---
layout: post
title: "Adding SFTP support to cURL"
date: "2012-10-15"
permalink: /blog/post/adding-sftp-support-to-curl
---

I was surprised today to find out that [cURL](http://curl.haxx.se/) does not have SFTP support built in. I ran into the issue trying to get set up with René Moser's awesome [git-ftp](https://github.com/resmo/git-ftp) script, which is exactly what I needed to manage deployments for one of my sites running in a restricted server environment. I'll write more about git-ftp later, but in short, it gives you all the deployment goodness and capabilities of git when you only have FTP access to a server. In particular, I need to use it with SFTP, but kept running into the error `Protocol sftp not supported or disabled in libcurl `.

As it turns out, there's no easy way to add SFTP support to cURL. In the end, I had to rebuild the library from source with some special tweaks. I  got frustrated from poring over archaic mailing list posts and jumbled documentation, so I wanted to share the steps I took to get it to work.

<break />

### What you'll need
You'll need to be comfortable doing things at the command line. You'll also need a C compiler of some sort. If you're on a Mac, the [XCode Command Line Tools](https://developer.apple.com/xcode/) package everything together nicely, otherwise I assume you have an equivalent installed. I did all of this on a Mac, but it should work on Linux as well.

### 1. Install libssh2
libssh2 is a client-side C library that implements the SSH2 protocol - we need to configure cURL to use this to get SFTP support. Download the tarball from [this page](http://www.libssh2.org/), unpack it, and `cd` into the directory. Then run:

```
./configure
make
make install
```

It will output a ton of noise to your console, but that's all you have to do here.


### 2. Install/configure/build cURL using libssh2
Here we're building cURL from source, and linking to libssh2 in the configuration. Same process as before - download the latest release from [here](http://curl.haxx.se/download.html), unpack it, and `cd` in.

```
./configure --with-libssh2=/usr/local
make
make install
```

To check that it worked, run `curl -V` and ensure that sftp appears somewhere in the Protocols list. If it doesn't, see the note below or make sure that something didn't fail unnoticed during the config/build steps (you may need to run some commands with `sudo`).

In my case, I found that the executable `/usr/bin/curl` was outdated and taking precedence over my custom build, which installs to `/usr/local/bin/`. I chose to just move the one in `/usr/bin/` to `/usr/bin/curl_backup` - if you like, you could copy the one from `/usr/local/` into `/usr/bin/`. Your call. Edit: The reason for this was that `/usr/bin` came before `/usr/local/bin` in my path. You can fix this if you like using `export PATH=/usr/local/bin:$PATH`.

I hope this can be of some use to someone! Feel free to leave a comment if you run into any trouble with any of the steps.