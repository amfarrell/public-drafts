You're going to learn:
- How to run two virtual machines as linux servers on your laptop using [vagrant](https://docs.vagrantup.com/v2/why-vagrant/index.html).
- How to write reproducable configurations of those servers using [saltstack](http://saltstack.com/).
- How to set up one of those servers with nginx and a web app.
- How to debug salt states with a few different techniques.

Who is this for
===============
   This tutorial is for a Python programmer who has run a linux or unix
machine and feels okay on the commandline, but wants to really learn devops. If
you can open and edit files, install packages with `apt-get` or `yum`, and google for ways to debug problems, you're good to go.
    If you already have some systems administration experience but have felt 
wish that managing a servers could feel like writing a program than caring
for a pet, you'll find this valuable. 

    <aside>
    If you want to get more comfortable with the unix command line, check out http://cli.learncodethehardway.org/book/ and then https://nixsrv.com/llthw.
    If you want to get more comfortable with python, check out http://learnpythonthehardway.org/ and then http://chimera.labs.oreilly.com/books/1234000000754
    </aside>

I'm going to hold your hand through the process of setting up two servers on
virtual machines, installing an existing django app with gunicorn, nginx,
postgres, and celery, then scaling that app up on {{ public cloud service,
probably DO, but people are probably more interested in AWS. }}. This tutorial
is broken up into {{ number }} steps and you should do them in order. However, if
you want to to a particular point, you can as long as you have installed the
prerequesites in step-0. Start any given step with `git checkout step-N` an then
`./refresh`. You can also use the refresh script at any point to destroy the VMs
we will have created and to spin up new ones in the state they were at the beginning
of any given excercise. {# TODO: write refresh script for earlier steps as
well. #} Each step also has tests that run at the beginning to check that the
previous step has been done correctly. Run these with ./test.sh {# TODO: That
script isn't windows-portable #}

To get started,
Please clone https://github.com/amfarrell/saltmarsh and run `git checkout step-0`

Step 0 Prerequesites: Python, Vagrant, & Virtualbox
===================================================

First, make sure you have a version of python installed. This tutorial is
tested to work with Python 3.4, Python 3.5, and legacy Python 2.7.
{# TODO: fix Python 3.4 & 3.5 error #}
If you do not have python, please download and install Python 3.5 from
https://www.python.org/downloads/.
After that, please install the python libraries we will be using with
`pip install -r ./requirements.txt`. You may want to want to do this within a
virtualenv or conda environment.


Virtualbox lets you run a virtual machine on your laptop for free.
Vagrant makes it simple and easy.

First, download and install Virtualbox.
<distrotable>
[windows](http://download.virtualbox.org/virtualbox/5.0.6/VirtualBox-5.0.6-103037-Win.exe)
[OSX](http://download.virtualbox.org/virtualbox/5.0.6/VirtualBox-5.0.6-103037-OSX.dmg)
[Linux](https://www.virtualbox.org/wiki/Linux_Downloads)
</distrotable>

Once you have installed virtualbox, run `virtualbox --help` and you should see
```
Oracle VM VirtualBox Manager 5.0.6
(C) 2005-2015 Oracle Corporation
All rights reserved.
```
at the top.
Opening it, you should see [something that looks like this](virtualbox-5.png).

Next, download and install Vagrant
<distrotable>
[Windows](http://www.vagrantup.com/downloads)
[OSX](http://www.vagrantup.com/downloads)
[Linux](http://www.vagrantup.com/downloads)
</distrotable>

Once that is installed, run `vagrant --version` and you should see
```
Vagrant 1.7.4
```

Again run `py.test -v --tb=line` and you will see all tests pass.

Step 1: Running a Single Virtual Machine on Vagrant
===================================================

{# TODO: write refresh script #}
First, run `vagrant global-status` and ensure that it prints out something like

```
id       name   provider state  directory
--------------------------------------------------------------------
There are no active Vagrant environments on this computer! Or,
you haven't destroyed and recreated Vagrant environments that were
started with an older version of Vagrant.
```

Now, we're going to create a vagrantfile. In your favorite text editor, create a file with the name `Vagrantfile` whose contents look like

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.box_url = "ubuntu/trusty64"
end
```

Run `vagrant up` to create your machine. When you run `vagrant global-status`, you will now see
```
id       name    provider   state   directory
-----------------------------------------------------------------------------
b68783c  default virtualbox running /Users/bbitdiddle/saltmarsh

The above shows information about all known Vagrant environments
on this machine. This data is cached and may not be completely
up-to-date. To interact with any of the machines, you can go to
that directory and run Vagrant, or you can use the ID directly
with Vagrant commands from any directory. For example:
"vagrant destroy 1a2b3c4d"
```
Run `vagrant ssh` to log in to it. You should see yourself at a terminal that looks like
```
vagrant@vagrant-ubuntu-trusty-64:~$
```
Vagrant creates a special directory /vagrant on this new machine. Lets `cd /vagrant` and look around.
```
vagrant@vagrant-ubuntu-trusty-64:/vagrant$ ls
tests  Vagrantfile
```
Run `echo 'synced' >> synced-file` within this directory and then hit ctrl-D to log out of the machine.
You'll see that this directory on the guest vm also exists on the host.

again, run `py.test -v --tb=line` If all the tests pass, commit the two new files and `git checkout step-2`.


    <aside>
    If you are interested in learning more about vagrant in general, their [documentation](https://docs.vagrantup.com/v2/getting-started/index.html) is quite good. I'll also show you a bit more as we move forward.
    </aside>


Step 2: Running two Machines on Vagrant
=======================================

We no longer need the single machine we just created, so go ahead and save some space by destroying it
Run `vagrant global-status` again and you should see a table with something like

```
id       name    provider   state   directory
------------------------------------------------------------------------------------------------------------
b68783c  default virtualbox running /Users/bbitdiddle/saltmarsh
```

See that id in the leftmost column? Take that and run `vagrant destroy b68783` to get rid of that vm.

Salt master-minion architecture
-------------------------------
Salt operates on a master-minion architecture. This means that there are two processes that can run on our

{{ explain salt-master and its relationship with salt-minion }}

{{ master-minion diagram }}

{{ Include that the salt-minion's name is based on the FQDM of the machine it runs on. }}

, so to do anything interesting with it, we are going to want to run not one,
but two virtual machines: one that will be the master as well as having a minion process running on it, and one that will simply be a minion. For vagrant to do this, it needs the `hostmanager` plugin. Install that with
`vagrant plugin install vagrant-hostmanager`. After running it, `vagrant plugin list` should include a line that looks like `vagrant-hostmanager (1.6.1)`.

Now, with that installed, we can modify our vagrantfile to look like

```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.box_url = "ubuntu/trusty64"

  config.vm.define :frodo , primary: true do |machine|
    machine.vm.provider :virtualbox do |v|
      v.name = "frodo"
    end

    machine.vm.hostname = "frodo"
    machine.vm.synced_folder "./frodo_vagrant", "/vagrant", :create => true

    machine.vm.network :private_network, ip: "10.10.10.100"
    machine.vm.network :forwarded_port, guest: 80, host: 8001
    machine.vm.network :forwarded_port, guest: 8080, host: 8081
    machine.vm.provision :hostmanager
  end

  config.vm.define :samwise do |machine|
    machine.vm.provider :virtualbox do |v|
      v.name = "samwise"
    end

    machine.vm.hostname = "samwise"
    machine.vm.synced_folder "./samwise_vagrant", "/vagrant", :create => true

    machine.vm.network :private_network, ip: "10.10.10.101"
    machine.vm.network :forwarded_port, guest: 80, host: 8002
    machine.vm.network :forwarded_port, guest: 8080, host: 8082
    machine.vm.provision :hostmanager
  end

  config.vm.provision :hostmanager
end
```
And run `vagrant up` to create our machines.

Lets dig into what this does:
This will create two machines, `frodo` (who will be our salt master) and `samwise` (who will be our salt minion).
It assigns the IP addresses `10.10.10.100` and `10.10.10.101` to these.
We will be running webservers on these machines, so we make the following ports available on each.

|host  |  frodo  |  samwise|
|------|---------|---------|
|8001  |     80  |         |
|8081  |   8080  |         |
|8002  |         |       80|
|8082  |         |     8080|

We also have created two separate synced folders: frodo_vagrant and samwise_vagrant.
{# TODO: diagram of synced folders and open ports of both machines and host #}

Verify now that you are able to connect to both of them with `vagrant ssh frodo` and `vagrant ssh samwise` respectively.

Congratulations! We now have two linux machines that we will use for the rest of the tutorial.

Step 3: Installing Salt
=======================

From here on out, you are going to need to have at least two terminals open: one where you have connected to the master with `vagrant ssh frodo`
and another where you have connected to the minion with `vagrant ssh samwise`.


both: `sudo add-apt-repository ppa:saltstack/salt`
both: `sudo apt-get update`
samwise: `sudo apt-get install salt-minion`
frodo: `sudo apt-get install salt-minion salt-master`

We now need to make the samwise aware of frodo.

On frodo, run `sudo salt-run manage.status`. You'll see that there are no minions connected to it. To command a salt minion, a salt master needs to accept that minion's public key so that it can securely send commands to it.
```
vagrant@frodo:~$ sudo salt-run manage.status
No minions matched the target. No command was sent, no jid was assigned.
down:
up:
```

To see the list of keys that have been submitted to the salt master Run `sudo salt-key -L` on frodo. You'll see that its list of unaccepted keys is empty, which means that samwise has not yet submitted a key.
```
vagrant@frodo:~$ sudo salt-key -L
Accepted Keys:
Denied Keys:
Unaccepted Keys:
Rejected Keys:
```
We need to get samwise to do that. First, it needs to know how to reach the salt master. This should have been be taken care of by the vagrant-hostmanager plugin.
Check that it was by running `ping frodo` on samwise. You should see
```
vagrant@samwise:~$ ping frodo
PING frodo (10.10.10.100) 56(84) bytes of data.
64 bytes from frodo (10.10.10.100): icmp_seq=1 ttl=64 time=0.387 ms
64 bytes from frodo (10.10.10.100): icmp_seq=2 ttl=64 time=0.382 ms
64 bytes from frodo (10.10.10.100): icmp_seq=3 ttl=64 time=0.449 ms
64 bytes from frodo (10.10.10.100): icmp_seq=4 ttl=64 time=0.357 ms
^C
--- frodo ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 2998ms
rtt min/avg/max/mdev = 0.357/0.393/0.449/0.041 ms
```
This is because vagrant-hostmanager added a line that looks like `10.10.10.100  frodo` to /etc/hosts on samwise.
If you are actually running salt on a set of digitalocean or aws machines, you will need to add this line yourself to point to the correct host.
But even though samwise knows how to find frodo, we still need to tell the salt-minion process running on samwise to call frodo the salt-master and send over the public key.

To accomplish this, open the minion config file `/etc/salt/minion`.
On line 16, you should see `#master: salt`.
This is the default value for the hostname of the salt master. Uncomment the line and change it to `master:frodo`.
Then, restart the salt-minion process with `sudo service salt-minion restart`

Now, back on frodo, try again to run `sudo salt-key -L` and you should now see an unaccepted key from samwise.
```
vagrant@frodo:~$ sudo salt-key -L
Accepted Keys:
Denied Keys:
Unaccepted Keys:
samwise
Rejected Keys:
```
Accept this key with `sudo salt-key --accept samwise`. Run `sudo salt-key -L` again and you'll now see
```
vagrant@frodo:~$ sudo salt-key -L
Accepted Keys:
Denied Keys:
Unaccepted Keys:
samwise
Rejected Keys:
```
Do the same for the salt-minion running on frodo:
```
vagrant@frodo:~$ sudo -i
root@frodo:~# cp /etc/salt/minion /etc/salt/minion.bak
root@frodo:~# cat /etc/salt/minion.bak | sed 's/^#master:\ salt/master: frodo/' > /etc/salt/minion
root@frodo:~# service salt-minion restart
root@frodo:~# salt-key --accept frodo -y
```

Now, `sudo salt-run manage.status` will show the both minions.
```
vagrant@frodo:~$ sudo salt-run manage.status
down:
up:
    - samwise
    - frodo
```
    <aside>`salt-run` is used just for convenience functions called <a href="https://docs.saltstack.com/en/latest/ref/runners/">Salt Runners</aside>

Now finally, we can run commands on the minions from the master. On frodo, run
```
sudo salt '*' cmd.run 'cat /var/log/salt/minion > /vagrant/`hostname`-minion-log'
```
This tells the salt-master process to find any minion and tell it to run that command.
You will see the result of running the command if you look for the files `frodo_vagrant/frodo-minion-log` and `samwise_vagrant/samwise-minion-log`.

And then run ./test.sh

Congratulations! You now have saltstack up and running!

Here is what things look like now:
<diagram>
Get up, stretch, grab a glass of water, and get ready, because in the next section, we are now going to dive into how salt works to configure machines.

Step 4 - States: Enforcing a Configuration on a Machine
=======================================================

The basic unit of salt configuration is a single "state". A state expresses
some condition about the system and, when applied to a minion, enforces that
the condition is true.

To see how this works, we are going to look at the logs of the salt-minion on samwise as
it is managed by the salt-minion on frodo. So, on samwise, open /etc/salt/minion, go to line 522,
uncomment it, and change it to `log_level_logfile: 'info'`. Then, `sudo service restart salt-minion`
and start watching by running `tail -f /var/log/salt/minion`. While we leave that terminal up on samwise,
Everything else we do in this section we will do on frodo.

Our first example, on frodo, run `salt 'samwise' state.single pkg.installed
name='silversearcher-ag'`. This tells the salt salt-master to apply this state
to any minion with the name "samwise". This state enforces that a package with
the name silversearcher-ag is installed. From the logs on samwise, you can see
that the salt-minion runs `dpkg-query` to see if the debian package is already
installed and then `apt-get` to actually install the package. If our minion was
a Red Hat or CentOS system, salt-minion would use the equivalent packages for
those systems.

Run `sudo salt 'samwise' state.single pkg.installed name='silversearcher-ag'` again.
You now see that the salt-minion runs the same `dpkg-query` command and immediately
after, concludes `Package tig is already installed.` and does not run `apt-get`.
This is the flow of a basic state: run a command to check to see if some condition
is true. If it is not, then run another command to make it so.

run ./test.sh, and you're ready to `git checkout step-5`, where we'll put this in a file.

Step 5 - SLS files: So let it be written. So let it be done.
============================================================

I promised you at the beginning of this tutorial that I would take you to the
point where doing this felt like writing and testing code. Here is where you
start writing code. Specifically, you will write salt states in YAML.
I mentioned at the beginning that this is a test-driven tutorial. So, you'll
also write a test for each of your salt states. I'll show you how to write both.

    <aside>I'm going to be show you tests written in python, but feel free to adapt
    them to the langauge you're most comfortable with. If you do write tests in another language,
    please do send me an email with a link; I'd like to read them. If you'd like
    to get better at writing tests in python, I highly recommend Harry Percival's 
    excellent book <a href="http://chimera.labs.oreilly.com/books/1234000000754"> Test-Driven
    Development with Python </a></aside>

Before we write our states, we first must tell salt-master where to find them.
Create a directory /vagrant/salt on frodo. This will be the root dirctory where
we place all of our states. Create a file there named `ag.sls` with the contents
```
install-silversearcher:
  pkg.installed:
    - name: silversearcher-ag
```
And save it to this directory. Note that because an salt state files can contain more
than one state, we needed to give each one an id like "install-silversearcher".

Next, open the salt-master config file /etc/salt/master. Go to lines 411-412,
and uncomment them. Then, change line 413 to `    - /vagrant/salt`.  To check
yourself, run `grep ^file_roots -A 2 /etc/salt/master`
The output should be
```
file_roots:
  base:
    - /vagrant/salt
```
This creates an environment called `base` which is where salt-master will look
for states that it can send to the salt-minions. An environment can have
multiple directories but for now, it only has `/vagrant/salt`. Finally, run
`sudo service salt-master restart` to load this new configuration. Run 
`sudo salt 'samwise' state.sls ag` which will tell salt to
look for the sls file with the name `ag.sls` in the file_roots of the base environment,
then to send that sls file to the minion to enforce all the states in it. If you are
still running `tail -f /var/log/salt/minion` on samwise, you will see that
running states this way has the same effect as running each of them individually with
`state.single`.

    <aside>You can also run a single state in an sls file with `state.sls_id install-silversearcher ag`</aside>

You should also see on frodo that applying the state printed something out that looks like
```
samwise:
    ----------
    pkg_|-install-silversearcher_|-silversearcher-ag_|-installed:
        ----------
        __run_num__:
            0
        changes:
            ----------
        comment:
            Package silversearcher-ag is already installed.
        duration:
            332.933
        name:
            silversearcher-ag
        result:
            True
        start_time:
            22:32:33.250542
```

This gives states **idempotence** and idempotence is fantastic.
{{ explain what is so great about idempotence. }}

run ./tests.sh and Congratulate yourself on writing your first salt state.

Step 6: State Depencencies
==========================


When configuring servers, we usually need more than one package installed. In fact, you probably need
- A web server installed
- That same web server configured
- the web server restarted so that it can can load the configuration
- 
{{
An illustration that includes a bunch of things, including
- web server installed
- latest version of app downloaded
- static assets compiled
}}


Lets write our own more detailed salt states in this directory, which is synced
at `frodo_vagrant/salt`. So, Create a file at `frodo_vagrant/salt/django.sls`.
We are going to write states to get a django app running in a virtualenv.
I've written a couple convenience functions for running commands
on frodo and samwise that you should import from tests/test_utils.py. You can
see how they are used in the tests I have written in tests/test_step5.py.

The first step in deploying our app is going to be to clone the repository,
https://github.com/amfarrell/django-example. We will take advantage of the
existance of `samwise_vagrant` as as a directory synced with `/vagrant/` on
samwise to test for the existance of the git repository on samwise.

Create this test and name it test_states_django.py
```
import os.path
import unittest
import yaml
import shutil
from test_utils import run_frodo, run_samwise, SaltStateTestCase

class TestSamwiseRepoCloned(SaltStateTestCase):

    def test_git_repo_exists(self):
        if os.path.exists('samwise_vagrant/django-example'):
            shutil.rmtree('samwise_vagrant/django-example')
        state_response_data = self.run_state(state_file='django', target='samwise')
        #This calls state.sls for you and passes the output to a dictionary.
        assert os.path.exists('samwise_vagrant/django-example/manage.py'), \
            "The app's git repository has not been cloned."
```
{{ figure out the madness with how imports work in python 3.4/5 }}
The structure of this test is essentially:
1) Check if the state is already enforced.
2) If so, make it no longer enforced.
3) Run the state.
4) Check that the state is enforced again.
Run this test with `py.test tests/test_states_django.py`

You'll see that it fails on an AssertionError. Now that we have a failing test,
we can write a salt state that causes it to pass. How do we write a salt state for
cloning git repositories? Salt comes with a large number of state modules builtin
that are used for writing states. You've already seen one, `pkg` and its function
`pkg.installed`. Go to <a href="https://docs.saltstack.com/en/latest/ref/states/all/">
https://docs.saltstack.com/en/latest/ref/states/all/</a> and in search box on the right,
search for "git". The first result will be for the 
<a href="https://docs.saltstack.com/en/latest/ref/states/all/salt.states.git.html">git state module</a>.
You'll see five functions in this state module: `config_set`, `config_unset`,
`latest`, `mod_run_check`, and `present`. Look at the descriptions for each of them and,
you should guess that we want `salt.states.git.latest`. You'll notice that this function
has a large number of arguments. We only need to specify two:
`name="https://github.com/amfarrell/django-example"`, and `target="/vagrant/django-example"`.

So, we can write that as
```
clone-example-app:
  git.latest:
  - name: https://github.com/amfarrell/django-example
  - target: /vagrant/django-example
```
and save it to `/vagrant/salt/django.sls`. You don't need to reload salt-master
again. Just re-run your test.

You'll see a stack trace that looks like 
```
samwise:
    The minion function caused an exception: Traceback (most recent call last):
      File "/usr/lib/python2.7/dist-packages/salt/minion.py", line 1161, in _thread_return
        return_data = func(*args, **kwargs)
      File "/usr/lib/python2.7/dist-packages/salt/modules/state.py", line 296, in apply_
        return sls(mods, **kwargs)
      File "/usr/lib/python2.7/dist-packages/salt/modules/state.py", line 681, in sls
        ret = st_.state.call_high(high_)
      File "/usr/lib/python2.7/dist-packages/salt/state.py", line 2067, in call_high
        ret = dict(list(disabled.items()) + list(self.call_chunks(chunks).items()))
      File "/usr/lib/python2.7/dist-packages/salt/state.py", line 1623, in call_chunks
        running = self.call_chunk(low, running, chunks)
      File "/usr/lib/python2.7/dist-packages/salt/state.py", line 1769, in call_chunk
        self._mod_init(low)
      File "/usr/lib/python2.7/dist-packages/salt/state.py", line 612, in _mod_init
        self.states['{0}.{1}'.format(low['state'], low['fun'])]  # pylint: disable=W0106
      File "/usr/lib/python2.7/dist-packages/salt/utils/lazy.py", line 90, in __getitem__
        raise KeyError(key)
    KeyError: 'git.latest'
```
The `git` state module relies on the actual executable for git being available
on the minion, but we haven't yet installed git on samwise, so this state
module has not yet been loaded.  We've discovered a that our state has a
dependency. But we know how to satisfy it! All we need to do is install git and
we can do that with the `pkg` state module.  First, in the spirit of
test-driven-development, we update our test to check that first:

```
import os.path
import unittest
import yaml
import shutil

from test_utils import run_frodo, run_samwise, SaltStateTestCase

class TestSamwiseRepoCloned(SaltStateTestCase):

    def test_git_repo_exists(self):
        if os.path.exists('samwise_vagrant/django-example'):
            shutil.rmtree('samwise_vagrant/django-example')
        run_samwise('sudo apt-get remove git')
        state_response_data = self.run_state(state_file='django', target='samwise')
        assert 'no packages found matching git' not in run_samwise("dpkg-query -l 'git'")
        assert os.path.exists('samwise_vagrant/django-example/manage.py'), \
            "The app's git repository has not been cloned."
```
This is definitely not an isolated unit test and this function tests both that
git is installed and that our repo has been cloned. For us to achieve
isolation, our test harness would need to shut down and spin up our two-node
cluster with each test. This is possible to do, but would make our tests run
very slowly, and for what benefit? We ultimately want full-system integration
tests, so for test-driven devops, that is what we write.

Run this test and you'll see it prompts the same error, so we now update our sls file
to look like:
```
install-git:
  pkg.installed:
  - name: git

clone-example-app:
  git.latest:
  - name: https://github.com/amfarrell/django-example
  - target: /vagrant/django-example
  - require:
    - pkg: install-git
```
Notice that in addition to the `install-git` state, we added a `require`
argument to `git.latest`.  All state modules let you specify this list of states
that must succeed before this state will be attempted. Each item in the list of
requirements must be of the form "<module-name>: <state-id>". The salt-minion
uses this to sort out the order in which states are run.

With our repository cloned, the next step is to install the packages like
django that it requires to run into a python virtualenv.  This repository
includes a requirements.txt, so all we should need to do is create a virtualenv
and `pip install` the correct python modules into it. Ctrl-f search 
<a href="https://docs.saltstack.com/en/latest/ref/states/all/">the list of salt states</a>
and find a few useful candidates:
- <a href="https://docs.saltstack.com/en/latest/ref/states/all/salt.states.virtualenv_mod.html">
  This state module for virtualenv</a>
- <a href="https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.pip.html">
  This state module for pip</a>
- And of course, our old friend <a href="https://docs.saltstack.com/en/latest/ref/states/all/salt.states.pkg.html">
  pkg</a>. We could use pkg to install the ubuntu packages for both virtualenv and pip.

If you look at the docs for virtualenv state module, you'll notice two things:
1) The name of the module that the docs display is virtualenv_mod. You should
still refer to it in sls files as `virtualenv` in your state file, as the example
at the bottom of the docs shows. Several state modules have "mod" or "_mod" added
to their names, which you can see if you visit 
<a href="https://github.com/saltstack/salt/tree/develop/salt/states">the source code
for all of the modules</a> and ctrl-f search for "mod.py".

2) `virtualenv.managed` lets you pass an argument `requirements` argument that
is the path to a requirements.txt file. So, we should be able to use pkg to
install virtualenv and then use virutalenv to create our environment, skipping
pip.

Here's a test
```
class TestSamwiseVenv(SaltStateTestCase):

    def test_virtualenv_exists(self):
        if self.debian_package_installed('python-virtualenv'):
            run_samwise('sudo apt-get remove python-virtualenv')
        if os.path.exists('samwise_vagrant/example-venv/bin/'):
            shutil.rmtree('samwise_vagrant/example-venv')
        state_response_data = self.run_state(state_file='django', target='samwise')
        assert self.debian_package_installed('python-virtualenv')
        assert os.path.exists('samwise_vagrant/example-venv/bin/')
        assert os.path.exists('samwise_vagrant/example-venv/bin/django-admin')
        assert os.path.exists('samwise_vagrant/example-venv/bin/gunicorn')
```
Which you can run with `py.test tests/test_states_django.py::TestSamwiseVenv`

And here is a pair of salt states
```
install-virtualenv:
  pkg.installed:
  - name: python-virtualenv

create-virtualenv:
  virtualenv.managed:
  - name: /vagrant/example-venv
  - requirements: /vagrant/django-example/requirements.txt
  - require:
    - pkg: install-virtualenv
    - git: clone-example-app # requirements.txt must be present
```
Note that we have a dependency on the clone-example-app. This means that
our test will fail if that state fails; the state we are currently testing
will also fail because requirements.txt will be absent. Based on this, you
could choose to add all of these test conditions to the earlier test. However,
as the number of states increased, this single test would continue to grow,
making the test take longer to run.

{#
Note also that we are encoding information about the contents of requirements.txt
here. I'm actually not sure if this is the best idea. I should get feedback on this
from someone else. A problem is that there is literally no way to check that the
environment for an app is correctly set up without checking for the packages
that it requires.
#}

You should be able to see the installed app. Open up a terminal on samwise (or
re-use the one that has been viewing the logfile this whole time), `cd /vagrant/django-example` and run
`/vagrant/example-venv/bin/gunicorn --bind 0.0.0.0:8080 django_example.wsgi:application`. Then, open your browser and visit http://0.0.0.0:8082 and you'll see the app running.

{{ picture of app }}

In our next section we're going to use salt pillars to store the configuration
files we need to get gunicorn running as a daemon process.
Run `./test.sh` and `git checkout step-7`

Step 7: Jinja Templates
=======================

Now that our app is started, how can we set it running automatically?
One way, which I don't recommend is to use the `cmd.run` module to do so from salt.
{{ section on cmd.run }}

So, since that doesn't work, we need another process monitoring tool. We are
going to go with upstart, the same tool we have been using to monitor and
restart salt itself. Writing upstart scripts is beyond the scope of this tutorial, but
I encourage you to look at the examples you find in /etc/init, including the one for
`salt-minion` and `salt-master`. For now, copy-paste the following to /etc/init/django-example on samwise:

```
description "Gunicorn running a django process."

start on runlevel [2345]
stop on runlevel [!2345]

respawn
setuid vagrant
setgid vagrant
chdir /vagrant/django-example/

exec /vagrant/example-venv/bin/gunicorn --bind 0.0.0.0:8080 django_example.wsgi:application
```
Run `sudo init-checkconf /etc/init/django-example.conf` to check for syntax errors, then run `sudo service django-example start`.
It should respond with something like `django-example start/running, process 11011`.
If it does not, check /var/log/upstart/django-example.log for error messages.

{# Mostly copied from https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-14-04. Not quite sure how to credit the author though. Probably blurb at the bottom? #}

Again, visit http://0.0.0.0:8082 and you'll see our example app running. So how
do we put this file in place with salt? We can place it alongside our sls file at `frodo_vagrant/salt/django-example.conf` and then
it will be accessable from sls files as `salt://django-example.conf`
We'll then use the `file.managed` state function to put it in its place and `service.running` to tell upstart to start and manage it. Here is our test:
```
   def test_gunicorn(self):
        run_samwise('sudo service django-example stop')
        run_samwise('sudo rm -f /etc/init/django-example.conf')
        state_response_data = self.run_state(state_file='django', target='samwise')
        assert 'django-example.conf' in run_samwise('sudo ls /etc/init/django-example.conf')
        assert 'syntax ok' in run_samwise('sudo init-checkconf /etc/init/django-example.conf')
        assert 'django-example.log' in run_samwise('sudo ls /var/log/upstart/django-example.log')
        assert 'django-example start/running' in run_samwise('sudo service django-example status')
```

And here are the states to create the file:
```
gunicorn-upstart-file:
  file.managed:
  - name: /etc/init/django-example.conf
  - source: salt://django-example.conf
  - user: root    #Match user, group, & permissions with other files in /etc/init.
  - group: root
  - mode: '0644'

gunicorn-running:
  service.running:
  - name: django-example
  - require:
    - file: gunicorn-upstart-file
    - virtualenv: create-virtualenv
```
Run this, and you'll see it places a file at /etc/init/django-example.conf and starts our service.

This works, but you should notice something unfortunate. We've now put the
name 'django-example' multiple places, both in django.sls and in django-example.conf.
This violates the `Don't Repeat Yourself` principle and will make this code less maintainable.
We need to move this into a variable and then to inject that value into both the salt state
and the upstart script. Salt lets us do this by interpreting every sls file as a jinja template.
    <aside>
    You may want to read more about about the
    <a href="http://jinja.pocoo.org/docs/dev/templates/">
    syntax for jinja templates</a>
    </aside>
It is possible to set values of template variables directly like so:
```
{% set virtualenv_path = '/vagrant/example-venv' %}
{% set django_app_path = '/vagrant/django-example' %}
{% set django_app_name = 'django_example' %}
{% set gunicorn_service_name = 'django-example' %}

install-git:
  pkg.installed:
  - name: git

clone-example-app:
  git.latest:
  - name: https://github.com/amfarrell/django-example
  - target: {{ django_app_path }}
  - require:
    - pkg: install-git

install-virtualenv:
  pkg.installed:
  - name: python-virtualenv

create-virtualenv:
  virtualenv.managed:
  - name: {{ virtualenv_path }}
  - requirements: {{ django_app_path }}/requirements.txt
  - require:
    - pkg: install-virtualenv
    - git: clone-example-app

gunicorn-upstart-file:
  file.managed:
  - name: /etc/init/{{ gunicorn_service_name }}.conf
  - source: salt://django-example.conf
  - user: root
  - group: root
  - mode: '0644'
  - template: jinja
  - context:
    django_app_path: {{ django_app_path }} # yaml dict; Note the lack of hyphens.
    virtualenv_path: {{ virtualenv_path }}
    django_app_name: {{ django_app_name }}
```
Note that we now explicitly tell file.managed that django-example.conf is a jinja template
and pass it a a yaml dictionary as context. This allows us to replace the hard-coded paths
in our config file with jinja-injected variables.
```
description "Gunicorn running a django process."

start on runlevel [2345]
stop on runlevel [!2345]

respawn
setuid vagrant
setgid vagrant
chdir {{ django_app_path }}

exec {{ virtualenv_path }}/bin/gunicorn --bind 0.0.0.0:8080 {{ django_app_name }}.wsgi:application
```
This time, run `sudo salt 'samwise' state.sls_id gunicorn-upstart-file django` to run just one
state. That way, you can then inspect django-example.conf or run  `sudo init-checkconf` on it.

    <aside>A word of warning about `state.sls_id`: If you try to run a state such as `git.latest`
    and git is not already installed, it will fail before finding a state that installs git.</aside>

And finally, run ./test.sh and you'll see our state tests still pass.


{# show the use of variables demonstrated here
https://docs.saltstack.com/en/latest/topics/best_practices.html#variable-flexibility
#}

Step 8: Highstate: Enforcing multiple SLS files.
================================================

Gunicorn is not built to serve an application to the outside world. For exposing
out app on the main http port 80, the gunicorn developers <a
href="http://gunicorn.org/index.html#deployment">advise putting it behind
nginx</a>, which is just what we're going to do. However, we notice that 
out sls file is growing rather large. Deploying nginx is not strictly
related to running a django app and we may want to allow it to serve other
other applications besides this one django example. Now is a good time to split 
off to a new sls file.

First, we install nginx and ensure it is running
```
install-nginx:
  pkg.installed:
  - name: nginx

nginx-running:
  service.running:
  - name: nginx
```
save that to frodo_vagrant/salt/nginx.sls and run `sudo 'samwise' state.sls
nginx`.  We mapped port 80 on nginx to port 8002 on our host, so visit
http://0.0.0.0:8002 and you'll see the nginx default welcome screen

{{ screenshot of "welcome to nginx" }}

    <aside>If you don't, take a look at the nginx default log file on samwise named in /etc/nginx/nginx.conf: /var/log/nginx/error.log.</aside>

Now we'll add our nginx config file as frodo_vagrant/salt/nginx-default.conf
```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    server_name 0.0.0.0;

    location / {
        proxy_pass http://0.0.0.0:8080;
    }
}
```
And add a state to manage that file to nginx.sls
```
nginx-conf-file:
  file.managed:
  - name: /etc/nginx/sites-enabled/default
  - source: salt://nginx-default.conf
```

again, run `sudo 'samwise' state.sls nginx`.
But if you go to http://0.0.0.0:8002, you will see the same screen.
Run `sudo 'samwise' state.sls django` just to be sure, and you see the same.

What gives?

We started the nginx service before we had written our configuration file.
When we enforced the states in this sls file again, the service.running state
saw that nginx was already running and did not restart it. We need to have
service.running watch another state, which we can do with the `- watch` argument,
which works much like the `require` argument. Our nginx.sls now becomes

```
install-nginx:
  pkg.installed:
  - name: nginx

nginx-running:
  service.running:
  - name: nginx
  - watch:
    - file: nginx-conf-file

nginx-conf-file:
  file.managed:
  - name: /etc/nginx/sites-enabled/default
  - source: salt://nginx-default.conf
```
Now, any change to nginx-default.conf will cause `nginx-running` to be
restarted.  On frodo, run `echo '#' >> /vagrant/salt/nginx-default.conf` and
again `sudo 'samwise' state.sls nginx` and check
<a href="http://0.0.0.0:8002">http://0.0.0.0:8002</a> in your browser.
You'll see the Django page.

    <extra-credit>
    Our gunicorn service currently won't reload if we update its conf file or if we
    change the code of our django service.  fix this by adding `- watch` arguments
    to the relevant states in `django.sls`
    </extra-credit>

Configuring our system now requires we run two commands, one for each sls file.
This is inconvenient We need to create a state that ties together multiple
smaller states and enforces them in together. Saltstack calls this a
"highstate". Think like the ancient high kings of Ireland that ruled over other
kings, each one ruling a piece of the same system, er, Island. Okay, its not that
good of a mnumonic. So how do you create one?

On frodo, create a file at `/vagrant/salt/top.sls` with the contents:

```
base:
  '*':
    - ag
  'samwise':
    - django
    - nginx
```
Then run `sudo salt '*' state.highstate`. This will tell the salt-minions on both frodo and samwise to
enforce `ag.sls` and tells the one one samwise to also enforce `django.sls` and `nginx.sls`.

Step 9: Pillars
===============
In creating multiple sls files, we have also violated DRY. The port number where
gunicorn serves our app and nginx proxies to it should be saved to a variable,
this time one accessable to multiple files. The salt-master stores this data in
a "pillar file". On frodo, Open /etc/salt/master on frodo again, this time to
lines 524-526 and set them to
```
pillar_roots:
  - base:
    - /vagrant/pillar
```
Create that directory and then run `sudo service salt-master restart` to force
salt-master to reload its new config file.

Next, create a file at /vagrant/pillar/ports.sls with contents:
```
django-example-port: '8080'
```
Also create a file /vagrant/pillar/top.sls with contents:
```
base:
  '*':
    - ports
```
Check that the salt-minions get access to the pillar data by running
`sudo salt '*' pillar.items` and you should see
```
samwise:
    ----------
    django-example-port:
        8080
frodo:
    ----------
    django-example-port:
        8080
```
We can now edit the templates for /vagrant/salt/django-example.conf and /vagrant/salt/nginx-default.conf
to
```
description "Gunicorn running a django process."

start on runlevel [2345]
stop on runlevel [!2345]

respawn
setuid vagrant
setgid vagrant
chdir {{ django_app_path }}

exec {{ virtualenv_path }}/bin/gunicorn --bind 0.0.0.0:{{ pillar['django-example-port'] }} {{ django_app_name}}.wsgi:application
```
and 
```
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        server_name 0.0.0.0;

        location / {
                proxy_pass http://0.0.0.0:{{ pillar['django-example-port'] }};
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Port 8002;
        }
}
```
Run `sudo salt '*' state.highstate`, and you'll see...
Well, you'll see that things don't change. But now,
they are more maintainable.

    <extra-credit>Proxying via a socket is faster and more secure than proxying via port.
    Rewrite the gunicon config and nginx config to do that instead.</extra-credit>


Step 10: Grains: Filling in salt state templates with server data
=================================================================
Our states also hard-code another assumption: that this service will
be deployed

{# TODO: put this in earlier in the tutorial. #}

Step 11: Scaling with vagrant providers
=======================================
