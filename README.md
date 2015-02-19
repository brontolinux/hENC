# hENC

A hierachical External Node Classifier (ENC) for CFEngine

**This is work in progress and not yet ready for release**

This ENC system for CFEngine is based on the following components:

- a CFEngine module, you'll find it in module/henc
- a configuration for the module, in lib/henc_cfg.cf
- a CFEngine agent bundle to run the module, you'll find it in module/enc.cf
- a few CFEngine bodies used in enc.cf, in lib/henc_lib.cf


Not included in the distribution are text files, whose format is based on CFEngine's module protocol, that are read by the module to set/cancel classes and variables. Of course, you do want to create these files by yourself.

