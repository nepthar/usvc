usvc
====

User Service - Run &amp; Manage a process in user space.

Given that I now have a job, I'm no longer ashamed to share this abomination with the world. It's just a glorified process manager.

What?
-----
I wanted to run my little programs on a Linux box somewhere. If they crashed, I wanted them to be restarted. I really didn't care exactly how this was done, I just need it to happen so I could move on to other things. I had no idea what an init script was or an upstart job. I also felt very strongly that my programs should be run in as a user in an interactive environment.

Why?
----
Back when I started Lumatic, I knew basically nothing about computers and every time I tried to use someone else's framework or system, I'd get burned. So I wrote my own using some tools I was familiar with. Then I over-engineered it because hey, why not? I also thought it was really cool to be able to see my stdout/stderr in realtime and log things that bash tells you about your process (like if it gets killed).

How?
----
It's a bash script that interfaces with tmux. When you start up a process, it runs the process in a loop and provides you with some stats.

And…?
-----
And it worked. Lumatic's entire backend system was controlled by usvc. Despite its flaws and silliness, it took me about a week to write usvc as a total amateur coder (at the time, I used Dropbox for version control) and I never once had to do any maintenance on it.

And…?
-----
And Lumatic isn't around anymore. It had nothing to do with usvc.

Code Review
===========
I've added a few comments to the code here and there during this process of putting usvc on github. These new comments are all dated

There's a lot of style inconsistency in this script, some of which is interesting.

How usvc Works
==============
The script basically just wraps tmux and runs each process in a loop. It is conf-file based internally and uses a conf file per process to manage things. For example, when the user calls `usvc start [process]`, what actually happens is `usvc start` generates a conf file for that process and then passes it to the process running code.

One really useful feature of usvc was process groups. You could assign a group to your process so you could identify batches of process by name. Want to restart all 'group1' processes? `usvc restart group1`.

Probably the main reason why I stuck with usvc for so long was its simplicity. Running a job is as simple as `usvc start [job command] [job args]`. As long as you've got the correct environment set up for running the job interactively, it will run in usvc just fine.

dependencies
------------
git, tmux, linux (only tested on ubuntu) and bash 4. Maybe others that I haven't thought about really.

prop.* functions
----------------
At some point, I decided to use `git config` to manage my config files. The prop.* functions all wrap calls to `git config`

initialization: `init()`
-----------------------
Before any processes are started, usvc has to initialize the tmux session that it uses for management. One of the more important things done here is to have tmux capture Ctrl+C so the user doesn't terminate a process without going through usvc

job lifecycle
-------------
Like any other process management system, jobs have various states of being with well defined transitions. Here's a few states:

*	stopped: The job has a config file but is not running now
*	starting: Well, uh... it's starting.
*	running: Job is running. `usvc status` will report that the job is in whatever state the kernel says its in. ('sleeping' if it's waiting on IO)
*	waiting: Job has respawned too many times and is being throttled
*	dead: Job has just terminated

starting jobs: `start_process()`
--------------------------------
Calling `usvc start ...` generates a configuration file for your job based on the arguments and parameters you've passed in. You can also have start run multiple copies of the same job.

Once the config file for the job is generated, usvc issues commands to tmux to create a new window with the title of the job name running `usvc _r_run path/to/config/file.conf`.

running jobs: `usvc_run_service()`
----------------------------------
This is what actually executes the job in a while loop. All of the job-related information is read from the configuration file passed as the first argument. I would often test this by running it directly rather than usvc/tmux managed.

looking at jobs: `get_status()`
-------------------------------
By far the coolest part, this little function takes a list of config files as arguments. Using linux's /proc virtual files, it generates a single system status summary line and then the status of the jobs corresponding to the config files. The output ends up looking pretty cool and I used to check cluster health by just running this command on all boxes and concatenating the results. Processes having issues stick out like a sore thumb.


Bugs
====
There's probably a bunch, but the only one that I ever ran in to was an issue where tmux would suddenly have a 'ghost' window with a bunch of garbage displayed in it. The actual processes seemed to be completely unaffected. The ghost window had to be removed by issuing the close-pane command.