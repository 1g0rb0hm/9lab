<p align="center">
  <img src="glendalab.png" alt="Banner">
</p>

Collection of programs, scripts, and documentation about **9front**.

# Desktop

When running in my notebook screen `vgasize=1440x900x32` works great. On the external monitor `vesa=2560x1440x32` works great (YMMV). To accommodate both I always boot using the smaller size and have the following **rc** script called `screen` to adjust the monitor size:

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

**rio** is configured via the following `riostart` script to have a 2-column 3-rows layout with **acme** across the middle row:

```sh
; cat $home/bin/rc/riostart
#!/bin/rc

# -- VARS
# Dim..screen dimension (W x H)
Dim=(`{echo $vgasize | awk -Fx '{printf("%d %d", $1, $2)}'})
W=`{echo $Dim | awk '{print $1}'}
H=`{echo $Dim | awk '{print $2}'}
# s..space from border
s=5
# w..window width ratio
w=`{echo $W | awk '{print $1/2}'}
# h..window height ratio
h=`{echo $H | awk '{print $1/10}'}
# x..list of x point pairs (2 columns -> 2 pairs)
x=(`{echo $w $s | awk '{printf("%d %d %d %d",
                               (0*$1)+$2 , (1*$1)-$2,
                               (1*$1)+$2 , (2*$1)-$2)}'})
# -- y..list of y point pairs (3 rows -> 3 pairs)
y=(`{echo $h $s | awk '{printf("%d %d %d %d %d %d",
                               (0  *$1)+$2 , (1.5*$1)-$2,
                               (1.5*$1)+$2 , (9  *$1)-$2,
                               (9  *$1)+$2 , (10 *$1)-$2)}'})
# -- FUNS
fn TopRow{
	ldim=`{echo $x(1),$y(1),$x(2),$y(2) | sed 's/[ ]+//g'}
	rdim=`{echo $x(3),$y(1),$x(4),$y(2) | sed 's/[ ]+//g'}
	window $ldim stats -lmisce
	window $rdim winwatch -e '^(winwatch|stats|faces|vdir)'
}
fn MidRow{
	dim=`{echo $x(1),$y(3),$x(4),$y(4) | sed 's/[ ]+//g'}
	ldim=`{echo $x(1),$y(3),$x(2),$y(4) | sed 's/[ ]+//g'}
	rdim=`{echo $x(3),$y(3),$x(4),$y(4) | sed 's/[ ]+//g'}
	window $dim  acme -c2 -a
	window $ldim 
	window $rdim
}
fn BotRow{
	ldim=`{echo $x(1),$y(5),$x(2),$y(6) | sed 's/[ ]+//g'}
	rdim=`{echo $x(3),$y(5),$x(4),$y(6) | sed 's/[ ]+//g'}
	window $ldim vdir
	window $rdim faces
}
# -- MAIN
TopRow
MidRow
BotRow
```

Here is what it looks like:

<p align="center">
  <img src="screen.png" alt="Screenshot">
</p>


# Acme

`acmeevent` is a small program that prints all events an **acme** window receives (stolen a from **Plan9 from User Space**).

Compile and install it:

```sh
; cd src/acme/cmd/acmeevent
; mk
; echo $objtype
amd64
; cp acmeevent $home/bin/$objtype
```

Open **acme** and paste the following into the **tag** of the **window** you want to debug:

```sh
cat /mnt/acme/$winid/event | acmeevent
```

Then you should see all events that are received by the **window** (see **event** in acme(4)).

# Go

To install Go one has to bootstrap the installation with an earlier version of Go already compiled for Plan 9. The process consists of obtaining the bootstap version and the target version, followed by telling the target version to use the bootstrapped version in order to build the toolchain.

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

Start the install:
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
