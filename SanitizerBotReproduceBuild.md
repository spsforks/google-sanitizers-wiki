# How to reproduce build from a Sanitizer Bot

## Requirements 
You need workstation similar to buildbot, details you can find on the bot details page, e.g. [sanitizer-buildbot1](http://lab.llvm.org:8011/buildslaves/sanitizer-buildbot1)
Scripts probably will work only on x86_64 Linux, still a system does not need to be exactly the same.

Also here is [setup script](https://github.com/google/sanitizers/blob/master/buildbot/start_script.sh) for bots, which can be used as a hint to install missing packages. DON'T RUN the this script itself. It's supposed to be used by our GCE instances.

Then you need to decide which script to run. To find the script name corresponding to the bot use [this mapping](https://llvm.org/svn/llvm-project/zorg/trunk/zorg/buildbot/builders/sanitizers/buildbot_selector.py).

## Example
```
mkdir scratch_dir
cd scratch_dir
svn checkout https://llvm.org/svn/llvm-project/zorg
BUILDBOT_CLOBBER= BUILDBOT_REVISION=300000 zorg/trunk/zorg/buildbot/builders/sanitizers/buildbot_fast.sh
```
The script will checkout, build and test code very closely to how it does on bots.

## Try local changes
Scripts support environment variable BUILDBOT_MONO_REPO_PATH
e.g. 
```
BUILDBOT_MONO_REPO_PATH=~/src/llvm.git/llvm-project BUILDBOT_CLOBBER= BUILDBOT_REVISION= \
zorg/trunk/zorg/buildbot/builders/sanitizers/buildbot_fast.sh
```
Variable will make the script to rsync sources from your LLVM git mono-repo checkout into the scratch dir, instead of checking out them from svn.