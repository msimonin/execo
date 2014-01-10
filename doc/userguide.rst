****************
execo user guide
****************

In this user guide, the code can be executed from python source files,
but it can also be run interactively in a python shell, such as
``ipython``, which is very convenient to inspect the execo objects.

Installation
============

Prerequisites: you need (debian package names, adapt for other
distributions): ``make``, ``python`` (>= 2.6), ``python-httplib2`` and
optionnaly ``python-keyring``. You also need ``ssh`` and optionnaly
``taktuk``.

In this tutorial it is shown how to install execo in subdirectory
``.local/`` of your home, allowing installation on computers where you
are not root or when you don't want to mix manually installed packages
with packages managed by your distribution package manager.

Install from a release tar.gz package::

 $ wget http://execo.gforge.inria.fr/downloads/execo-2.1.tar.gz
 $ tar xzf execo-2.1.tar.gz
 $ cd execo-2.1/
 $ make install PREFIX=$HOME/.local

Or install from source repository if you want the very latest
version::

 $ git clone git://scm.gforge.inria.fr/execo/execo.git
 $ cd execo
 $ make install PREFIX=$HOME/.local

To add ``~/.local/bin`` to ``PATH`` (i assume bash shell here, adapt
for other shells)::

 $ export PATH="$HOME/.local/bin${PATH:+:${PATH}}"

You can put this line in your ``~/.profile`` to have your environment
setup automatically in all shells.

execo
=====

Core module. Handles launching of several operating system level
processes in parallel and controlling them *asynchronously*.  Handles
remote executions and file copies with ssh/scp and taktuk.

- Standalone processes: `execo.process.Process`, `execo.process.SshProcess`

- Parallel processes: `execo.action.Action`

Processes
---------

- `execo.process.Process`: abstraction of an operating system
  process. Fine grain asynchronous lifecycle handling:

  - start, wait, kill

  - stdout, stderr, error, pid, exit code

  - start date, end date

  - timeout handling

  - callbacks

  - shell; pty

- `execo.process.SshProcess`: Same thing but through ssh. Additional
  parameter: Host, ConnectionParams

  - Host: abstraction of a remote host: address, user, keyfile, port

  - ConnectionParams: connection parameters, ssh options, ssh path,
    keyfile, port, user, etc.

Process examples
................

Local process
'''''''''''''

List all files in the root directory::

 from execo import *
 process = Process("ls /")
 process.run()
 print "process:\n%s" + str(process)
 print "process stdout:\n" + process.stdout
 print "process stderr:\n" + process.stderr

The ``ls`` process was directly spawned, not using a subshell. Set
process property ``shell`` to True if a full shell environment
is needed (e.g. to expand environment variables or to use pipes). To
find all files in /tmp belonging to me::

 process = Process("find /tmp -user $USERNAME")
 process.shell = True
 process.run()

Here a warning log was probably displayed, because if you are not
root, there are probably some directories in ``/tmp`` that ``find``
could not visit (lack of permissions), ``find`` does not return 0 in
this case. The default behavior of execo is to issue warning logs when
processes are in error, do not return 0, or timeout. If needed we can
instruct execo to stop logging the exit code by setting process property
``nolog_exit_code`` to True. In this case, no log will be issued
but the process will still be considered in error from execo's point
of view. You can also instruct execo to ignore the exit code by
setting process property ``ignore_exit_code`` to True.

Remote process over ssh
'''''''''''''''''''''''

On one host *host1* Start an `execo.process.SshProcess` *process_A*
running a listening netcat, then wait 1 seconds, then on another host
*host2* start an `execo.process.SshProcess` *process_B* running netcat
sender, then wait wait for *process_B* termination, then kill
*process_A*::

 from execo import *
 process_A = SshProcess("nc -l -p 6543", "<host1>")
 process_B = SshProcess("echo 'hi there!' | nc -q 0 <host1> 6543", "<host2>")
 process_A.start()
 sleep(1)
 process_B.run()
 process_A.wait()
 print process_A.stdout

We sleep for 1 second after starting the servers to make sure that
they are ready to receive incoming connections.

The netcat option ``-i 1`` is important in this example because as
process_A and process_B are started almost simultaneously, we want to
introduce a little delay (1 second) before process_B tries to connect
to process_A, to be sure that process_A has finished its
initialization and is ready to receive incoming connections.

This example shows the asynchronous control of processes: while a
process is running (the netcat receiver), the code can do something
else (run the netcat sender), and later get back control of the first
process, waiting for it (it could also kill it).

Actions
-------

- `execo.action.Action`: abstraction of a set of parallel
  Process. Asynchronous lifecycle handling:

  - start, kill, wait

  - access to individual Process

  - callbacks

  - timeout

  - errors

- `execo.action.Local`: A set of parallel local Process

- `execo.action.Remote`: A set of parallel remote SshProcess

- `execo.action.TaktukRemote`: Same as Remote but using taktuk instead
  of plain ssh

- `execo.action.Put`, `execo.action.Get`: send files or get files in
  parallel to/from remote hosts

- `execo.action.TaktukPut`, `execo.action.TaktukGet`: same using
  taktuk

- `execo.report.Report`: aggregates the results of several Action and
  pretty-prints summary reports

Remote example
..............

Run a netcat client and server simultaneously on two hosts, to
generate traffic in both directions::

 from execo import *
 hosts = [ "<host1>", "<host2>" ]
 targets = list(reversed(hosts))
 servers = Remote("nc -l -p 6543 > /dev/null", hosts)
 clients = Remote("dd if=/dev/zero bs=50000 count=125 | nc -q 0 {{targets}} 6543", hosts)
 servers.start()
 sleep(1)
 clients.run()
 servers.wait()
 print Report([ servers, clients ]).to_string()
 for s in servers.processes + clients.processes:
   print "%s\nstdout:\n%s\nstderr:\n%s" % (s, s.stdout, s.stderr)

The netcat command line on clients shows the usage of *substitutions*:
In the command line given for Remote and in pathes given to Get, Put,
patterns are automatically substituted:

- all occurences of the literal string ``{{{host}}}`` are substituted by
  the address of the Host to which execo connects to.

- all occurences of ``{{<expression>}}`` are substituted in the
  following way: ``<expression>`` must be a python expression, which
  will be evaluated in the context (globals and locals) where the
  expression is declared, and which must return a
  sequence. ``{{<expression>}}`` will be replaced by
  ``<expression>[index % len(<expression>)]``. In short, it is a
  mapping between the sequence of command lines run on the hosts and
  the sequence ``<expression>``. See :ref:`execo-substitutions`.

execo_g5k
=========

A layer built on top of execo. It's purpose is to provide a convenient
API to use Grid5000 services:

- oar

  - oarsub, oardel

  - get current oar jobs

  - wait oar job start, get oar job nodes

- oargrid

  - oargridsub, oargriddel

  - get current oargrid jobs

  - wait oargrid job start, get oargrid job nodes

- kadeploy3

  - kadeploy: basic deployment

  - deploy: clever kadeploy: automatically avoids to deploy already
    deployed nodes, handles retries on top of kadeploy, callbacks to
    allow dynamically deciding when we have enough nodes (even for
    complex topologies)

- Grid5000 API:

  - list hosts, clusters, sites

  - get the cluster of a host, the site of a cluster

  - get API attributes from hosts, clusters, sites

  - all of this in a secure way: even when used from outside Grid5000,
    there is no need to put Grid5000 API password in clear in scripts,
    password can be stored in the desktop environment keyring if
    available.

To use execo on grid5000, you need to install it inside grid5000, for
example on a frontend. execo dependencies are installed on grid5000
frontends. (Note: to use execo_g5k from outside Grid5000, see
:ref:`tutorial-configuration`)

oarsub example
--------------

Run iperf servers on a group of 4 hosts on one cluster, and iperf
clients on a group of 4 hosts on another cluster. Each client targets
a different server. We get nodes with an OAR submissions, and delete
the OAR job afterwards::

 from execo import *
 from execo_g5k import *
 import itertools
 jobs = oarsub([
   ( OarSubmission(resources = "/cluster=2/nodes=4"), "nancy")
 ])
 if jobs[0][0]:
     try:
         nodes = []
         wait_oar_job_start(jobs[0][0], jobs[0][1])
         nodes = get_oar_job_nodes(jobs[0][0], jobs[0][1])
         # group nodes by cluster
         sources, targets = [ list(n) for c, n in itertools.groupby(
           sorted(nodes,
                  lambda n1, n2: cmp(
                    get_host_cluster(n1),
                    get_host_cluster(n2))),
           get_host_cluster) ]
         servers = Remote("iperf -s",
                          targets,
                          connection_params = default_oarsh_oarcp_params)
         for p in servers.processes:
             p.ignore_exit_code = p.nolog_exit_code = True
         clients = Remote("iperf -c {{[t.address for t in targets]}}",
                          sources,
                          connection_params = default_oarsh_oarcp_params)
         servers.start()
         sleep(1)
         clients.run()
         servers.kill().wait()
         print Report([ servers, clients ]).to_string()
         for index, p in enumerate(clients.processes):
             print "client %s -> server %s - stdout:" % (p.host.address,
                                                         targets[index].address)
             print p.stdout
     finally:
         oardel([(jobs[0][0], jobs[0][1])])

This example shows how python try / finally construct can be used to
make sure reserved resources are always released at the end of the
job. It also shows how we can use python tools (itertools.groupby) to
group hosts by cluster, to build an experiment topology, then use this
topology with execo substitutions. The exit code of the servers is
ignored (not counted as an error and not logged) because it is normal
that they are killed at the end (thus they always have a non-zero exit
code).

grid5000 planning
-----------------

In this example, the planning module is used to automatically compute
how many resources we can get on Grid5000.

Here, we simply ask for the maximum number of Grid5000 nodes that we
can get right now for a 10 minutes job, we then perform the
reservation with oargrid, wait the job start and retrieve the list of
nodes. Then, we connect with a TaktukRemote (similar as a Remote, but
using Taktuk under the hood, for scaling to huge number of remote
nodes) and remotely execute shell commands to get the current cpufreq
governor for each core, as well as the hyperthreading activation
state. To each remote process, a stdout_handler is added which directs
its stdout to a file on localhost, the filename being <nodename>.out::

 from execo import *
 from execo_g5k import *

 blacklisted = [ "graphite", "reims", "helios-6.sophia.grid5000.fr",
    "helios-42.sophia.grid5000.fr", "helios-44.sophia.grid5000.fr",
    "sol-21.sophia.grid5000.fr", "suno-3.sophia.grid5000.fr" ]

 planning = get_planning()
 slots = compute_slots(planning, 60*10, blacklisted)
 wanted = { "grid5000": 0 }
 start_date, end_date, resources = find_first_slot(slots, wanted)
 actual_resources = distribute_hosts(resources, wanted, blacklisted)
 job_specs = get_jobs_specs(actual_resources, blacklisted)
 jobid, sshkey = oargridsub(job_specs, start_date,
                            walltime = end_date - start_date)
 if jobid:
     try:
         wait_oargrid_job_start(jobid)
         nodes = get_oargrid_job_nodes(jobid)

         check = TaktukRemote('cat $(find /sys/devices/system/cpu/ '
                              '-name scaling_governor) ; '
                              'find /sys/devices/system/cpu '
                              '-name thread_siblings_list -exec cat {} \; '
                              '| grep , >/dev/null '
                              '&& echo "hyperthreading on" '
                              '|| echo "hyperthreading off"',
                              nodes,
                              connection_params = default_oarsh_oarcp_params)
         for p in check.processes:
             p.stdout_handlers.append("%s.out" % (p.host.address,))
         check.run()
     finally:
         oargriddel([jobid])

After running this code, you get in the current directory on localhost
a file for each remote hosts containing the scaling governor and
hyperthreading state (easy to check they are all the same with ``cat *
| sort -u``)

This code also shows how some clusters / sites or nodes can be
blacklisted if needed.

Note that with this kind of code, there is still the possibility that
the oar or oargrid reservation fails, since oar is not transactional,
and someone can still reserve some resources between the moment we
inquire the available resources and the moment we perform the
reservation.

Note also that as this example uses TaktukRemote, it must be run from
inside Grid5000.

The planning module has several possibilities and modes, see its
documentation for further reference.

execo_g5k.api_utils
-------------------

This module is automatically imported only if httplib2 is available.

It provides various useful function which deal with the Grid5000 API.

For example, to work interactively on all grid5000 frontends at the
same time: Here we create a directory, copy a file inside it, then
delete the directory, on all frontends simultaneously::

 from execo import *
 from execo_g5k import *
 sites = get_g5k_sites()
 Remote("mkdir -p execo_tutorial/",
        sites,
        connection_params = default_frontend_connection_params).run()
 Put(sites,
     ["~/.profile"],
     "execo_tutorial/",
     connection_params = default_frontend_connection_params).run()
 Remote("rm -r execo_tutorial/",
        sites,
        connection_params = default_frontend_connection_params).run()

If ssh proxycommand and execo configuration are configured as
described in :ref:`tutorial-configuration`, this example can be run
from outside grid5000.

More advanced usages
====================

.. _tutorial-configuration:

Configuration of execo, execo_g5k
---------------------------------

Execo reads configuration file ``~/.execo.conf.py``. A sample
configuration file ``execo.conf.py.sample`` is created in execo source
package directory when execo is built. This file can be used as a
canvas to overide some particular configuration variables. See
detailed documentation in :ref:`execo-configuration` and
:ref:`execo_g5k-perfect_configuration`.

For example, if you use ssh with a proxycommand to connect directly to
grid5000 servers or nodes from outside, as described in
https://www.grid5000.fr/mediawiki/index.php/SSH#Using_SSH_with_ssh_proxycommand_setup_to_access_hosts_inside_Grid.275000
the following configuration will allow to connect to grid5000 with
execo from outside. Note that
``g5k_configuration['oar_job_key_file']`` is indeed the path to the
key *inside* grid5000, because it is used at reservation time and oar
must have access to it. ``default_oarsh_oarcp_params['keyfile']`` is
the path to the same key *outside* grid5000, because it is used to
connect to the nodes from outside::

 import re

 def host_rewrite_func(host):
     return re.sub("\.grid5000\.fr$", ".g5k", host)

 def frontend_rewrite_func(host):
     return host + ".g5k"

 g5k_configuration = {
     'oar_job_key_file': 'path/to/ssh/key/inside/grid5000',
     'default_frontend' : 'lyon',
     'api_username' : 'g5k_username'
     }

 default_connection_params = {'host_rewrite_func': host_rewrite_func}
 default_frontend_connection_params = {'host_rewrite_func': frontend_rewrite_func}

 default_oarsh_oarcp_params = {
     'user':        "oar",
     'keyfile':     "path/to/ssh/key/outside/grid5000",
     'port':        6667,
     'ssh':         'ssh',
     'scp':         'scp',
     'taktuk_connector': 'ssh',
     'host_rewrite_func': host_rewrite_func,
     }

Processes and actions factories
-------------------------------

Processes and actions can be instanciated directly, but it can be more
convenient to use the factory methods `execo.process.get_process`
`execo.action.get_remote`, `execo.action.get_fileput`,
`execo.action.get_fileget` to instanciate the right objects:

- `execo.process.get_process` instanciates a Process or SshProcess
  depending on the presence of argument host different from None.

- `execo.action.get_remote`, `execo.action.get_fileput`,
  `execo.action.get_fileget` instanciate ssh or taktuk based
  instances, depending on configuration variables "remote_tool",
  "fileput_tool", "fileget_tool"