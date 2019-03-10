---
layout: post
title: "Example to use python-daemon and runner.DaemonRunner"
date: 2019-03-10
excerpt: "Just example."
tags: [python, daemon.runner]
comments: true
---

# What's the daemon?

  Daemon is the process running in background and do some task e.g. write the log or response from the client.

  In technically, the PID of parent process of daemon process is 1 and no terminal to control this process.

  And __you cannot run the same daemon processes if you ran it__.


  According to Stevens in [stevens](https://books.google.co.kr/books/about/UNIX_Network_Programming.html?id=Rc1QAAAAMAAJ&redir_esc=y), a program should perform the following steps to become a Unix daemon process. (From [this](https://www.python.org/dev/peps/pep-3143/#correct-daemon-behaviour) document.)
   
   - Close all open file descriptors.
   - Change current working directory.
   - Reset the file access creation mask.
   - Run in the background.
   - Disassociate from process group.
   - Ignore terminal I/O signals.
   - Disassociate from control terminal.
   - Don't reacquire a control terminal.
   - Correctly handle the following circumstances:
   - Started by System V init process.
   - Daemon termination by SIGTERM signal.
   - Children generate SIGCLD signal.


# How to make daemon process in Python.

 Python provides [standard daemon process library](https://www.python.org/dev/peps/pep-3143/). And I made simple example code like following.



``` python
import time
import os
import logging
from daemon.runner import DaemonRunner

_base_path = "/path/to/daemon"

class MyApp(object):
    """
      Define the required attributes
    """
    stdin_path = "/dev/null"
    stdout_path = os.path.join(_base_path, "myapp.out") # Can also be /dev/null 
    stderr_path =  os.path.join(_base_path, "myapp.err") # Can also be /dev/null
    pidfile_path =  os.path.join(_base_path, "myapp.pid")
    pidfile_timeout = 3 

    def run(self):
        # Basic logging 
        logging.basicConfig(format="%(asctime)s [ARDUINO/%(processName)s] %(levelname)s %(message)s",
                            filename=os.path.join(_base_path, "app.out"),
                            level=logging.INFO)
        logging.info("Starting")
        try:
            with open(os.path.join(_base_path,'spam.data'), 'w') as o:
                while True:
                    # Loop will be broken by signal
                    o.write("Hey\n")
                    logging.info("Hearbeat")
                    time.sleep(1)
                o.write("Done\n")
        except (SystemExit,KeyboardInterrupt):
            # Normal exit getting a signal from the parent process
            pass
        except:
            # Something unexpected happened? 
            logging.exception("Exception")
        finally:
            logging.info("Finishing")

if __name__ == '__main__':
    """
      Call with arguments start/restart/stop
    """
    run = DaemonRunner(MyApp())
    run.do_action()
```
