# hENC

A radically simple hierachical External Node Classifier (ENC) for CFEngine

With just 60 lines of Perl to read and merge settings from plain text files, hENC is probably the simplest external node classifier for CFEngine on the planet. I doubt you can find anything as simple and as flexible and powerful (but if you do please share it because I want to use it!).

This ENC system for CFEngine is based on the following components:

- a CFEngine module, you'll find it in module/henc
- a configuration for the module, in lib/henc_cfg.cf
- a CFEngine agent bundle to run the module, you'll find it in module/enc.cf
- a few CFEngine bodies used in enc.cf, in lib/henc_lib.cf
- a set of text files with classes and variables to represent different systems configurations in your infrastructure

The text files, whose format is based on CFEngine's module protocol, are read by the module to set/cancel classes and variables that represent your infrastructure. Of course, you do want to create these files by yourself and they are not included in this distribution. The format of the files is explained below, along with suggestions about how to build your file hierarchy.


## Does it work on my CFEngine installation? ##

Clone the repository, get into the directory and run

`sudo -u root cf-agent -Kf henc_test.cf`

If you run CFEngine with a user other than root, then replace root in the command line above with the appropriate user name.

If the output looks like this:

```
bronto@murray:~/Lab/hENC$ sudo -u root cf-agent -Kf ./henc_test.cf 
R: OK - global_class_to_be_set_by_henc found
R: OK - global_class_to_be_cancelled_by_henc not found
R: OK - global_class_to_be_lowered not found
R: OK - test scalar has the expected value
R: OK - test list was slashed by henc
R: -----------------------------------------------
R: FINAL RESULT: ALL TESTS SUCCESSFUL!
```

then congratulations, you can use hENC on your system.

If the last line looks like this instead:

```
R: FINAL RESULT: Some tests failed :-(
```

you can either open an issue here, or fork the project, fix the problem and make a pull request.


## How do I install hENC on my system? ##

1. copy module/henc in your modules directory
2. copy module/enc.cf in your masterfiles
3. either copy lib/henc_lib.cf in your masterfiles, or include its content in your site library, if you have one
4. edit lib/henc_cfg.cf according to your needs:
    1. `master_modules` should be set to the directory on the policy hub where you keep your modules;
    2. `local_modules` should be set to the directory on end nodes where you keep a local copy of the modules;
    3. `master_enc` should be set to the directory on the policy hub where you will store your ENC files;
    4. `local_enc` should be set to the directory on end nodes where you will keep a local copy of the ENC files;
5. copy the so modified henc_cfg.cf in your masterfiles, or include its content in your site library, if you have one;
6. add all the .cf files to your inputs in `body common control`; you can also put them somewhere in a `body files control` if the version of CFEngine you're using supports it

## hENC file format ##

Four "primitives" of the module protocol are used in hENC files: `+` to set a class, `-` to cancel a class, `=` to set a scalar variable and `@` to set a list variables. The classes will be global in scope, while the variables will be defined in the `henc` context, e.g.: `$(henc.myvar)`, `@(henc.mylist)`.

In addition, we added two primitives:
* `_` will "lower" a class: the module will forget whatever it knew about that class until then;
* `/` will "slash" a variable (list or scalar): the module will forget whatever it knew about that variable until then.

Data (JSON) containers (the `%` primitive in the module protocol) is not implemented because we haven't completed the transition to CFEngine 3.6.x. However, a glance at the code should be enough to make you realize that implementing data containers in hENC is straightforward.

**Trailing comments or continuation lines are not allowed**: the format is strongly line based.


## How does hENC work? ##

You pass a list of specially formatted text files to a bundle, the bundle runs a module that reads the files, merges the information in them to remove conflicts, and sets or unsets classes and variables.

The text files use a subset of CFEngine's module protocol plus a couple of additions. Anything else in the file is ignored.

### Build a list of settings' files ###

The most important thing to do is deciding how you will build the list of files containing the settings you want to apply to your systems. It's the most important thing, the rest is just "mechanics".

The system is flexible enough to allow for both static and dynamic lists, or anything in between. Actually, hENC doesn't really care how you build your file list: all it asks is to be passed the fully qualified name of the list itself. The following are just suggestions based on our own experience.

We use configuration management on several datacenters around the world. In each datacenter, there may be different "environments" that require specific settings, for example: machines confined in an "isolated" network segment will probably use different DNS servers than the rest, so having a file to map specific settings based on the environment is probably a good idea. Finally, it's always nice if you can override any specific setting down to the node level, and here you have another good candidate for a file to read.

In short, this means that in a similar setting you need at least four levels:

1. General defaults
2. Location defaults
3. Location environment defaults
4. Node-specific settings

We have found this scheme to work pretty well as long as you don't need to share the first three levels across several different projects. In that case, we found that replicating the first three level into a "project" level gave us a scheme that worked well in this case, too:

1. General defaults
2. Location defaults
3. Location environment defaults
4. Project general defaults
5. Project defaults for a specific location
6. Project defaults for a specific environment (e.g.: production, preproduction, testing, development, integration...)
7. Node-specific settings

Each of these "levels" is mapped to a file name and each file name is added to a list. The fully qualified name of the list is then passed to the `henc` bundle.


### Read the files and get the settings ###

You need to pass the `henc` bundle the fully qualified **name** of a list containing the relative paths to your settings' file, their path will be relative to `$(henc_cfg.master_enc)` for the source files on the policy hub and to `$(henc_cfg.local_enc)` for the local copies of the files. E.g., if the full name of the list is "`classify.enc`", that is: a list called `enc` defined in a bundle called `classify`, you'll read the configuration with a methods promise:

```
  methods:
    any::
      "ENC"
          comment   => "External node classification",
          usebundle => henc("classify.enc") ;
```

Once that bundle is evaluated, all the information in the settings' file will be applied and you can use it straight away.


## Examples ##

### bundle common henc_cfg ###

The following is a very simple example of a configuration for hENC


```
bundle common henc_cfg
# configuration for bundle agent henc; edit the variables in this file
# to suit your needs
{
  vars:
    any::
      "master_modules"
        comment => "Source directory for modules",
        string  => "/var/cfengine/masterfiles/modules" ;

      "local_modules"
        comment => "Local directory for modules",
		string  => "/var/cfengine/modules" ;

      "master_enc"
        comment => "Base directory for ENC files",
        string  => "/var/cfengine/masterfiles/ENC" ;

      "local_enc"
        comment => "Local directory for caching ENC files",
        string  => "/etc/cfengine/ENC" ;
}
```

With these settings and a settings' files list containing, e.g., just one item "default", the file `/var/cfengine/masterfiles/ENC/default` on the policy hub will be copied locally on `/etc/cfengine/ENC/default` and then read by the module. If, for example, we are on an NTP server that needs special settings, the list may contain both "default" and "ntp_server". In that case the file `/var/cfengine/masterfiles/ENC/ntp_server` will also be copied to `/etc/cfengine/ENC/ntp_server`, the `henc` module will read these two files in that order, build a coherent set of classes and variables and hand them to the agent, and you can use them in other parts of the policy. If both files say something about a certain setting, the setting read last is retained and the previous ones discarded -- that's how the hierarchical merging happens in hENC.


### Settings' files ###

The following file will set two global classes: `ntp_client` and `ntp_unicast`, and a list called ntp_servers that will be created in the context `henc` (named after the module's name). The fully qualified name will then be `henc.ntp_servers`:

```
# General settings for all servers

+ntp_client
+ntp_unicast
@ntp_servers={ "ntp1.example.com","ntp2.example.com","ntp3.example.com","ntp4.example.com" }
```

On one of your NTP servers you'll want to override those settings and provide a differenent list of upstreams. You could have this file read adter the previous one:

```
# Settings for ntp1.example.com

_ntp_client
+ntp_server
@ntp_servers={ "no.pool.ntp.org","se.pool.ntp.org","fi.pool.ntp.org","dk.pool.ntp.org" }
```

When henc will read the first file and then this one, it will "lower" the class `ntp_client`: the module will simply forget about it and won't set anything. Notice that if we cancelled it, the agent would have prevented the policy from raising the class again.

The final result in ntp1.example.com would be that the classes `ntp_server` and `ntp_unicast` will be set, and the list `henc.ntp_server` will contain a different list of upstream servers.

## More information ##

* The video of my seminar "[the classification problem: challenges and solutions](https://www.youtube.com/watch?v=7Jbq2DCgf0Y)" from FOSDEM'14 ([slides available](https://speakerdeck.com/brontolinux/the-classification-problems-challenges-and-solutions) on SpeakerDeck)
* The [blog post](http://syslog.me/2014/03/10/the-classification-problem-challenges-and-solutions/) with the same name
* The updated [full slides set](https://speakerdeck.com/brontolinux/the-classification-problem-challenges-and-solutions-ed-2015), presented at this year's [Software 2015](http://sched.co/25zY) in Oslo.
