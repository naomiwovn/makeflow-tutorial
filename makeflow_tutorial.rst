=====================================================
*Distributed Computing with Makeflow and Work Queue*
=====================================================

.. sectnum::

.. contents:: Table of Contents


_ Getting Started with Makeflow
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Download and install the CCTools software in your home directory in one of two ways:

To build our latest release:

.. code-block:: bash

    $ wget http://ccl.cse.nd.edu/software/files/cctools-6.2.6-source.tar.gz

    $ tar zxpvf cctools-6.2.6-source.tar.gz

    $ cd cctools-6.2.6-source

    $ ./configure --prefix $HOME/cctools --tcp-low-port 9000 --tcp-high-port 9500

    $ make

    $ make install

    $ cd $HOME

If you use bash then do this to set your path:

.. code-block:: bash

    $ export PATH=$HOME/cctools/bin:$PATH

If you use tcsh instead, then do this:

.. code-block:: bash

    $ setenv PATH $HOME/cctools/bin:$PATH

Now double check that you can run the various commands, like this:

.. code-block:: bash

    $ makeflow -v

    $ work_queue_worker -v

    $ work_queue_status

_ Your First Makeflow Project
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Start by making your own makeflow file, and the input files it will need. Lets count some numbers.

.. code-block:: bash

    $ mkdir mkflow_count

.. code-block:: bash

    $ cd mkflow_count

.. code-block:: bash

    $ nano count.py

Copy the following code to your count.py:

.. code-block:: python

    # Loop and Print 0 - 10
    	for i in range(11):
    		print(i)

Now make your first makeflow file in jx format

.. code-block:: bash

    $ nano count.jx

Add the following code to your count.jx

.. code-block:: python

    {
	   "rules" : [
	       {
        			"command" : "python count.py > count.txt",
        			"outputs" : ["count.txt"],
        			"inputs" : ["count.py"],
        			"local_job" : 1
        	   }
        ]
    }

Makeflow's JX format specifies a set of rules. For this example, we will only use one rule. Rules require you to specify the files used as "inputs" and the files you expect to generate as "outputs". 

.. figure:: makeflow_structure.png

The "command" option is used to specify a shell command. In this case, it's the python command to run the python script and the > to write the output of the file to a new file called count.txt.

The [ ] brackets indicate a list. This list contains one rule in { } curly braces. 

Save the file and exit back to the command line and enter the following:

.. code-block:: bash

    $ makeflow --jx count.jx

You will need to tell makeflow that you want to use the JX format by adding the --jx then specify the jx file you want to use.


You should see something similar to the following:

.. code-block:: bash

    parsing count.jx...
    local resources: 12 cores, 15878 MB memory, 118331 MB disk
    max running local jobs: 12
    checking count.jx for consistency...
    count.jx has 1 rules.
    starting workflow....
    submitting job: python count.py > count.txt
    submitted job 6373
    job 6373 completed
    nothing left to do.


In the rules, we specified we were running a local job. Here makeflow utilized all available cores on my local machine. 

Now check your mkflow_count directory:

.. code-block:: bash

    $ ls

Makeflow has created two new files.

.. code-block:: bash

    count.jx  count.jx.makeflowlog  count.py  count.txt

You will see a new file called count.txt

.. code-block:: bash

    $ cat count.txt

The output file should contain the numbers printed by count.py

.. code-block:: bash

    0
    1
    2
    3
    4
    5
    6
    7
    8
    9
    10

Now try submitting the makeflow job again. You should see something like the following.

.. code-block:: bash

    $ makeflow --jx count.jx
    parsing count.jx...
    local resources: 12 cores, 15878 MB memory, 118331 MB disk
    max running local jobs: 12
    checking count.jx for consistency...
    count.jx has 1 rules.
    recovering from log file count.jx.makeflowlog...
    starting workflow....
    nothing left to do.

After starting workflow... makeflow decides there's nothing left to do. That's because all the output files are already built.

Clean everything up before trying again:

.. code-block:: bash

    makeflow --jx --clean count.jx


Now check the makeflow directory again

.. code-block:: bash

    $ ls

You should see your original two files, the other two are cleaned up.

.. code-block:: bash

    count.jx  count.py

Now you can run makeflow again to generate your output file

.. code-block:: bash

    $ makeflow --jx count.jx

_ Makeflow Example
~~~~~~~~~~~~~~~~~~~~~

Let's begin by using Makeflow to run a handful of simulation codes.

First, make and enter a clean directory to work in inside of ``makeflow-examples``:

.. code-block:: bash

    $ cd $HOME/makeflow-examples

    $ mkdir tutorial

    $ cd tutorial

Download this program, which performs a highly sophisticated simulation of black holes colliding together:

.. code-block:: bash

    $ wget http://ccl.cse.nd.edu/software/tutorials/cyversecc18/simulation.py

Try running it once, just to see what it does:

.. code-block:: bash

    $ chmod 755 simulation.py

    $ ./simulation.py 5

Now, let's use Makeflow to run several simulations.

Create a file called ``example.makeflow`` and paste the following

text into it:

.. code-block:: text

    input.txt:

        LOCAL /bin/echo "Hello Makeflow!" > input.txt

    output.1: simulation.py input.txt

        ./simulation.py 1 < input.txt > output.1

    output.2: simulation.py input.txt

        ./simulation.py 2 < input.txt > output.2

    output.3: simulation.py input.txt

        ./simulation.py 3 < input.txt > output.3

    output.4: simulation.py input.txt

        ./simulation.py 4 < input.txt > output.4

To run it on your local machine, one job at a time:

.. code-block:: bash

    $ makeflow example.makeflow -j 1

Note that if you run it a second time, nothing will happen, because all of the files are built:

.. code-block:: bash

    $ makeflow example.makeflow

    $ makeflow: nothing left to do

Use the -c option to clean everything up before trying it again:

.. code-block:: bash

    $ makeflow -c example.makeflow

Here are some other options for built-in batch systems:

.. code-block:: bash

    $ makeflow -T slurm example.makeflow

    $ makeflow -T torque example.makeflow

    $ makeflow -T sge example.makeflow

_ Running Makeflow with Work Queue
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You will notice that a workflow can run very slowly if you submit each job individually. To get around this limitation, we provide the Work Queue system. This allows Makeflow to function as a master process that quickly dispatches work to remote worker processes. 

.. code-block:: bash

    $ makeflow -c example.makeflow

    $ makeflow -T wq example.makeflow -p 0

    listening for workers on port XXXX.

    ...

Now open up another shell and run a single worker process:

.. code-block:: bash

    $ work_queue_worker crcfe01.crc.nd.edu XXXX

Go back to your first shell and observe that the makeflow has finished.

Of course, remembering port numbers all the time gets old fast,

so try the same thing again, but using a project name:

.. code-block:: bash

    $ makeflow -c example.makeflow

    $ makeflow -T wq example.makeflow -N project-$USER

    listening for workers on port XXXX

    ...

Now open up another shell and run your worker with a project name:

.. code-block:: bash

    $ work_queue_worker -N project-$USER



_ Running on Atmosphere/Jetstream
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To start out we are going to launch an instance:

We are going to be using an Ubuntu instance with Docker already installed:

`Ubuntu 16.04 Devel and Docker v.1.13 <https://use.jetstream-cloud.org/application/images/107>`_

Please note you should use images of at least **Medium** size.

Once the instance is up, we are going to add a few packages to allow for easy installation.

Most of these packages are already installed on batch submission sites, but possibly not in all

Jetstream instances.

.. code-block:: bash

    $ sudo apt-get install zlib1g-dev libncurses5-dev g++

Additionally, we are going to add our current user to the docker group:

.. code-block:: bash

    $ sudo usermod -aG docker ${USER}

We are also going to install 

Singularity if you have not done so yet. This should be done using the

provided ansible script:

.. code-block:: bash

    $ ezs

After adding this log out and back in. 

.. code-block:: bash

    $ exit

Now re-open the in web-shell.

Once you are logged back in, we are going to pull the docker image we will use today:

.. code-block:: bash

    $ docker pull nekelluna/ccl_makeflow_examples

    $ docker save -o mfe.tar nekelluna/ccl_makeflow_examples

    $ singularity pull docker://nekelluna/ccl_makeflow_examples

Note: If you would like to test this out with Work Queue on another machine, now is a great time

to launch and do these setup steps on each machine. ``Hint hint`` you should do this.

_ Using Containers with Makeflow
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We are going to start using Containers in the Makeflow by showing the different configurations

that we talked about in the slides. There is a simple, 1 rule, makeflow that we will use to show

these:

.. code-block:: make

    hello.out:

        echo "hello, world!" > hello.out

The first configuration we discussed would be to run both the Makeflow and the Worker inside

of container to allow for a consistent environment. 

We will not do this here, as that is extremely similar to running in Atmosphere/Jetstream to begin with.

This is great way to test out different software configurations when determining what is needed for a workflow

and how different software will interact.

The second configuration is to run each task inside of separate containers. This configuration is useful

for specializing the configuration each task uses and not assuming the execution site has any software

requirements aside from docker or singularity.

Assuming we are wrapping each task in a container, there are two ways to do this in Makeflow. The first is

to manually add the container to your command. This allows for precise control of how the task is executed

and in which container this occurs. We will show this now:

We are going to look at what the hello-containers folder:

.. code-block:: bash 

    $ cd $HOME/makeflow-examples

    $ cd hello-containers

Inside of the ``hello-containers`` folder, there is a python script, ``hello_world_creator.py``, 

that will create a simple hello world example which uses a container:

.. _docker:

To test with Docker:

.. code-block:: bash

    $ python hello_world_creator.py --docker nekelluna/ccl_makeflow_examples

.. _singularity:

To test with Singularity

.. code-block:: bash

    $ ln -s $HOME/ccl_makeflow_examples.simg ccl_makeflow_examples.simg

    $ python hello_world_creator.py --singularity ccl_makeflow_examples.simg

After running these, look at ``hello_world.mf`` and see how the above run has been

wrapped by the container command. Now we are just going to run this locally:

.. code-block:: bash

    $ makeflow hello_world.mf -T local

Now, instead of wrapping each task by hand, we are going to assume that each task will use

the same container. For this we will use Makeflow's built in support for containers. 

We will assume that the above steps for either docker or singularity have been done:

.. code-block:: bash 

    $ cd $HOME/makeflow-examples

    $ cd hello-world

We are going to start from the existing ``hello-world`` example. To run Makeflow with

either docker or singularity we specify the container in the arguments:

Docker: 

.. code-block:: bash

    $ ln -s $HOME/mfe.tar mfe.tar

    $ makeflow hello_world.mf --docker=nekelluna/ccl_makeflow_examples --docker-tar=mfe.tar

 

Singularity:

.. code-block:: bash

    $ ln -s $HOME/ccl_makeflow_examples.simg ccl_makeflow_examples.simg

    $ makeflow hello_world.mf --singularity=ccl_makeflow_examples.simg 

 

We have three additional examples that will work with the above provided container.

- `5.1. BLAST in a Container`_

- `5.2. BWA in a Container`_

- `5.3. Text Analysis in a Container`_

Each of these examples may have a small amount of setup to pull/compile the software needed. 

_ BLAST in a Container
~~~~~~~~~~~~~~~~~~~~~~~~~

BLAST is a common bioinformatic application used for determining alignment of a query dataset with

a known reference set. BLAST compares each line independently of each other, allowing for clear 

parallelism opportunities.

.. code-block:: bash 

    $ cd $HOME/makeflow-examples

    $ cd blast

We use an older BLAST executable for this example, as this creation script has not been changed. These commands

pull down the executable and a reference database.

.. code-block:: bash

    $ wget ftp://ftp.ncbi.nlm.nih.gov/blast/executables/legacy.NOTSUPPORTED/2.2.26/blast-2.2.26-x64-linux.tar.gz

    $ tar xvzf blast-2.2.26-x64-linux.tar.gz

    $ cp blast-2.2.26/bin/blastall .

    $ wget ftp://ftp.ncbi.nlm.nih.gov/blast/db/nt.44.tar.gz

    $ mkdir nt

    $ tar -C nt -xvzf nt.44.tar.gz

We are now going to generate a random data set to align with the reference:

.. code-block:: bash

    $ ./fasta_generator 200 1000 > test.fasta

Based on the generated data, we will now write a makeflow:

.. code-block:: bash

    $ ./makeflow_blast -d nt -i test.fasta -p blastn --num_seq 5 --makeflow blast_test.mf

Assuming you have already pulled the images needed for either singularity_ 

or docker_ we will run them similarly to how it was done above:

Docker: 

.. code-block:: bash

    $ ln -s $HOME/mfe.tar mfe.tar

    $ makeflow blast_test.mf --docker=nekelluna/ccl_makeflow_examples --docker-tar=mfe.tar

 

Singularity:

.. code-block:: bash

    $ ln -s $HOME/ccl_makeflow_examples.simg ccl_makeflow_examples.simg

    $ makeflow blast_test.mf --singularity=ccl_makeflow_examples.simg 

 

_ BWA in a Container
~~~~~~~~~~~~~~~~~~~~~~~

BWA is similar to BLAST in that it is a bioinformatics tool that aligns a query dataset 

with a reference dataset. BWA does not operate on highly structured reference data like

BLAST, but uses a fasta or fastq data file for both the query and reference.

.. code-block:: bash 

    $ cd $HOME/makeflow-examples

    $ cd bwa

We will download and compile the software:

.. code-block:: bash

    $ git clone https://github.com/lh3/bwa bwa-src

    $ cd bwa-src

    $ make

    $ cp bwa ..

    $ cd ..

Create the data we will use for the analysis:

.. code-block:: bash

    $ ./fastq_generate.pl 10000 1000 > ref.fastq

    $ ./fastq_generate.pl 1000 100 ref.fastq > query.fastq

The first line creates the reference dataset and the second will create a query dataset based on a portion

of the provided reference dataset. This allows us to guarantee there will be some overlap and data analysis at

each step for this example.

Now we will create the makeflow based on the input dataset:

.. code-block:: bash

    $ ./make_bwa_workflow --ref ref.fastq --query query.fastq --num_seq 100 > bwa.mf

Again assuming that the docker and singularity images have been pulled down, run the makeflow:

Docker: 

.. code-block:: bash

    $ ln -s $HOME/mfe.tar mfe.tar

    $ makeflow bwa.mf --docker=nekelluna/ccl_makeflow_examples --docker-tar=mfe.tar

 

Singularity:

.. code-block:: bash

    $ ln -s $HOME/ccl_makeflow_examples.simg ccl_makeflow_examples.simg

    $ makeflow bwa.mf --singularity=ccl_makeflow_examples.simg 

_ Text Analysis in a Container
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The test analysis example that we are providing is a simple makelfow that analyzes a set

of Shakespeare's plays. This workflow gives an example of using Makeflow to parallelize 

a text search through a collection of William Shakespeare's plays. 

Makeflow will download the plays, package up the version of Perl at the location Makeflow is running, 

and run a text analysis Perl script in parallel to figure out which character had the most dialogue 

out of the plays selected. 

.. code-block:: bash 

    $ cd $HOME/makeflow-examples

    $ cd shakespeare

This workflow relys on Perl and CCTools being installed, so there is no further setup needed.

Docker:

.. code-block:: bash

    $ ln -s $HOME/mfe.tar mfe.tar

    $ makeflow shakespeare.makeflow --docker=nekelluna/ccl_makeflow_examples --docker-tar=mfe.tar

 

Singularity:

.. code-block:: bash

    $ ln -s $HOME/ccl_makeflow_examples.simg ccl_makeflow_examples.simg

    $ makeflow shakespeare.makeflow --singularity=ccl_makeflow_examples.simg

_ Cooperative Computing Lab Resources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

More information can be found a http://ccl.cse.nd.edu.

For specific information on Makeflow execution see http://ccl.cse.nd.edu/software/manuals/makeflow.html

For Work Queue see http://ccl.cse.nd.edu/software/manuals/workqueue.html.






_ About Makeflow
~~~~~~~~~~~~~~~~~
Makeflow is a workflow system for executing large complex workflows on clusters, clouds, and grids.

_ **Makeflow is easy to use.** if you can write a Makefile, then you can write a Makeflow

.. figure:: makeflow-flow-color.png

The Makeflow language is similar to traditional Make, so . A workflow can be just a few commands chained together, or it can be a complex application consisting of thousands of tasks. It can have an arbitrary DAG structure and is not limited to specific patterns.

- **Makeflow is production-ready.** Makeflow is used on a daily basis to execute complex scientific applications in fields such as data mining, high energy physics, image processing, and bioinformatics. It has run on campus clusters, the Open Science Grid, NSF XSEDE machines, NCSA Blue Waters, and Amazon Web Services. Here are some real examples of workflows used in production systems:

- **Makeflow is portable.** A workflow is written in a technology neutral way, and then can be deployed to a variety of different systems without modification, including local execution on a single multicore machine, public cloud services such as Amazon EC2 and Amazon Lambda, batch systems like HTCondor, SGE, PBS, Torque, SLURM, or the bundled Work Queue system. Makeflow can also easily run your jobs in a container environment like Docker or Singularity on top of an existing batch system. The same specification works for all systems, so you can easily move your application from one system to another without rewriting everything.

- **Makeflow is powerful.** Makeflow can handle workloads of millions of jobs running on thousands of machines for months at a time. Makeflow is highly fault tolerant: it can crash or be killed, and upon resuming, will reconnect to running jobs and continue where it left off. A variety of analysis tools are available to understand the performance of your jobs, measure the progress of a workflow, and visualize what is going on.


_ About Work Queue
~~~~~~~~~~~~~~~~~~

Work Queue is a framework for building large master-worker applications that span thousands of machines drawn from clusters, clouds, and grids. Work Queue applications are written in C, Perl, or Python using a simple API that allows users to define tasks, submit them to the queue, and wait for completion. Tasks are executed by a standard worker process that can run on any available machine. Each worker calls home to the master process, arranges for data transfer, and executes the tasks. The system handles a wide variety of failures, allowing for dynamically scalable and robust applications.

.. figure:: Workqueue-flow.png

Work Queue has been used to write applications that scale from a handful of workstations up to tens of thousands of cores running on supercomputers. Examples include Lobster, NanoReactors, ForceBalance, Accelerated Weighted Ensemble, the SAND genome assembler, the Makeflow workflow engine, and the All-Pairs and Wavefront abstractions. The framework is easy to use, and has been used to teach courses in parallel computing, cloud computing, distributed computing, and cyberinfrastructure at the University of Notre Dame, the University of Arizona, and the University of Wisconsin - Eau Claire.


_ Pre-requisites
~~~~~~~~~~~~~~~~~
- cctools
- python
- your favorite text editor



