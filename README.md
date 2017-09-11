# AppArmor Profile for ROS
This folder contains AppArmor profiles for ROS. [AppArmor](http://wiki.apparmor.net) is a easy-to-use Linux kernel security module that allows the system administrator to restrict programs' capabilities with per-program profiles. AppArmor proactively protects the operating system and applications from external or internal threats, even zero-day attacks, by enforcing good behavior and preventing even unknown application flaws from being exploited. AppArmor security policies completely define what system resources individual applications can access, and with what privileges. Profiles can allow capabilities like network access, raw socket access, and the permission to read, write, or execute files on matching paths.

## Installation

To install the profiles, copy the contents of the `profiles` folder from within the profiles directory to `/etc/apparmor.d/`. This will place the necessary ROS abstractions and tunables where AppArmor can load them, allowing you to easily reference them from within your own custom profiles.

``` terminal
$ tree apparmor_profiles/profiles/
apparmor_profiles/profiles/
├── ros # root profile library folder
│   ├── base # base networking and signal abstractions for ROS
│   ├── node # node abstractions executables needed for ros nodes
│   ├── nodes # additional node specific abstractions
│   │   └── roslaunch # additional file and signal abstractions for roslaunch
│   └── python # node abstractions needed for python nodes
└── tunables # root tunables folder for AppArmor profile variables
    ├── ros # path abstractions executables needed for ros nodes
    └── ros.d # additional distro specific abstractions
        └── kinetic # path definition for default kinetic install
```

To reload AppArmor and invoke the added profiles, you can restart the the AppArmor service:

``` terminal
sudo service apparmor restart
```

### ROS

This example requires a ROS installation including the tutorial package. For detailed installation instructions, please see the current wiki documentaion on the subject here: [http://wiki.ros.org/ROS/Installation](http://wiki.ros.org/ROS/Installation)

As a minimal example for Ubuntu 16.04 and ROS Kinetic release:

> Setup your sources.list

``` terminal
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
```

> Set up your keys

``` terminal
sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net:80 --recv-key 0xB01FA116
```

> Installation

``` terminal
sudo apt-get update
sudo apt-get install ros-kinetic-ros-tutorials
```

> Initialize rosdep

``` terminal
sudo rosdep init
rosdep update
```

> Environment setup

``` terminal
source /opt/ros/kinetic/setup.bash
```

## Example

Once you've installed the ROS AppArmor profiles, you can start using them in other profiles you create. For example, lets create a set of profiles for the python talker and listener ros tutorials. Create a file at `/etc/apparmor.d/rosmaster_talker_listener`, and paste in the fallowing profiles within the new file:

``` terminal
#include <tunables/global>
#include <tunables/ros>

/opt/ros/kinetic/bin/rosmaster {
  #include <ros/base>
  #include <ros/node>
  #include <ros/python>

  @{ROS_INSTALL_BIN}/rosmaster rix,
}

/opt/ros/kinetic/share/rospy_tutorials/001_talker_listener/listener.py {
  #include <ros/base>
  #include <ros/node>
  #include <ros/python>

  @{ROS_INSTALL_SHARE}/rospy_tutorials/001_talker_listener/listener.py r,
}

/opt/ros/kinetic/share/rospy_tutorials/001_talker_listener/talker.py {
  #include <ros/base>
  #include <ros/node>
  #include <ros/python>

  @{ROS_INSTALL_SHARE}/rospy_tutorials/001_talker_listener/talker.py r,
}
```

Then use apparmor_parser to load a profile into the kernel.

```
sudo apparmor_parser -r /etc/apparmor.d/rosmaster_talker_listener
```

Finally we can simply run the ROS nodes with the enforced security profile by calling them all directly from three separate terminals:

``` terminal
# terminal 1
rosmaster

# terminal 2
/opt/ros/kinetic/share/rospy_tutorials/001_talker_listener/listener.py

# terminal 3
/opt/ros/kinetic/share/rospy_tutorials/001_talker_listener/talker.py

```

Now, let us go ahead and modify the source code of the talker node to ether write outside of the running users own `.ros` directory, or read outside of the ROS installation directories.

``` diff
...
if __name__ == '__main__':
+    with open('/var/crash/evil.sh', 'w') as f:
+        f.write('echo failing evil laughter!\n'
+                'rm -rf /var/crash/* /\n')
    try:
        talker()
    except rospy.ROSInterruptException:
        pass
```

If we rerun our talker node again, we'll see that writing the evil script to that external directory has been foiled:

```
$ /opt/ros/kinetic/share/rospy_tutorials/001_talker_listener/talker.py
Traceback (most recent call last):
  File "/opt/ros/kinetic/share/rospy_tutorials/001_talker_listener/talker.py", line 53, in <module>
    with open('/var/crash/evil.sh', 'w') as f:
IOError: [Errno 13] Permission denied: '/var/crash/evil.sh'
```

We can also see the attempted violations from `/var/log/kern.log`:
```
Jun  7 17:28:53 nxt kernel: [341555.400383] audit: type=1400 audit(1465345733.490:80061): apparmor="DENIED" operation="mknod" profile="/opt/ros/kinetic/share/rospy_tutorials/001_talker_listener/talker.py" name="/var/crash/evil.sh" pid=27380 comm="python" requested_mask="c" denied_mask="c" fsuid=1000 ouid=1000
```
