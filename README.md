# hENC

####A radically simple hierachical External Node Classification (ENC) system for CFEngine####

With just 60 lines of Perl to read and merge settings from plain text files, hENC is probably the simplest external node classifier for CFEngine on the planet. I doubt you can find anything as simple and as flexible and powerful (but if you do please share it because I want to use it!).

The hENC system for CFEngine revolves around the **henc** module, with these surrounding components:

- your text files with classes and variables to represent the different systems configurations in your infrastructure
- an agent bundle [enc.cf] which does all the file copying and ensures the file list before handing off to the henc module
- a configuration [henc_cfg.cf] file which describes the [hub] source and [client] destination directories to be used
- a library [henc_lib.cf] which manages the compare/copy operations and the module execution permissions
- the henc module itself, which parses all of the plain text files and assembles the appropriate list of settings

The text files, whose format is explained below, are read by the hENC system to set/cancel classes and variables that represent your infrastructure. 

You then configure your promises library to execute dynamically depending on the specific configurations of classes and variables that result from each separate execution of the hENC system.  


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

Then congratulations, you can use hENC on your system.

If the last line looks like this instead:

```
R: FINAL RESULT: Some tests failed :-(
```

If you run into problems/bugs you can open an issue here.  

If fix any problems/bugs then please fork the project and make a pull request so that I can incorporate your changes.


## How do I install hENC on my system? ##

(these instructions assume you are using the standard configuration, detailed in henc_cfg.cf)

1. create a masterfiles/ENC directory and place your [plain text] settings files here
2. copy module/henc into a masterfiles/modules directory (create the modules dir if you don't have one already)
3. copy module/enc.cf into the masterfiles directory
4. copy lib/henc_lib.cf into the masterfiles directory (or include its content in your site library, if you have one)
5. copy lib/henc_cfg.cf into the masterfiles directory (or include its content in your site library, if you have one)
6. ensure that all of your .cf files are listed as inputs in masterfiles/promises.cf -- hENC will not do this for you.

Remember: hENC will not exe: henc 
sets/cancels classes and sets variables, that's it. It's up to you to 
make good use of them. 

These options are for more advanced usage:

* if you have a site library then store the henc_lib.cf & henc_cfg.cf files in there instea of your masterfiles directory
* if the version of CFEngine you're using supports it, you can list your .cf files elsewhere in a `body files control` bundle
* the henc_cfg.cf file can be modified according to your needs:
    1. `master_modules` should be set to the directory on the policy hub where you keep your modules;
    2. `local_modules` should be set to the directory on end nodes where you keep a local copy of the modules;
    3. `master_enc` should be set to the directory on the policy hub where you will store your ENC files;
    4. `local_enc` should be set to the directory on end nodes where you will keep a local copy of the ENC files;

Final Note: the henc module is written in perl, so you must have perl installed on any machine that will try to execute it.  If the henc module fails with an error [1] 


## hENC file format ##

The text files use a subset of CFEngine's module protocol plus a couple of additions. Anything else in the file is ignored.

Four "primitives" of the module protocol are used in hENC files: `+` to set a class, `-` to cancel a class, `=` to set a scalar variable and `@` to set a list variables. The classes will be global in scope, while the variables will be defined in the `henc` context, e.g.: `$(henc.myvar)`, `@(henc.mylist)`.

In addition, we added two primitives:
* `_` will "lower" a class: the module will forget whatever it knew about that class until then;
* `/` will "slash" a variable (list or scalar): the module will forget whatever it knew about that variable until then.

Data (JSON) containers (the `%` primitive in the module protocol) is not yet implemented because we haven't completed the transition to CFEngine 3.6.x.  Please feel free to fortk the project and contribute an implementation supporting data containers if you can!

Note that in this version **Trailing comments or continuation lines are not allowed** as the format is line based.


## How does hENC work? ##

You pass a list of specially formatted text files to a bundle, the bundle runs a module that reads the files, merges the information in them to remove conflicts, and sets or unsets classes and variables.

hENC will read the files in the order they are given in the list, build a coherent set of classes and variables and hand them to the agent.  If more than one file controls the same class or variable, the setting read last is retained and the previous ones discarded.


### Build a list of settings' files ###

The most important thing to do is deciding how you will build the list of files containing the settings you want to apply to your systems. It's the most important thing, the rest is just "mechanics".

The system is flexible enough to allow for both static and dynamic lists, or anything in between. Actually, hENC doesn't really care how you build your file list: all it asks is to be passed the fully qualified name of the list itself. The following are just suggestions based on our own experience.

We use configuration management on several datacenters around the world. In each datacenter, there may be different "environments" that require specific settings, for example: machines confined in an "isolated" network segment will probably use different DNS servers than the rest, so having a file to map specific settings based on the environment is probably a good idea. Finally, it's always nice if you can override any specific setting down to the node level, and here you have another good candidate for a file to read.

We have found a scheme of four levels to be sufficient for most needs:

1. General defaults
2. Location defaults
3. Location environment defaults
4. Node-specific settings

As long as you don't need to share the first three levels across several different projects it works well. In that case, we found that replicating the first three level into a "project" level gave us a scheme that worked well:

1. General defaults
2. Location defaults
3. Location environment defaults
4. Project general defaults
5. Project defaults for a specific location
6. Project defaults for a specific environment (e.g.: production, preproduction, testing, development, integration...)
7. Node-specific settings

Each of these "levels" is mapped to a file name and each file name is added to a list. The fully qualified name of the list is then passed to the `henc` bundle.


### Read the files and get the settings ###

You need to pass the `henc` bundle the fully qualified **name** of a list containing the relative paths to your settings' file, E.g., if the full name of the list is "`classify.enc`", that is: a list called `enc` defined in a bundle called `classify`, you'll read the configuration with a methods promise:

```
bundle agent classify {
	vars:
		"enc"
			slist 		=> 		{ "henc.default" };
	methods:
		any::
			"ENC"
				usebundle => henc("classify.enc");
}
```

Once the bundle is evaluated, all the information in the settings' file will be applied and you can use the classes and variables from it's results straight away.

The paths to these text files will be automatically constructed relative to `$(henc_cfg.master_enc)` for the source files on the policy hub and to `$(henc_cfg.local_enc)` for the local copies of the files -- you do not need to (and should not) specify these paths. 


## Examples ##


The following is a very simple example of a usage of hENC (the files are in ./examples/one

*(Note that some of the hENC files are symlinks, git will return them as text files containing pathnames of the targets on those systems that do not support symlinks.)*

### masterfiles/ENC/henc.default ###

For purposes of this sample we will use one text file [ENC/henc.default] which will raise a class [updateMOTD] that we will use to trigger an updated of the /etc/motd file whose contents are displayed to [*nix] terminal login sessions.

```
+updateMOTD
```

### masterfiles/managed.cf ###

This bundle will modify the /etc/motd file to include the line: "This system is being actively managed by CFEngine"

```
bundle agent managed {
	vars:
		"motd" string => "/etc/motd";
	files:
		"$(motd)"
			create => "true",
			edit_line => addmessage;
}

bundle edit_line addmessage {
	delete_lines:
		".*CFEngine.*";
	insert_lines:
		"This system is being actively managed by CFEngine";
}
```

### masterfiles/sample.cf ###

For purposes of this demo we will do both: create and hand a simple list to the henc module; and also evaluate a promise that will execute the managed.cf bundle that will update the /etc/motd file. 

More complex and indirect usages of hENC would not combine these actions.

```
bundle agent sample {
	vars:
		"base"
			comment     =>      "all systems use these defaults",
			policy		=>		"free",
			slist 		=> 		{ "default" };
	methods:
		any::
			"ENC"
				comment   => "External node classification",
				usebundle => henc("sample.base");
		updateMOTD::
			"updateMOTD" usebundle => managed;
}
```


### bundle common henc_cfg ###

These is the configuration for this sample

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

### the resulting actions ###

- the sample.cf policy built and passed a list to the henc agent

- the text file raises the class updateMOTD using the '+' sign
- the henc agent copies the text file to the agents and builds a list with that one class 
- (the config file described where the file was to be copied)
- (the library managed the compare/copy operations and ensured the henc modules execution permissions)
- the henc module parsed that single file and defined the class updateMOTD 

- the sample.cf bundle saw the defined class upateMOTD and executed the managed.cf policy
- 

## a more involved example ##

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
* The updated [full slide set](https://speakerdeck.com/brontolinux/the-classification-problem-challenges-and-solutions-ed-2015), presented at this year's [Software](http://sched.co/25zY) conference in Oslo.
