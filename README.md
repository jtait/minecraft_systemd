# minecraft_systemd
systemd files to run a Minecraft server

### Description

Includes an instantiable service and accompanying socket. The socket connects to stdin of the server process to allow commands to be passed, including the stop command used by the service.

This can create multiple servers by instantiating the service with different instance names: e.g. minecraft@vanilla.service, minecraft@modded.service, minecraft@snapshot.service, etc.  Each server must still have its own unique port assignments, and they must be located in separate sub-directories of ```/srv/minecraft```.

The service expects certain conventions:
1. All server files must be stored in ```/srv/minecraft/<instance_name>```, where _<instance_name>_ is the name of the systemd instance.
    e.g. minecraft@vanilla.service would require a directory at ```/srv/minecraft/vanilla```
1. The service is run as the user "minecraft".  This user needs read/write permissions on the ```/srv/minecraft/<instance_name>``` directory.  All the server files should be kept inside this directory. The "minecraft" user should be a restricted user, and does not require a shell or home directory.

### Installation
1. Copy the .service and .socket files into /etc/systemd/system.
1. Execute ```sudo systemctl daemon-reload```
1. If you want to provide access to a group of people that don't have elevated privileges on your system, copy ```minecraft.sudoers``` into ```/etc/sudoers.d```.

### Basic Usage Example - Vanilla Minecraft Server
1. Create a folder in ```/srv/minecraft``` named 'vanilla', and ensure it is owned by minecraft:minecraft.
1. Get a copy of the server jar from [minecraft.net](https:://minecraft.net) and place it in the folder.
1. To keep things simple, rename the jar "minecraft.jar".  Alternatively, create a symlink in the folder, e.g. ```ln -s /srv/minecraft/vanilla/minecraft_<version_number>.jar /srv/minecraft/vanilla/minecraft.jar```.  Creating a symlink has the advantage of keeping your original file with the version number, and when you want to update you can get the new jar and just delete and recreate the symlink, in case you need to roll back to the old jar.
1. You can start the service now, but it will fail. That's because the first time the server starts it creates a number of files and directories, including the eula.txt file.  If you decide to start the server manually for the first time, make sure you correct the permissions once it fails by running ```chown -R minecraft:minecraft /srv/minecraft/vanilla``` otherwise the systemd service will fail.
1. To start the service, run ```sudo systemctl start minecraft@vanilla.service```.

### Configuration

The systemd files do not need to be edited directly.  In order to configure the service, add a file named "systemd.conf" inside the root of your server directory.  Following the examples so far, this would be in ```/srv/minecraft/vanilla/systemd.conf```.

The available variables are:
- MIN_MEM - applies a minimum memory value for the Java virtual machine.  This is used as -Xms${MIN_MEM}, so it requires a valid format (e.g. 1024M or 2G)
- MAX_MEM - same requirements as MIN_MEM, but applies to the maximum memory for the Java virtual machine
- JAR_PATH - Set the path to the jar file for the server.  Can be relative to the working directory (```/srv/minecraft/<instance_name>/```) or can be an absolute path.  The jar still needs to be within the working directory in order to function correctly.

### Socket
The service uses a socket to pass data on stdin.  This socket is created when the service starts and is destroyed when the service stops. While it is technically possible to send commands directly to the socket this can be prone to typos.  THe socket FIFO is intentionally restricted to only be accessible by the minecraft user.

It may be tempting to try and write scripts to and write to the socket as the minecraft user, but this is not a supported or particularly safe operation.  Instead, use an rcon client to send commands to the server, or if you don't need anything fancy the in-game commands still work as expected.

### Logging
The service logs to journald for stdout and stderr.

### Extra notes, not covered by these files
1. You should run a firewall on your server. You need to add a rule to allow the server port access.  The port is configured in server.properties in the root of the server files (e.g. ```/srv/minecraft/vanilla/server.properties```).  I use [ufw](https://launchpad.net/ufw) on Debian 10.  It's simple to configure and use.  If your system uses a different program that would work too.
1. You shouldn't give everyone who needs access to run the service sudo access.  This is a security hazard, and is actually pretty easy to restrict but still have good access.  You can instead modify the sudoers file to allow access to the service to just members of the minecraft group.
    - Placing ```minecraft.sudoers``` in ```/etc/sudoers.d``` on most systems will add the necessary rules.  Otherwise you can edit the sudoers file itself.  NOTE: ALWAYS use ```visudo``` to edit the sudoers file, or use known tested files in ```/etc/sudoers.d```.  An error in the file can cause serious problems that are difficult to recover, and ```visudo``` will check for errors before saving which will prevent possibly getting locked out of your system.
1. You need to make sure to configure ports for the server and rcon in the individual server.properties files.  These files don't configure the server, just the JVM that runs the server files.
1. Some modded server or plugins require custom jars or come with a startup script.  For example, most FTB servers include a startup script (ServerStart.sh) that runs the java command with arguments.  This service file only supports calling Java directly, not running the startup script.  You can read the startup script and find the necessary variables to run the java command in the systemd file.

### Issues and Pull Requests
If you use these files and find an issue, please report it via the issue tracker.
If you have a fix for an issue, or if you have an improvement, pull requests are welcome.  However, these files are intended to be basic and simple to use, so keep that in mind.  These aren't going to configure a server from scratch, they're only intended to handle day-to-day execution after the server has been configured.

---
This has been tested on Debian 10, but should work on any Linux system running systemd.  I would add the caveat that it's not tested with SELinux, so your mileage may vary.
