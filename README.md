# hENC

A hierachical External Node Classifier (ENC) for CFEngine

**This is work in progress and not yet ready for release**

This ENC system for CFEngine is based on the following components:

- a CFEngine module, you'll find it in module/henc
- a configuration for the module, in lib/henc_cfg.cf
- a CFEngine agent bundle to run the module, you'll find it in module/enc.cf
- a few CFEngine bodies used in enc.cf, in lib/henc_lib.cf


Not included in the distribution are text files, whose format is based on CFEngine's module protocol, that are read by the module to set/cancel classes and variables. Of course, you do want to create these files by yourself.

## How does hENC work? ##

You pass a list of specially formatted text files to a bundle, the bundle runs a module that reads the files, merges the information in them to remove conflicts, and sets or unsets classes and variables.

The text files use a subset of CFEngine's module protocol plus a couple of additions. Anything else in the file is ignored.


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

### Example ###

The following file contains the settings that apply to all systems... almost.

```
# General settings for all servers

+ntp_client
+ntp_unicast
@ntp_servers={ "ntp1.example.com","ntp2.example.com","ntp3.example.com","ntp4.example.com" }
```

However, on one of your NTP servers you'll want to read this file, too:

```
# Settings for ntp1.example.com

_ntp_client
+ntp_server
@ntp_servers={ "no.pool.ntp.org","se.pool.ntp.org","fi.pool.ntp.org","dk.pool.ntp.org" }
```

Notice how the class `ntp_client` was lowered instead of cancelled: hENC will simply forget about it and won't set anything. If we cancelled it, the agent would have prevented the policy from raising the class again.

## How do I use hENC? ##

You need to pass the `henc` bundle the fully qualified **name** of a list; the list will contain the files to be read and their path will be relative to `$(henc_cfg.master_enc)` for the source files on the policy hub and to `$(henc_cfg.local_enc)` for the local file. For example, if you have this code:

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

bundle agent classify
{
  vars:
    any::
      "enclist" slist => { "default" }

    ntp_server::
      "enclist" slist => { @(enclist), "ntp_server" } ;


  methods:
      "ENC" usebundle => henc("classify.enc") ;
}
```

the file `/var/cfengine/masterfiles/ENC/default` on the policy hub will be copied locally on `/etc/cfengine/ENC/default`; similarly, if the `ntp_server` class is defined the file `/var/cfengine/masterfiles/ENC/ntp_server` will be copied on `/etc/cfengine/ENC/ntp_server`. Finally, the `henc` module will read these two files in that order, build a coherent set of classes and variables and hand them to the agent, and you can use them in other parts of the policy. If both files say something about a certain setting, the setting read last is retained and the previous ones discarded -- that's how the hierarchical merging happens in hENC.



