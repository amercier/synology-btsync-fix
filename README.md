synology-btsync-fix
===================

BitTorrent's Sync service (btsync) is a great tool which, in combination with a Synology allows you doing fancy things. Unfortunately the people behind the Synology community package didn't put as much love into it as it would deserve.

The BitTorrent Sync default installation on a Synology NAS creates a "btsync" user that runs the homonymous service. This makes it impossible to write data into other user's home-directories (at least, if they're configured in a safety-conscious way ;-)). Besides, each user should be allow to use their very own version of the WebGUI, to configure sync directories and retrieve access codes.

Fixing
------

Now, here's what I did to solve this issues:

First, I modified the BitTorrent Sync init.d-script ("start-stop-script"). You can find an up to date version of it within this repository.

The script contains of a variable named USER. All users that should run their own btsync process have to be added (space-separated) there.

Next, I overwrote the package's original script at `/var/packages/btsync/scripts/start-stop-status` with my version, by connecting to my NAS through SSH, as "root" (Using `ssh root@IP` – [more info](http://www.chainsawonatireswing.com/2012/01/06/logging-in-to-the-synology-diskstation-as-root/)). 


**Make sure the BitTorrent Sync service is turned off before!
**

Then, I changed permissions to the `/usr/local/btsync/var/` folder, so that every user is able to create a pid-file:

	chown btsync.users /usr/local/btsync/var
	chmod 770 /usr/local/btsync/var

After that, I created the btsync data-folders and config files for each of the specified users:

	mkdir /volume1/btsync
	mkdir /volume1/btsync/user1
	touch /volume1/btsync/user1.btsync.conf
	chown user1.users /volume1/btsync/user1*
	mkdir /volume1/btsync/user2
	touch /volume1/btsync/user2.btsync.conf
	chown user2.users /volume1/btsync/user2*
	...

(replace user1 and user2 by the usernames you entered in the USER-variable)

And last but not least, use `vi` to edit the "user.btsync.conf" you just touched ([for more information about the vi command"](http://www.computerhope.com/unix/uvi.htm)). 


	cd /volume1/btsync
	vi user1.btsync.conf


Here is an example configuration you could use for the "user.btsync.conf" file:

	{
  		"device_name": "mynas",
		"storage_path" : "/volume1/btsync/user1",
  		"pid_file" : "/usr/local/btsync/var/syncapp-user1.pid",
		"listening_port": 0,
  		"check_for_updates": true,
		"use_upnp": false,
  		"download_limit": 0,
		"upload_limit": 0,
  		"webui" : {
  	  		"listen" : "192.000.0.0:9991",
	    	"login" : "enter here",
  	  		"password" : "enter here"
	  	}
	  
	}


Enter your correct IP/hostname, replacing "192.000.0.0". 

Also make sure that every user uses a different "listen"-port for the webui. 
So for user1, this would be 9991, for user2, 9992, etc. 

Then specifiy a btsync webui "login" and "password", replacing "enter here".

After completing all these changes, login to your Synology NAS web-interface as admin and run the BitTorrent Sync service. It should come up with no issues at all.

Using `ps | grep btsync` you should be able to confirm that there will be multiple btsync processes, ran by the users you specified. Each user can then connect to the btsync webui using your Synology's IP/hostname and the port you specified in the "user.btsync.config" file (For example: 192.000.0.0:9991).

The Downside (yes, unfortunately there is one)
----------------------------------------------

As soon as a new update for the BitTorrent Sync package will be available, the modified start-stop-script will probably get overwritten with the package's script. Actually I don't expect BitTorrent to change a lot within this script, meaning that you should be able copy away your version of the script before updating and then copy it back afterwards, without running into any issues.

Another thing that could happen is a permission reset on the `/usr/local/btsync/var/`-folder. It depends on whether the update will remove and re-create this folder or just leave it as. However, simply reset the permissions of this folder as shown before and everything should be okay again.

If the service still won't start, try launching it through the root-SSH console, by simply running `./start-stop-status start` within the `/var/packages/btsync/scripts/` directory and see what happens.

If you get the error

	su: can't run /sbin/nologin: No such file or directory

You will need to amend the `/etc/passwd`, finding the line with your Sync user which will contain for example "/var/services/homes/user1:/sbin/nologin".

	vi /etc/passwd


Change the last part of the line, where it says "/sbin/nologin". It needs to say "/bin/sh".
 


Compatibility
-------------
This modification has been tested with the following versions of the Bittorrent Sync package for Synology:

- 1.3.106-2
- 1.4.72-4
- 1.4.75-5
- 1.4.83-8

Feel free to test it with other versions yourself and report back, if it works. Always back up your files before testing!
