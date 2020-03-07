## Lab a10: Overview
For this lab, we will be installing and configuring a Minecraft server. All of
this configuration can be sucessfully done without a Minecraft client. In fact,
the server runs out of memory before it even starts properly (before Part h was
added)  so it would be impossible to play Minecraft on the server anyways.

## Getting help
If you want any help with any part of this lab, join the OCF slack
(<https://ocf.io/slack>), and mention me (@fydai) in #decal-general.

## The most important trick with puppet

If you mess anything up, delete everything (in particular `/home/minecraft`),
and just run puppet again! Puppet ensures that even starting from nothing, you
can reconstruct your entire previous state. If you do this and get issues with
Puppet executing things out of order than you would like them, add in a
`require` parameter to the resource that should be defined later.  For instance,
if you want to create something after the `/home/minecraft` directory, throw in
an `require => File['/home/minecraft']` option. In general, capitalize the name
of the resource, and put the string before the colon between the square braces.

## Part 1: Installing Puppet

First, we’re going to install Puppet. Feel free to simply copy the commands
below to set up Puppet. Make sure to copy the whole thing!

```
wget https://apt.puppetlabs.com/puppet6-release-bionic.deb && \
sudo dpkg -i puppet6-release-bionic.deb && \
sudo apt-get update && \
sudo apt-get -y install puppet
```

## Part 2: Using Puppet

Make a `minecraft.pp` file anywhere, which through the course of this lab, will
eventually configure and run a Minecraft Server. To run your puppet code, use
sudo `puppet apply minecraft.pp`. Puppet, being declarative, will do nothing if
the system is already configured properly, so __run puppet early and often__ to
detect bugs as soon as possible.

Useful references are this section are the
[Puppet documentation](https://puppet.com/docs/puppet/5.5/puppet_index.html)
and there is a lot of sample code avaliable in the
[OCF Puppet configuration](https://github.com/ocf/puppet). When you are stuck,
looking at existing code and see how they did things will generally be helpful.
Also remember that there are code examples on the slides!

### Part a: Making a Home Directory and a User

Put the following code into `minecraft.pp`:

```puppet
file { '/home/minecraft':
  ensure => 'directory',
}
```
Run `sudo puppet apply minecraft.pp` to apply it, and ensure that the
`/home/minecraft` directory was created.

Inside `minecraft.pp`, write some Puppet code to create a user named
`minecraft`, that the Minecraft server will run under.  The `minecraft` user
should have home directory `/home/minecraft`.  Now modify the code creating
`/home/minecraft` to set owner the owner to `minecraft` user you just made.
Check the example code on the slides if you are unsure about how to do this.

#### Checking Part a:

Run `sudo puppet apply minecraft.pp`.

Now run `ls -l /home` and verify that `/home/minecraft` is owned by the
minecraft user.

### Part b: Install java

Add a few lines to `minecraft.pp` to install the `default-jre` package.

#### Checking Part b:

Run `java` and verify that the binary exists.

### Part c: Installing the Minecraft Server configuration

Copy paste the contents of
<https://raw.githubusercontent.com/0xcf/decal-labs/master/a10/server.properties>
locally into a file named `server.properties`

Ensure that `/home/minecraft/server.properties` contains the contents of the
`server.properties` you just saved.

Hint: Use the `file` function! Also note that in this lab, you should be using
__absolute__ file paths.

Read and agree to the
[Minecraft EULA](https://account.mojang.com/documents/minecraft_eula), and
ensure that `/home/minecraft/eula.txt` contains the text `eula=true` by
hardcoding the string `eula=true` into your `minecraft.pp`.

Make sure that all of the above files are owned by the `minecraft` user.

#### Checking Part c:
Run `ls -l /home/minecraft` and ensure that the above files exist, contain what
they're supposed to, and they are all owned by the `minecraft` user, not yours.

### Part d: Installing the Minecraft Server

Ensure that /home/minecraft/server.jar contains the Minecraft Server, available
at
<https://launcher.mojang.com/v1/objects/3dc3d84a581f14691199cf6831b71ed1296a9fdf/server.jar>.

A previous version of the lab specified the incorrect link
<https://launcher.mojang.com/v1/objects/3dc3d84a582f14691199cf6831b71ed1296a9fdf/server.jar>.
The wrong link has "582" has a substring, whereas the correct link has "581".

Note that the `source` parameter of the `file` resource accepts a URL as its
argument. Also make it owned by the `minecraft` user.

#### Checking Part d:

You know what to do!

### Part e: Templating a systemd unit file

Copy the following template into the same directory into your `minecraft.pp`
file as `minecraft.service.erb`.

Edit the file to be a proper `erb` template, so that `<INSERT YOUR RAM AMOUNT
HERE>` becomes the value of the `memory_available` variable when puppet runs.
You want to use the templated variable `@memory_available` in the `.erb` file,
and declare the variable `$memory_available` it in the `.pp` file.

Hint: In the slides, there is an example of templating a file.

Now edit your `minecraft.pp` file, so that it sets the `$memory_available`
variable to be the __half__ the total amount of RAM available to the system (use
Google and StackOverflow), and that it puts the templated file into
`/etc/systemd/system/minecraft.service`.

Hint: You need to figure out how to define variables and set variables in
puppet.  Note that variables should be prefixed by `$`. You can assign variables
just like any other language. Don't forget to look at the sample code on the
slides (it doesn't cover variable assignment however) and don't forget to use
__absolute paths__!

Note that the systemd unit file does not have a proper ExecStop, which maybe
result in some world corruption.

```
[Unit]
Description=Minecraft Server

Wants=network.target
After=network.target

[Service]
User=minecraft
WorkingDirectory=/home/minecraft
# This should look like ExecStart=/usr/bin/java -Xmx504578 -jar server.jar
ExecStart=/usr/bin/java -Xmx<INSERT YOUR RAM AMOUNT HERE> -jar server.jar
ExecStop=/bin/kill -- $MAINPID TimeoutStopSec=5

[Install]
WantedBy=multi-user.target
```

#### Checking Part e
Look at `/etc/systemd/system/minecraft.service` to ensure it contains the
contents you want before proceeding.

### Part f: Running the service

Ensure that the `minecraft` systemd service is enabled and started.

#### Checking Part f

This is the critical moment! If you've done everything before correctly, this
should work (until Minecraft OOMs)!  If the systemd unit fails immediately, try
to run the ExecStart command manually, by going into `/home/minecraft/` and
running `sudo -u minecraft java -Xmx1009156 -jar server.jar`.

You can verify that something is trying to start by running `tail -f
/home/minecraft/logs/latest.log`. If it ever stops loading, the server has run
out of memory, and "Part h" below should have a workaround.


### Part g: Backups

We should backup our minecraft server!

Ensure there is a directory `/home/minecraft/backups/`, owned by the `minecraft`
user.

Ensure there is a script, `/home/minecraft/backup.sh` that is executable, with
the following contents however you'd like.

```sh
#!/bin/sh
cp -r /home/minecraft/world "/home/minecraft/backups/world-$(date -Is)"
```

The command copies the directory containing into the minecraft world into a
subdirectory of `backups` indexed by the current date.

Use puppet to add a cron entry to execute `/home/minecraft/backup.sh` every
minute as the minecraft user.

#### Checking Part g:

Look in the `/home/minecraft/backups` contains backups!

### Part h: (Bonus, optional if you want Minecraft to not Out Of Memory)

We could be doing this by typing this a bunch of commands to add a swap file,
and enabling it as swap, but we will do this puppet style!

Configure puppet to create a 4GB file `/swapfile` with the command `dd
if=/dev/zero of=/swapfile bs=2M count=2048`.  Look through the flags to the
`exec` resource to see how you can do this.

Now configure Puppet to run `mkswap /swapfile && swapon /swapfile` unless swap
is currently active, which you can check by seeing if `swapon -s | grep
/swapfile` returns a zero code. There are two arguments that can do this,
`unless` or `onlyif`. Experiment to see which one works.

#### Checking Part h:
Run `swapon -s`, and check that `/swapfile` is listed. Ensure that there is no
extranous output when you run puppet, if your checks are correctly done, this
shouldn't happen. Check `/home/minecraft/logs/latest.log` to make sure that the
server has start up properly. If it has, then congratulations!

### Part i: (Bonus, running Minecraft)

Due to security issues, the Minecraft server running is only listening to
`127.0.0.1`, which means by default you can only access a Minecraft client
running on the VM. You have two ways to get around this. One is changing
`server-ip` in server.properties to `0.0.0.0`, which will allow access by
anybody. If you do this you probably want to set up a whitelist. The other
option is SSH port forwarding. The command, if run on your machine, captures
traffic at port 25565 on your local computer, and forwards them to port 25565 on
your VM. The command will run forever, just leave it in the background while you
try to connect.

```
ssh -NL 25565:localhost:25565 <your VM>
```

If you chose the first option, you can connect by typing your domain name or IP
into minecraft, if you chose the second, you can connect to `localhost`.

## Part 3: Cleanup

There is currently a cronjob copying the minecraft world every minute. That
might run you out of disk space. To disable that, you should run `sudo crontab
-e` and remove the line for the backups.

Trying to run a Minecraft Server constantly might also eat up some system
resources. You can stop and disable the minecraft systmemd unit manually if it
is causing issues.

If you added the swapfile, you might want to remove that.

Another way you can clean up is with puppet. By default, puppet doesn't remove
files it doesn't know about. However, you can use `ensure => absent` to make
sure files are gone, and similarly for the other resource types.

## Part 4: Submission

Congratulations on finishing the lab!

To submit, copy paste the code you have for each section into the gradescope
submission.
