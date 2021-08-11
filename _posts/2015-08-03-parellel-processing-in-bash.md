---
title: Parallel Processing in Bash
date: 2015-08-03
tags:
  - Bash
  - Dirty Tricks
---

So you need to perform a task or run a command across a medium-to-large number of hosts, in some kind of controlled fashion. The typical options are:  

*   Grinding through a list of entries one-by-one with a script
*   Using tools like GNU parallel or xargs, to process multiple entries at a time
*   Using a vendor-provided API, or else an orchestration tool of some kind

The first is simple, but slow. The second is more efficient, but leaves you open to race conditions and clobbering- especially if your parallel jobs need to write into the same files. The third is ideal, but it presumes that you have those orchestration tools in place (and the knowledge to use those tools).
  
In our case, we needed something better than option 1, more controllable than option 2... and there was no option 3 available at the time (cough *Ansible* cough)

### Background Processes
  
The quick-and-dirty way to parallelize a process in bash, is to run it in the background- done by putting an ampersand after the command. You can also do this with functions inside your bash script. Below is a simple reboot script:
{% raw %}
```yaml
function reboot_host {
  server=$1
  ssh $server "reboot"
  sleep 60
} 

# Feed our server list into a while loop, and ask to reboot for each server.

while read -r hostname; do
read -n 1 -p "Reboot host ${hostname}? [y/n]: " choice

  case $choice in
    y|Y)
    reboot_host ${hostname} &
    ;;
    n|N)
    echo "Stopped on ${hostname}"
    exit 1
    ;;
    *)
    exit 1
    ;;
  esac

done < /path/to/server_list
```
{% endraw %}

The above would let you kick off background instances of the **reboot\_host** function as fast as you could mash the 'y' key, until you have read through your server list.  

### File Locking

In some cases, you may want to maintain a log file, or a list of successful/failed/remaining items. So now you have to worry about multiple background processes clobbering with each other, when they try to write into the same set of files.  
  
One Bash-friendly way to avoid this clobbering, is to use **[flock](http://linux.die.net/man/1/flock)**: a utility that provides a kernel-supported method for file locking. The idea is to assign a file handle (or descriptor) to a file, and then have your running script or function use flock to claim a lock on that file descriptor. If other jobs also use flock to claim access to a file, they will wait for any existing flocks to be released, before continuing their activities. It looks like this in action:
{% raw %}
```yaml
# Open or create a file descriptor into file /path/to/server_list (can be any number but 0, 1, or 2)
exec 200>/path/to/server_list || echo "ERROR: descriptor not opened!"

# Acquire an exclusive lock on descriptor 200 (-e 200), or wait until descriptor is available for up to 30 seconds (-w 30)
flock -w 30 -e 200 

# any commands to run when you get the lock...
  echo "$servername" >> /path/to/server_list
  command1
  command2
  command3

#release lock on file descriptor 200 when done
flock -u 200
```
{% endraw %}
  
In the above example, you have to explicitly call flock twice: once to get the lock, and a second time to release it. A more elegant method is to use a subshell () that runs when you open the file descriptor. In this case you don't have to worry about releasing the flock manually- it will release automatically at the end of the subshell, when the file descriptor closes. The usual Bash restrictions apply when working with subshells.
{% raw %}
```yaml
function process_item {

  # take item as argument
  item=$1

  (
    flock -w 30 -e 200 || echo "ERROR: flock attempt failed!"
    echo -e "$item" >> /path/to/processed_list
    sed -i "/^${item}$/d" /path/to/remaining_list
    sleep 10

  ) 200>/path/to/lockfile

}

process_item item01 &
process_item item02 &
```
{% endraw %}   

Above, item02 should take 10 seconds to appear on the lists, since item01 is keeping the flock open on the lockfile while sleeping.
  
Some important things to note:

*   You can use the optional **\-w** flag to set a timeout on any flock attempts, in case something goes wrong. Then you don't worry about a rogue flock attempt hanging forever.
*   To control write access, you don't have to put a flock on the same descriptor/file that you are writing into. In the second example, function **process\_item** uses a separate lockfile to keep track of when it is allowed to write into several other files. 
*   Running flock with the **\-n** flag makes for a non-blocking flock, meaning it will fail with a return code instead of waiting for the descriptor to be available. Useful if you know you only want once instance of a process opened, no matter what. The default is a blocking flock, which will wait for access.
*   Flocks work against file _**descriptors**_, and not on the files directly- it only has significance to other jobs using flock on that descriptor. It doesn't magically protect the file from modification by other processes.