
# Installation of the Open Ephys GUI on a 32-bit or 64-bit Debian machine

These steps are correct as of Aug 4, 2016, in particular for the 0.4.1 version of the plugin-GUI.

Also note that if you're on a new version of Ubuntu, you don't need to compile the HDF5 library by hand, and you can follow [these instruction](https://github.com/margoliashlab/ArfFormat) to make the software. However, you still need to perform the last steps of this guide (grabbing the .bit file and making sure acquisition board is recognized - this seems to be especially important on Ubuntu).

First check if the machine is 32 or 64 bit.

    uname -m

If the machine is 64 bit, you can download the latest binary with:

    https://github.com/open-ephys-GUI-binaries/linux-64/archive/master.zip
    
Though the instructions recommend compiling from source for some reason. Otherwise follow the compile instructions below.

They were pieced together from https://open-ephys.atlassian.net/wiki/display/OEW/Linux and other sources of information.

You will need to build the program from source, as compiled binaries currently only exist for 64-bit Linux. The steps are:
1. Obtain the source code
2. Install the ZeroMQ library
3. Install the latest HDF5 library version
4. Install the Open Ephys GUI


## Install the ZeroMQ library.

First, add the PPA to the package manager’s sources list:

    sudo add-apt-repository ppa:chris-lea/zeromq

Because PPAs are a Ubuntu-specific feature, you need to modify the appropriate source list file to make it look like you’re updating from a machine running Ubuntu. The file (assuming you’re running Debian 7.0 ‘wheezy’) is

    /etc/apt/sources.list.d/chris-lea-zeromq-wheezy.list

This file will contain the following two lines:

    deb http://ppa.launchpad.net/chris-lea/zeromq/ubuntu wheezy main
    deb-src http://ppa-launchpad.net/chris-lea/zeromq/ubuntu wheezy main

You need to change both instances of ‘wheezy’ to the name of a valid Ubuntu version. I used ‘trusty’. No other changes are needed. (The name of the .list file itself, funnily enough, doesn’t need to be changed.)

Now update the package list information

    sudo apt-get update

Now you can install the libraries themselves. For the version of the Open Ephys GUI current at the time of this writing, you need version 3 of the ZeroMQ API. This is specified in the command below:

    sudo apt-get install libzmq3-dbg libzmq3-dev libzmq3

### Wheezy (Debian 7) and libc
If you get an error about the libc version being too old in the repositories, you can do a partial upgrade to Jessie (Debian 8). Do this by editing `/etc/apt/sources.list` and replacing all instances of `wheezy` with jessie. Then `sudo apt-get update` and `sudo apt-get upgrade`.

## Install the HDF5 library

The version of the Open Ephys GUI current as of this writing requires a version of the HDF5 library higher than 1.8.11. Currently, updating the HDF5 library using apt-get on Debian only updates to 1.8.8.9, so you’ll need to build this library from source.

Download link for source: http://www.hdfgroup.org/HDF5/release/obtainsrc.html

If you want to be able to keep running jill software, it is a good idea to only install this later version of the HDF5 library locally instead of globally. This will prevent this later version from interfering with the library bindings jill modules use to create .arf files. (This may not be an issue, but I wasn’t about to risk it the first time.)

Place the source tarball in a folder (not /usr/bin! /usr/local might be ok. I installed it in a folder called ‘hdf5-current’ in my home folder) of your choice and extract it (if tar complains that it’s zipped, add ‘z’ to the ‘-xf’):

    tar -xf [tarball filename]

cd to the new extracted directory and run the configure script. For the ‘--prefix’ argument, provide the directory containing the tarball and extracted source code directory. Make sure to include the ‘--enable-cxx’ flag, or make will only install the C bindings:

    ./configure --prefix=[wherever you installed it] --enable-cxx

Now there are four make invocations, each one of which may take some time (tens of minutes in total, in my case):

    make
    make check
    make install
    make check-install

You may need superuser privileges to `make install` globally. The last thing to do is to make sure the current session’s environment variables contain the appropriate paths to this new HDF5 library. You need to set three environment variables:

    export CPLUS_INCLUDE_PATH=[path to hdf5 directory]/include
    export LIBRARY_PATH=[path to hdf5 directory]/lib
    export LD_LIBRARY_PATH=[path to hdf5 directory]/lib


## Install the Open Ephys GUI from source

Obtain the source code.

    git clone https://github.com/open-ephys/plugin-GUI.git
    cd plugin-GUI

Run the dependency installation script in the GUI/Resources/Scripts folder:

    sudo ./install_linux_dependencies.sh

Then copy the device interface rules file (in the same directory as above) to the system’s udev (not dev) folder and refresh the udev service:

    sudo cp 40-open-ephys.rules /etc/udev/rules.d
    sudo service udev restart

cd to GUI/Builds/Linux and run make

    make -j4

Then install the plugins

    make -j4 -f Makefile.plugins

The installation should go smoothly if you’ve followed the above instructions. The binary to run the GUI is called ‘open-ephys’ and resides in GUI/Builds/Linux/build.

## Running the Open Ephys GUI

The last wrinkle involves those environment variables you set above. If you close your current session and restart it, they won’t be set any more. Setting them to be updated at login isn’t a great idea, because they could interfere with jill modules’ access to the older HDF5 library. To get around this, the easiest thing to do is write a short shell script that sets them and then starts the Open Ephys GUI in the same fork. You would then use this script to start the GUI, much like how we use Kyler’s intan-interface.sh script.

You can copy the script I wrote from my directory on modeln, or you can copy it in its entirety here (with appropriate modification of paths for your installation):

    #! /bin/bash
    
    export CPLUS_INCLUDE_PATH=[path to hdf5 directory]/include
    export LIBRARY_PATH=[path to hdf5 directory]/lib
    export LD_LIBRARY_PATH=[path to hdf5 directory]/lib
    
    [path to open-ephys folder]/plugin-GUI/Builds/Linux/build/open-ephys

Don’t forget to chmod it to allow execution:

    chmod a+x open-ephys.sh

## Grab the Intan EVAL Rhythm .bit file

Open Ephys has their own special board instead of the EVAL board. To use the EVAL board ADCs, you must copy the main.bit from the Intan source and replace the RHD2000.bit that comes with the open-ephys source code.
This looks something like this:

    cd [path to open-ephys folder]/plugin-GUI/Builds/Linux/build
    mv rhd2000.bit rhd2000.bit.backup
    cp [path to intan source]/main.bit rhd2000.bit
    

## Appendix A: Some Reading for the Curious or Desperate

The build instructions from the Open Ephys devs are linked at the top of this document. Further information I found helpful in creating this guide can be found at the following links. These may prove helpful if you run into trouble and can’t solve it immediately.

* http://zeromq.org/intro:get-the-software
* http://www.hdfgroup.org/HDF5/release/obtainsrc.html (in particular, the INSTALL file, which can also be viewed within the source code directory)
* http://www.webupd8.org/2014/10/how-to-add-launchpad-ppas-in-debian-via.html
* https://wiki.debian.org/SourcesList
