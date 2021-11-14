# CONTENT

  1. configuration
  2. compile
  3. install
  4. runtime env
  5. troubleshooting

# 1. CONFIGURATION

SKV uses cmake to build libraries and executables. We require cmake
version 2.8.

After downloading the skv source (lets say to $HOME/skv), you create a
build directory (e.g. $HOME/skv_build).

Go into the build directory and start configuring the build:

```
$> cmake ../skv [OPTIONS]
```

We've prepared a few site-specific cmake files in CMake/Site/. These
files set essential paths and config options for a particular site.  A
site could also be just your computer.  You might want to look into
CMake/Site/pers_example.cmake.  This file sets the path to the MPI
compilers.  If you need different settings, you can either modify the
file directly or create your own and then run:

```
$> cmake ../skv -DSKV_SITE:STRING=<sitename>
```

Predefined sitenames are ykt, cscs, pers_example (the name of the file
in CMake/Site/ without the .cmake extension)

Alternatively, you can run:

```
$> ccmake ../skv
```

to start a gui-like cmake configuration tool that allows you to
set/adjust individual variables.  You can set the SKV_SITE there and
configure.

Note on multi-configuration requirements, in case your environment
requires different build configurations (for example if you run
clients and servers on different architectures):

The cmake philosophy separates the build from the source. Therefore,
if you need multiple build configurations, you'd just create a new
build directory and configure the next/separate build in there.

# 2. COMPILE

If configuration was successful, stay in the build directory and run

```
$> gmake
```

to build SKV.

# 3. INSTALL

To install, run:

```
$> gmake install
```

You can also build packages for your Linux distribution by using
cpack. For example:

```
$> cpack -G RPM
```

will create an RPM that can be used to install SKV.

# 4. RUNTIME ENVIRONMENT

## 4.1 prerequisites and dependencies

 Installation and setup of the following is not covered here:

  * any MPI runtime env to start the server (and parallel clients if
    applicable)

    * Ubuntu 20.04

      ```
       $ sudo apt install -y libmpich-dev mpich-doc
      ```

  * SoftIWarp (siw) if you plan to run server and clients on
    overlapping sets of nodes or nodes without InfiniBand or other
    verbs providers installed

  * an OFED-RDMA capable network and setup (alternative siw)

## 4.2 SKV runtime

 * skv_server.conf: this file contains important configuration
   information for server AND client.  The search for the config file
   is performed in the following order:

   * command line parameter for the Server `(-c <path+filename>)`
   * per-user config under `${HOME}/.skv_server.conf`
   * global config under `/etc/skv_server.conf`

   the skv_install script installs an example copy as
   `<SKV_DIR>/etc/skv_server.conf` The file contains descriptions of
   each setting and should be self-explaining

 * running the server:

   * a single server can be run by just executing
     `<SKV_DIR>/bin/SKVServer [-h] [-c <configfile>]`. If you want a
     parallel server you need to use your preferred MPI program
     launcher to kick it off. For example:
   
        `mpiexec -n 8 -f /etc/machinefile <SKV_DIR>/bin/SKVServer`

   * if everything works fine, you'll _not_ get back a prompt. You'll
     see an SKVServer process running and `/tmp/` (as of the defaults in
     `skv_server.conf`) will contain several skv_files as described below

 * skv server files:

   * skv_server.info; skv_machinefile: depending on the settings in
     the config file, the file(s) contains a list of IP addresses and
     port numbers indicating how to connect to the server on THIS
     particular node.  Each server instance replaces its entry with
     the loopback address and can be reached via a proper siw
     installation (this is only if the file is located on a local file
     system, if the configuration points to a shared file system
     between servers, the address is not replaced. Note that this will
     cause problems for clients on BG/Q, because they need to connect
     to the loopback address via siw)

   * skv_store.N: is the mmapped file that contains the stored
     data. Where N is the MPI rank of the server process. Currently,
     all storage memory has to be pinned and thus the capacity per
     node is limited by the amount of available memory and the memlock
     limit of the user (see `ulimit -l` and `/etc/security/limits...`)

   * *.FxLog logfiles of the server. Logging can be configured via the
     make.conf(.in) file in the base directory of the SKV distribution
     by commenting in/out the *_LOG lines.

 * skv client:

   * a client requires the `skv_server.conf` file in one of the places
     described above.  If the client program is written without
     commandline support for the config file, you need to pick one of
     the 2 latter options (`$HOME` or `/etc`).  Every client needs access
     to the server machinefile to learn about the location of the
     servers.  Since the server creates this file, there's no manual
     intervention required if a client runs on a node that has a
     server too.  However, if the client runs on a distinct set of
     nodes, this file has to be created manually or on a shared file
     system (or by copying one of the server-created files and
     replace the lines that have the loopback address with the proper
     IP address).  In every case, it is suggested to consider the
     server-generated machinefile as a template.

   * clients can be either MPI or non-MPI programs.  Depending on
     which library and compiler was used to create the executable.
     The default build of SKV creates and installs MPI- and non-MPI
     versions of the client libraries.

   * a detailed description of the client API will be available
     later. For now, unfortunately we can only point to the source
     code examples and the file `include/client/skv_client.hpp` (for C++
     clients) or `lib/include/skv.h` (for C clients).

# 5. TROUBLESHOOTING

  * Large Scale runs fail because of open file limit exceeded

    * when running SKV at a large scale - i.e. with many many clients,
      it is important to observe the user's limit for open file
      descriptors.  The server needs at least one file desciptor per
      client (unless clients connect via the forwarder option).
