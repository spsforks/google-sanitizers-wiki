# How to reproduce build from a Sanitizer Bot

## Requirements 
You need workstation similar to buildbot, details you can find on the bot details page, e.g. [sanitizer-buildbot1](https://lab.llvm.org/buildbot/#/builders/sanitizer-x86_64-linux-fast)

Scripts probably will work only on x86_64 Linux, still a system does not need to be exactly the same.

*OPTIONAL*: Also here is [setup script](https://github.com/google/sanitizers/blob/master/buildbot/start_script.sh) for bots, which can be used as a hint to install missing packages. **DON'T RUN** the this script itself. It's supposed to be used by our GCE instances.

Then you need to decide which script to run. To find the script name corresponding to the bot use [this mapping](https://github.com/llvm/llvm-zorg/blob/master/zorg/buildbot/builders/sanitizers/buildbot_selector.py).

**WARNING**: The script can delete content of the current directory!

## Example
```
mkdir scratch_dir
cd scratch_dir
git clone https://github.com/llvm/llvm-zorg.git
BUILDBOT_CLOBBER= BUILDBOT_REVISION=ba3f863dfb9c5f9bf5e6fdca2198b609df3b7761 llvm-zorg/zorg/buildbot/builders/sanitizers/buildbot_fast.sh
```
The script will checkout, build and test code very closely to how it does on bots.

## Try local changes
Scripts support environment variable BUILDBOT_MONO_REPO_PATH
e.g. 
```
BUILDBOT_MONO_REPO_PATH=~/src/llvm.git/llvm-project BUILDBOT_CLOBBER= BUILDBOT_REVISION= \
llvm-zorg/zorg/buildbot/builders/sanitizers/buildbot_fast.sh
```
BUILDBOT_MONO_REPO_PATH makes the script to use your LLVM checkout instead of cloning it from github.