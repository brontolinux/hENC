# hENC

A radically simple hierachical External Node Classifier (ENC) for CFEngine

With just 60 lines of Perl to read and merge settings from plain text files, hENC is probably the simplest external node classifier for CFEngine on the planet. I doubt you can find anything as simple and as flexible and powerful (but if you do please share it because I want to use it!).

Since the henc module is written in Perl, you must have a working Perl installation on any machine where you want to use it. If you forget about that, you'll probably get an error.

## hENC components ##

The hENC system for CFEngine revolves around the **henc** module, with these surrounding components:

- your text files with classes and variables to represent the different systems configurations in your infrastructure
- a bundle agent henc defined in [enc.cf](module/enc.cf) which does all the file copying and ensures the file list before handing off to the henc module
- a configuration file [henc_cfg.cf](lib/henc_cfg.cf) which describes the source (on the hub) and  destination (on the client) directories to be used
- a library [henc_lib.cf](lib/henc_lib.cf) with two helper bodies which manage the compare/copy operations and the module execution permissions
- the henc module itself, which parses all of the plain text files and assembles the appropriate list of settings

The text files, whose format is based on CFEngine's module protocol and is explained below, are read by the hENC system to set/cancel classes and variables that represent your infrastructure.

You then configure your promises library to execute dynamically depending on the specific configurations of classes and variables that result from each separate execution of the hENC system.  


## Does it work on my CFEngine installation? ##

Clone the repository, get into the directory and run

`sudo -u root cf-agent -Kf ./henc_test.cf`

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

If you run into problems/bugs you can open an issue here.

If fix any problems/bugs then please fork the project and make a pull request so that I can incorporate your changes.


## How do I install hENC on my system? ##

These instructions assume you are using the standard configuration, detailed in henc_cfg.cf

On the policy hub, proceed as follows:

1. create a masterfiles/ENC directory and place your *plain text* settings files here
2. copy module/henc into a masterfiles/modules directory (create the modules dir if you don't have one already)
3. copy module/enc.cf into the masterfiles directory
4. copy lib/henc_lib.cf into the masterfiles directory (or include its content in your site library, if you have one)
5. copy lib/henc_cfg.cf into the masterfiles directory (or include its content in your site library, if you have one)
6. ensure that any .cf file you have added in the previous steps are listed as inputs in masterfiles/promises.cf -- hENC will not do this for you.

The hENC system is now ready to be used, but nothing will happen if you don't actually use it. Please also note that henc only sets/cancels classes and sets variables and then it's up to you to use them -- it won't automatically run a bundle for you, for example.


### Advanced configuration ###

* if you have a site library then store the contents of henc_lib.cf (and henc_cfg.cf if you so wish) in there instead of your separate files in the masterfiles directory;
* if the version of CFEngine you're using supports it, you can list your .cf files elsewhere in a `body files control` bundle
* the henc_cfg.cf file (or its content) can be modified according to your needs:
    1. `master_modules` should be set to the directory on the policy hub where you keep your modules;
    2. `local_modules` should be set to the directory on end nodes where you keep a local copy of the modules;
    3. `master_enc` should be set to the directory on the policy hub where you will store your ENC files;
    4. `local_enc` should be set to the directory on end nodes where you will keep a local copy of the ENC files;

## hENC file format ##

The text files use a subset of CFEngine's module protocol plus a couple of additions. Anything else in the file is ignored.

Four "primitives" of the module protocol are used in hENC files: `+` to set a class, `-` to cancel a class, `=` to set a scalar variable, `@` to set a list variables and `%` to set a data container variable in JSON format. The classes will be global in scope, while the variables will be defined in the `henc` context, e.g.: `$(henc.myvar)`, `@(henc.mylist)`.

In addition, we added two primitives:
* `_` will "lower" a class: the module will forget whatever it knew about that class until then;
* `/` will "slash" a variable (list or scalar): the module will forget whatever it knew about that variable until then.

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

As long as you don't need to share the first three levels across several different projects, this scheme works well. If you need to support more than one project, we found that replicating the first three "generic" levels into three additional project-specific levels works as well:

1. General defaults
2. Location defaults
3. Location environment defaults
4. Project general defaults
5. Project defaults for a specific location
6. Project defaults for a specific environment (e.g.: production, preproduction, testing, development, integration...)
7. Node-specific settings

Each of these "levels" is mapped to a file name and each file name is added to a list. The fully qualified name of the list is then passed to the `henc` bundle.


### Read the files and get the settings ###

You need to pass the `henc` bundle the fully qualified **name** of a list containing the relative paths to your settings' file. The paths to these text files will be automatically constructed relative to `$(henc_cfg.master_enc)` for the source files on the policy hub and to `$(henc_cfg.local_enc)` for the local copies of the files -- you do not need to (and should not) specify full paths in the list.

### Example ###

If the full name of the list is "`classify.enc`", that is: a list called `enc` defined in a bundle called `classify`, you'll read the configuration with a methods promise:

```
bundle agent classify {
	vars:
		"enc"
			slist 		=> 		{ "henc.default" };


  methods:
    any::
      "ENC"
          comment   => "External node classification",
          usebundle => henc("classify.enc") ;
}
```

Once the bundle is evaluated, all the information in the settings' file will be applied and you can use the classes and variables from it's results straight away. Notice that the variables will be set in the henc context: if, for example, you defined a variable called "foo", you'll refer to it as `$(henc.foo)`.


## Sample files ##

The sample files in the `examples/ajoslin` directory and the explanation below are courtesy of [Allen Joslin](https://github.com/ajoslin103).


The following is a very simple example of a usage of hENC.

### masterfiles/ENC/henc.default ###

For the purpose of this sample we will use one text file `ENC/henc.default` which will raise a class `updateMOTD` that we will use to trigger an update of the `/etc/motd` file, whose contents are displayed when starting a login sessions on a *NIX system.

```
+updateMOTD
```

### masterfiles/managed.cf ###

The bundle agent managed in this file will modify the `/etc/motd` file to include the line: "This system is being actively managed by CFEngine"

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

The bundle agent sample in this file  will create and hand a simple list to the henc module and also apply a promise that will evaluate bundle agent managed that, in turn, will update the `/etc/motd` file. 

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

This bundle contains the configuration of hENC for this sample

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

When the installation of hENC and these files is completed on the policy hub you can run the CFEngine agent on the agent and ask it to apply the `sample` bundle. The course of action for the agent will be:

- the bundle agent sample from sample.cf will build a file list and pass it to bundle agent henc
- bundle agent henc will copy or update the henc module and the text file on the client (where the files will be looked up on the hub and where they will be copied on the client is described in the config file; the way the copy will happen and the permissions applied are described in the library)
- bundle agent henc will run the henc module, the module will read the text file and raise the class `updateMOTD`, as requested by using the '+' sign
- bundle agent sample notices that the class `upateMOTD` is defined and will evaluate bundle agent managed, thus ensuring that the `/etc/motd` file matches the given description.

## A more complex example ##

The real power of hENC appears when you start using a hierarchy of settings files instead of just one. You can build a list of files based on any criteria that suits your case and have the settings in the different files merged together.

As an example, let's say that you want to apply a certain configuration for the NTP service on all clients, but you'll also want to apply a different configuration on your NTP servers. An easy way to do it is to have a general default file read by all machines, and an additional file read on servers so that some of the clients' settings are overridden with server settings.

The following file will set two global classes: `ntp_client` and `ntp_unicast`, and a list called ntp_servers that will be created in the context `henc` (named after the module's name). The fully qualified name will then be `henc.ntp_servers`:

```
# General settings for all servers

+ntp_client
+ntp_unicast
@ntp_servers={ "ntp1.example.com","ntp2.example.com","ntp3.example.com","ntp4.example.com" }
```

On one of your NTP servers you'll want to override those settings and provide a different list of upstreams. You could have this file read adter the previous one:

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
