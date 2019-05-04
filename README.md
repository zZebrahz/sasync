# sasync

This basic script uses rclone with google service accounts to do server-side copy or sync between rclone remotes using sync 'sets'.
If you only use a few service accounts then this may be overkill. But if you have many SAs or many TDs it can help automate syncs/backups.
It is written in bash to allow it to function in most OSes without installing dependencies. Feel free to adapt it to other languages.

Before using this script it is important that you understand how to use service accounts with rclone. There are plenty of 
threads in the rclone forum as well as discord channels to learn about SAs (i.e. Don't ask me). 
See  https://rclone.org/drive/#service-account-support 

In most cases if you are having problems the culprit will be your permissions. Permissions for Team Drive copy/sync are pretty straighforward. 
Copy/sync with My Drive can be done but is a bit more tricky. You may need --drive-shared-with-me and --drive-impersonate, as SA accounts
only see My Drive files in Shared with Me.

There are several files in the repo:

1. sasync is the main script that pulls sync sets from 'set' files and runs rclone sync (or copy with the -c flag).
2. json.count is an external counter file for the json set. The easiest setup is to number your jsons 1.json, 2.json etc.
3. set.* files specify rclone source, destination, transfers, checkers, chunks, number of service accounts and max-transfer size.
4. exclude.txt specifies file patterns to exclude. Some files hang rclone server-side copying. If that happens to you add them to exclude.txt .
5. sacopy simply runs sasync with the -c script. Sometimes easier to use for clarity (copy vs sync batches)

Each set.* file specifies which sync pairs you would like to run along with rclone flags for that sync pair. The set file format is:
<pre>
# set.tv
# 1source    2destination   3transfers  4checkers   5chunksize     6SAs     7maxtransfer
td_tv:       my_tv:         4           20          16M            5        600G
td_tv_4k:    my_tv_4k:      2           20          16M            2        500G
</pre>

Run the script with this syntax "./sasync set.tv" to cycle rclone sync for each source-destination pair and each block of SAs in your set.* file.

You can run rclone copy rather than sync by using the -c flag.  e.g. "./sasync -c set.tv"

Each sync set can have an unlimited number of sync pairs.

If you prefer to create smaller/separate sets then you can create individual set.* files for each subset. e.g. tv, movies, etc.

If you have multiple sets then you can run them in sequence with "./sasync set.tv set.movies set.books" or in parallel with "./sasync set.tv & ./sasync set.movies &" .

If you don't have many SAs but you want to run multiple source-destination pairs try setting --max-transfer to a lower number (100G, 200G ...)

WARNING: At the moment there is an issue with rclone and server side copies/syncs when you hit the hard limit of 750G uploads per day. 
If there are files in the transfer 'buffer' and you hit the hard limit then rclone can hang indefinitely. There are a few ways to mitigate this problem.
1. Use a hard timeout. This is embedded in the sasync script but can be hashed out # if you want to omit it
2. Use conservative --max-transfer settings so that the number of --transfers times typical library file size will never hit the hard upload limit. 
e.g. If 4k file sizes are typically 40-80GB and you are doing 4 simultaneous transfers then set --max-transfer to < (750 - (4 x 80)) , say 400GB.
3. Add --disable move,copy to the rclone script. This will slow down your transfers as they are downloaded then uploaded rather than server side. 
But rclone will finish non-server-side transfers that are have been started rather than hanging.


Resource usage for this script is very light as the copy/move actions are all executed server side. 
The script can be modified to use the --disable move,copy flag if you prefer, in which case I/O and CPU usage will rise. chunk size in the set files to speed up copying.
If you do use --disable move,copy you may want to increase allocated resources when running the script.

NOTE: --fast-list increases RAM usage. You can delete the fast list flag and the script will run fine, but scans will take a bit longer. 
You can also try using --no-traverse rather than --fast-list.

There is a clean_tds option in the script to dedupe and remove empty directores from source or destination and to delete trash. 
If you don't need that feature simply insert a hash '#' before the clean_tds command or in front of individual lines in the function.

This script runs on any system with bash >4.x (before which the readarray command did not exist). The current version of bash is 5.x. On MacOS you will need to install a newer version of bash, which is easy enough.
