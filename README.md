<p align="center">
  <img src="glendalab.png" alt="Banner">
</p>


# 9lab - Random Plan9 Stuff

Collection of programs and scripts used on **9front** running on **Qemu**.

# Desktop

When running in my notebook screen `vgasize=1440x900x32` works great. On my external monitor `vesa=2560x1440x32` works great. To accommodate both I always boot using the smaller size and have the following **rc** script called `screen` to adjust the monitor size:

```sh
; screen big
; screen small
; cat $home/bin/rc/screen
#!/bin/rc

fn Help {
  echo `{basename $0}^' (small|big)'
}

switch ($#*) {
  case 0
    Help
  case *
    switch ($1) {
      case small
        @{rfork n; aux/realemu; aux/vga -m vesa -l 1440x900x32}
      case big
        @{rfork n; aux/realemu; aux/vga -m vesa -l 2560x1440x32}
      case *
        Help
    }
}
```

**rio** is configured via the following `riostart` script:

```sh
; cat $home/bin/rc/riostart
#!/bin/rc
# -- vgasize=1440x900x32

#-- top left:  stats
window 0,0,473,120 stats -lmisce
#-- top middle: vdir
window 483,0,956,120 vdir
#-- top right: winwatch
window 966,0,1440,120 winwatch -e '^(winwatch|stats|faces)'

#-- left: term
window 0,130,720,800
#-- right: acme
window 730,130,1440,800 acme -a -c 1

#-- bottom left: capturing console messages
window 0,810,720,890 cat /dev/kprint
#-- bottom right: face
window 730,810,1440,890 faces

# run a system shell on the serial console
~ $#console 0 || window -scroll console
```

Here is what it looks like:

<p align="center">
  <img src="screen.png" alt="Screenshot">
</p>


# Acme

Cut and Past mouse chording is broken for me on **Qemu** using **9front** *emailschaden*. To debug it I have stolen a program from **Plan9 from User Space** called `acmeevent.c` that prints all events an **acme** window receives.

Compile and install it:

```sh
; cd src/acme/cmd/acmeevent
; mk
; echo $objtype
amd64
; cp acmeevent $home/bin/$objtype
```

Open **acme** and paste the following into the **tag** of the **windows** you want to debug:

```sh
cat /mnt/acme/$winid/event | acmeevent
```

Then you should see all events that are received by the **window** (see **event** in acme(4)).

# Go

To install Go we one has to bootstrap the installation with an earlier version of Go already compiled for Plan 9. The process consists of obtaining the bootstap version and the target version, and telling the target version to use the bootstrapped version in order to build the toolchain. s

Temporary scratch space:
```sh
term% ramfs
term% cd /tmp
```

Download **bootstrap** and **target** versions of **go**:
```sh
term% hget http://www.9legacy.org/download/go/go1.14.1-plan9-amd64-bootstrap.tbz | bunzip2  -c | tar x 
term% hget https://golang.org/dl/go1.14.7.src.tar.gz | gunzip -c | tar x
```

Create **target** installation directory and **bind** downloaded version to that directory:
```sh
term% mkdir -p /sys/lib/go/amd64-1.14.7
term% bind -c go /sys/lib/go/amd64-1.14.7 
```

Setup `GOROOT_BOOTSTRAP`:
```sh
term% cd /sys/lib/go/amd64-1.14.7/src 
term% GOROOT_BOOTSTRAP=/tmp/go-plan9-amd64-bootstrap
```

To pass standard library tests configure loopback address:
```sh
term% ip/ipconfig -P loopback /dev/null 127.1 
term% ip/ipconfig -P loopback /dev/null ::1
```

Start the install (this takes a while...):
```sh
term% ./make.rc
Building Go cmd/dist using /tmp/go-plan9-amd64-bootstrap
Building Go toolchain1 using /tmp/go-plan9-amd64-bootstrap.
Building Go bootstrap cmd/go (go_bootstrap) using Go toolchain1.
Building Go toolchain2 using go_bootstrap and Go toolchain1.
Building Go toolchain3 using go_bootstrap and Go toolchain2.
Building packages and commands for plan9/amd64.
---
Installed Go for plan9/amd64 in /sys/lib/go/amd64-1.14.7
Installed commands in /sys/lib/go/amd64-1.14.7/bin
*** You need to bind /sys/lib/go/amd64-1.14.7/bin before /bin.
```

Persist installation and cleanup:
```sh
term% unmount /sys/lib/go/amd64-1.14.7 
term% dircp /tmp/go /sys/lib/go/amd64-1.14.7 
term% cp /sys/lib/go/amd64-1.14.7/bin/* /amd64/bin 
term% unmount /tmp
```

Extend `$home/lib/profile` with:
```sh
term % cat $home/lib/profile
...
GO111MODULE=on
switch($service){
case terminal
	bind -a $home/go/bin /bin
...	
```

To build go modules CA certificates are required:
```sh
term% hget https://curl.haxx.se/ca/cacert.pem >/sys/lib/tls/ca.pem 
```