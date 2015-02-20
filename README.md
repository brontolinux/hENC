# hENC

A hierachical External Node Classifier (ENC) for CFEngine

**This is work in progress and not yet ready for release**

This ENC system for CFEngine is based on the following components:

- a CFEngine module, you'll find it in module/henc
- a configuration for the module, in lib/henc_cfg.cf
- a CFEngine agent bundle to run the module, you'll find it in module/enc.cf
- a few CFEngine bodies used in enc.cf, in lib/henc_lib.cf


Not included in the distribution are text files, whose format is based on CFEngine's module protocol, that are read by the module to set/cancel classes and variables. Of course, you do want to create these files by yourself.

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

