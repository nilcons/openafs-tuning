# Problem statement

At nilcons, we use AFS not just on a local network (like many
universities and enterprises), but over the internet, in the best case
having 10-12ms of latency (and sometimes much worse, e.g. when
travelling).

OpenAFS unfortunately performs very badly with the default settings
under these circumstances, because of window size.

In this project, we set out to debug and fix this, concentrating on
this simple use case: small AFS installation of one server, used by
2-3 clients at the same time.

# Status quo, performance with default settings

If we calculate with 12.5ms roundtrip time, that gives us 80
roundtrips per second, and if we have a packet size of 1500 bytes,
then without any send/receive window, a simplified send-waitforack
protocol would result in a maximum of 120000 B/s.  If we have a window
size of 32, then potentially we might get 3840000 B/s (~3.84 MB/s).

In practice, OpenAFS, which has a maximum window of 32 packets,
achieves ~2 MB/s in a situation like this, no matter what is the
bandwidth that you have between the hosts.

Most probably, there are multiple reasons for the huge 50% difference
between theory and pracitce: packet size of 1500 was an overestimate
(MTU is usually a bit less on the internet, and not the whole packet
is useful AFS data, we need space for IP headers, UDP headers,
encryption, checksums, etc.); processing on the fileserver also takes
time, and we calculated only with network roundtrip; processing on the
client side also takes time.

Overall, 2 MB/s is not acceptable: you can't stream a high-quality
movie, you can't download an install ISO for a VM in reasonable time.

If we back git-annex or any other stuff with a storage of 2 MB/s, then
that will just simply not be good enough.

# Is this being fixed in the OpenAFS community?

Looking at the mailing list, there is active development happening in
this area, so our hope is, that performance will greatly improve in
the coming decades, but OpenAFS development always takes time, and
it's even worse now, with Auristor.

# Temporary solution

So, here is the temporary solution, that boosts this situation to
10MB/s, an 5x increase, without too much work:
  - have to recompile the debian package with increased rx windows
  - have to apply some tuning on the server side
  - have to apply some tuning on the client side

Until when 10MB/s is good enough with all the new software engineers
and startups, disrupting some businesses on a daily basis?  Who knows,
but definitely longer than 2MB/s.  In the meantime, we just have to
pray, that electrical engineers designing routers, SFP modules,
network cards, network cables, etc. can make the latency smaller,
OpenAFS developers can make the code more performant, while the rest
of us in the IT industry keep adding bloat...  Jokes aside, let's hope
that we don't have to do a second round of "Temporary solution" in 10
years.

## Patch OpenAFS for bigger rx windows

```
apt-get install --no-install-recommends devscripts
apt-get build-dep openafs
apt-get source openafs
cp openafs_1.8.8.1.orig.tar.xz openafs_9.1.8.8.1.orig.tar.xz
cd openafs-*
wget -O debian/patches/biggerwindows https://raw.githubusercontent.com/nilcons/openafs-tuning/master/biggerwindows
echo biggerwindows >>debian/patches/series
DEBEMAIL=localbuild@company.com dch -v 9.1.8.8.1-1+biggerwindows biggerwindows
dpkg-buildpackage -us -uc -rfakeroot
```

Use the resulting deb packages (with `dpkg -i`) on your workstations
and on the server.  This is the part, that you will have to constantly
maintain on your machines, as OpenAFS releases new versions to keep
the kernel module compatible with the kernel changes.

(Note: I know that we should go with `9:` instead of `9.` in the
version number to force debian to always use our version number, but
unfortunately if you try `9:`, you will see that openafs-modules-dkms
or dkms has a bug somewhere, that with a version number like that the
module build fails.)

## Tuning the server

`/etc/openafs/BosConfig`:

```
...
parm /usr/lib/openafs/dafileserver -p 23 -busyat 600 -rxpck 400 -s 1200 -l 1200 -cb 65535 -b 240 -vc 1200 -sendsize 4000000 -udpsize 4000000
...
```

Essentially, we just added `-sendsize 4000000 -udpsize 4000000` to
increase the sending buffers.

These definitely needs more work and fine tuning, if you are serving
many users; if you figure out how to fix the issue in bigger cells,
please do not hesitate to open a PR, I don't have a bigger cell and
users to play with.

## Tuning the clients

```
echo 'OPTIONS="-daemons 6 -chunksize 22"' >>/etc/openafs/afs.conf
```

I guess this would also need some tuning, if you have multi-user shell
servers, but I tested this quite a lot on one-user-per-laptop
machines, and in that scenario, this works OK.
