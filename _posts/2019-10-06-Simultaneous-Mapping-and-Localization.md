---
description: Simultaneous Mapping and Localiation (SLAM)
---

Recently, I've been getting interested in electronics and hardware again.  After
several discussions with my brother, a project we've been working on involves
Simultaneous Mapping and Localization (SLAM).  The idea behind SLAM is to
build up a map of an environment while at the same time keeping track of your
current position within the environment.  As you can imagine, this has many
applications in robotics - for example, some robot vacuum cleaners use SLAM to
construct maps of the areas they're cleaning, which allows them to more
efficiently clean the room instead of bouncing around randomly, and some models
even allow you to [view the maps](https://www.youtube.com/watch?v=mxnwskab5NY).

A common sensor used in SLAM algorithms is lidar, which uses lasers to obtain
distance measurements.  Some SLAM algorithms also incorporate knowledge of the
robot's movement (e.g. if you are controlling the robot's velocity and
direction), while other SLAM algorithms can work with only the (lidar) distance
measurements.  Even without direct knowlege of the robot's movement, the robot's
location and orientation can still be determined because there is a very high
correlation between consecutive lidar measurements.

To get started, we chose to use Google's
[Cartographer](https://github.com/googlecartographer/cartographer), as the
authors published a [nice
paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45466.pdf)
on it and they also provide an integration to [Robot Operating
System](https://www.ros.org/).  We used a low-cost, spinning hobby lidar device
while walking around in an open space, and fed the data through Cartographer.
You can see the algorithm in action in this
[image](/assets/images/SLAM.gif){:target="_blank"} (warning for slower
connections, GIF is 4MB).  The output is a bird's eye view of the room - darker
spots indicate that there is a higher probability of an object being in that
location.  You can also see the trajectory of the lidar (e.g. us walking through
the room) and some other Cartographer-specific information.  When we compared
this map to the actual measurements of the room, it was fairly accurate -
the constructed map was accurate to within 30cm.  I suspect that we can improve
the accuracy further by attaching an IMU sensor and also providing this
information to Cartographer.

There are also similar algorithms that use images/video instead of lidar as the
input - these techniques are called visual SLAM and photogrammetry.  I haven't
looked into those further yet, but I may post about it in the future if I ever
get around to them.

![SLAM-constructed map](/assets/images/SLAM-screenshot.png)
