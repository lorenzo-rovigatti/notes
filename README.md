# My notes

## Removing sensitive data from a git repository

There are many guides and tutorials online, but here I will report a very straightforward recipe taken from [here](http://palexander.posthaven.com/remove-a-password-from-gits-commit-history-wi):

```bash
# Sync with the remote master
git pull

# Force your clone to look like HEAD
git reset --hard

# AGAIN, A WARNING: This can really break stuff!

# Run your filter branch command, replacing all instances of "old_text" with "new_text"
# The example looks for python files ("*.py"), you can change this to match your needs
git filter-branch --tree-filter 'git ls-files -z "*.py" |xargs -0 perl -p -i -e "s#(old_text)#new_text#g"' -- --all

# Overwrite your master with local changes
git push origin master --force
```

## Setting up NFS

[NFS](https://en.wikipedia.org/wiki/Network_File_System) (Network File System) is a protocol that makes it possible to remotely (and transparently) access a file system over the network. Here I will explain how to setup a remote machine (aka the *server*) so that a local machine (aka the *client*) can have access to one or more of its directories.

 1. Install the NFS server. On Ubuntu, this boils down to installing the `nfs-kernel-server` package.
 2. Open (with root privilegies) the `/etc/exports` file. Add a line for each folder you want to export. The syntax is
 
     ```PATH_TO_DIRECTORY  CLIENT(OPTIONS)```
     
     Note the lack of whitespace between CLIENT and the open parenthesis. The CLIENT field can be an IP address or a DNS name and can contain wildcards (*e.g.* `*` or `?`). The list of available options depend on the NFS server version and can be found online (for example [here](https://linux.die.net/man/5/exports)). Valid configuration examples are
     
     ```/home/lorenzo/RESULTS 192.168.0.10(rw,no_root_squash)```
     ```/home/lorenzo/RESULTS *(ro,no_root_squash)```
     
     The above lines export the `/home/lorenzo/RESULTS` directory in read/write mode for the client identified by the IP address `192.168.0.10` and in read-only mode for everybody else.
     
 3. Restart the NFS server. On many Linux distros this can be done with the command `sudo service nfs-kernel-server restart`
 4. On the client, install the NFS client (the `nfs-common` package on Ubuntu)
 5. The remote filesystem can be mounted either manually (`sudo mount 192.168.0.1:/home/lorenzo/RESULTS /home/lorenzo/SERVER_RESULTS`) or automatically by adding a line like this to the `/etc/fstab` file:
 
     ```192.168.0.1:/home/lorenzo/RESULTS /home/lorenzo/SERVER_RESULTS nfs rw,user,auto 0 0```
     
     and then using `sudo mount -a`.  Here I assumed that the server has IP address `192.168.0.1` and that the directory `/home/lorenzo/CLIENT_RESULTS` exists on the client.
     
**Nota Bene:** on default installations, the shared directories are owned by the user and group identified by the `uid` and `gid` of the server, respectively. If the user on the client has a different `uid` and/or `gid`, its access to these directories may be limited. If this is the case (and if you are sure there will be no issues arising from such a drastic change) the best course of action is to change the user and group ids on either machine in order to make them match. This can done with the `usermod` command (*e.g.* `usermod -g 1110 -u 1205 lorenzo` will change lorenzo's `gid` and `uid` to 1110 and 1205, respectively). Note that `usermod` will change the ownership of all the files and directories that are in the user's home directory and are owned by them. The ownership of any other file or directory must be fixed manually.

## Setting up slurm

Many people recommend to compile slurm from the source, but if you do not need the cutting-edge version just use the one packaged for your distro. Once you have installed it consider the next few points:

1. Use [this form](https://slurm.schedmd.com/configurator.html) to assist in the generation of the `slurm.conf` file.
2. On my Ubuntu box I had to run `/usr/sbin/create-munge-key` (with root permissions) to make it possible to start `munged`. Note that the generated key file, which by default is `/etc/munge/munge.key`, should be then copied over to all the nodes of the cluster.
3. Slurm requires a mail program. This can be set in the `slurm.conf` file with the `MailProg` key which, if not set, points to `/bin/mail`. The simplest way of setting up a mailserver is to install postfix and makes it relay email to another server (see for example [here](https://linode.com/docs/email/postfix/postfix-smtp-debian7/)).

## Making movies

We would like to generate a movie out of a (maybe very long) list of png images. We first create a file containing the list of png files sorted according to the order with which they should appear in the movie. If the names of the files are `img-1.png`, `img-2.png`, *etc.*, this can be accomplished by typing

```bash
$ ls -1v img-*.png > list.dat
```

The `-v` option is a GNU extension that tells `ls` to order entries according to the natural sorting of the numbers contained in the entry names. If on your system this option is not supported or it does something else (this might be the case on BSD/OSX systems), then you can use the following command

```bash
$ ls -1 img-*.png | sort -n -t'-' -k 2 > list.dat
```

`-n` applies natural sorting, `-t'-'` tells sort to use dashes as the separators to split the entries it acts on into fields and `-k 2`  to sort according to the value of the second field

A movie of width WIDTH, height HEIGHT and framerate FPS can now be generated by using `mencoder` with the following options:

```bash
$ mencoder mf://@list.dat -mf w=WIDTH:h=HEIGHT:fps=FPS:type=png -ovc xvid -xvidencopts bitrate=200 -o output.avi
```

The `-ovc xvid` controls the type of output and may not be suitable on all systems (or for all the purposes). The `-xvidencopts bitrate=200` option sets a sort of *overall* quality of the output.

## Adding a fading, rounded border to figures with GIMP

1. Select the whole picture (`ctrl + a`)
2. Shrink the border by an appropriate amount of pixels (Select &rarr; Shrink). I would say no more than 10% of the figure size.
3. Round the selection (Select &rarr; Rounded Rectangle)
4. Invert the selection (`ctrl + i`)
5. Feather the border (Select &rarr; Feather). I usually choose a number similar to the one used in step 2.
6. Delete the selected region (press delete)
7. Turn white into alpha (Colours &rarr; Colour to Alpha)
8. Create a new layer (same width and height as the figure) and put it underneath all other layers
9. Fill the layer with the background colour you want your picture to fade to at the border (usually black)
