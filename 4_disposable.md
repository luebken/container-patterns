# 4. Disposable

The [Pets vs Cattle](https://blog.engineyard.com/2014/pets-vs-cattle) is the infamous article about an analogy which differs two different server types. There are pets which you give names and want to hold on and cattle which you give numbers and can be exchanged easily.

A module container should always strive to be able to be exchanged with a fresh instance at any point of time. This is especially true in a cluster environment, where there are many reasons that a particular container can be stopped:
 
 * Rescheduling because of limit or bad resources
 * Down-scaling
 * Errors within the container
 * Migration to new hardware / locality of services

This concept is so widely accepted in the container space that developers use the `--rm` with Docker as a default which always removes the container after it has stopped. We have chosen the term "disposable" from the [12Factor app](http://12factor.net/disposability).

**Best practices on being disposable:**

* Be robust against sudden death.  
  If the container gets interrupted, pass your current job on to another instance of possible. (See ["React to signals"](#react-to-signals))
* Minimal setup  
  If more setup needed let the scheduler know and use [hooks](#hooks).