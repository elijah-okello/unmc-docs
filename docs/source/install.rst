Installation 
============


.. _installation:


Deploying UNMC requires a kubernetes cluster that is setup to run the
system workloads. This documentation shows how to setup UNMC on bare
metal virtual machines in this case provided by `NITA-U`_ .

.. _NITA-U: https://www.nita.go.ug/

The following steps are required to deploy the system.

-  Setup VPN access to NITA-U servers
-  Get access to 4 virtual machines. Ensure that you have ``ssh`` access
   to all of them with sudo privileges.
-  Setup a 3 node kubernetes cluster with 3 VMs.
-  Configure ingress-nginx as the load balancer
-  Configure ``kubectl`` to point to your cluster’s context
-  Configure the 1TB storage VM as the fileserver and database server

   -  SSL
   -  Database Setup
   -  Nginx
   -  Deploying ``unmc-django-upload`` service

VPN access to NITA-U servers
----------------------------

NITA-U restricts access to their datacenter resources to VPN access. So
you must configure access to the servers via VPN. Use these instructions
to follow through how to set up the VPN. `VPN
Documentation <https://drive.google.com/file/d/14SpKo9qZqv9d-BoaedUwH1FPIMVERSoE/view?usp=sharing>`__

Get access to 4 virtual machines.
---------------------------------

Reach out to NITA-U (or any other cloud provider) and request them to avail 4 virtual machines that
can be accessed via ssh with ``sudo``. For our setup we were sent this
`information <https://drive.google.com/file/d/1eVnlGbCqjW_5U1B3KnmFgaro-Dd9bOOq/view?usp=sharing>`__
which contains information about the virtual machines and how to ssh
into them

Setup a 3 node kubernetes cluster with 3 VMs using kubeadm on Ubuntu 20.04.
---------------------------------------------------------------------------

`Kubernetes <https://kubernetes.io/>`__ is a container orchestration
system that manages containers at scale. Initially developed by Google
based on its experience running containers in production, Kubernetes is
open source and actively developed by a community around the world.

.. note:: 

   Note: This tutorial uses version 1.22 of Kubernetes.

Kubeadm automates the installation and configuration of Kubernetes
components such as the API server, Controller Manager, and Kube DNS. It
does not, however, create users or handle the installation of
operating-system-level dependencies and their configuration. For these
preliminary tasks, it is possible to use a configuration management tool
like Ansible or SaltStack. Using these tools makes creating additional
clusters or recreating existing clusters much simpler and less error
prone.

In this guide, you will set up a Kubernetes cluster from scratch using
Ansible and Kubeadm, and then deploy ``UNMC`` to it.

Your cluster will include the following physical resources:

-  One master node

   -  The master node (a node in Kubernetes refers to a server) is
      responsible for managing the state of the cluster. It runs Etcd,
      which stores cluster data among components that schedule workloads
      to worker nodes.

-  Two worker nodes

   -  Worker nodes are the servers where your workloads
      (i.e. containerized applications and services) will run. A worker
      will continue to run your workload once they’re assigned to it,
      even if the master goes down once scheduling is complete. A
      cluster’s capacity can be increased by adding workers.

Prerequisites
-------------

-  Ansible installed on your local machine. If you’re running Ubuntu
   18.04 as your OS, follow the “Step 1 - Installing Ansible” section in
   `How to Install and Configure Ansible on Ubuntu 18.04 to install
   Ansible. <https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-18-04#step-1-—-installing-ansible>`__
   For installation instructions on other platforms like macOS or
   CentOS, follow the official `Ansible installation
   documentation. <http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-the-control-machine>`__

-  Familiarity with Ansible playbooks. For review, check out
   `Configuration Management 101: Writing Ansible
   Playbooks. <https://www.digitalocean.com/community/tutorials/configuration-management-101-writing-ansible-playbooks>`__

-  Knowledge of how to launch a container from a Docker image. Look at
   “Step 5 — Running a Docker Container” in `How To Install and Use
   Docker on Ubuntu 18.04 if you need a
   refresher. <https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04#step-5-—-running-a-docker-container>`__

Step 1 — Setting Up the Workspace Directory and Ansible Inventory File
----------------------------------------------------------------------

In this section, you will create a directory on your local machine that
will serve as your workspace. You will configure Ansible locally so that
it can communicate with and execute commands on your remote servers.
Once that’s done, you will create a ``hosts`` file containing inventory
information such as the IP addresses of your servers and the groups that
each server belongs to.

Out of your three servers, one will be the master with an IP displayed
as ``control_plane_ip``. The other two servers will be workers and will
have the IPs ``worker_1_ip`` and ``worker_2_ip``.

Create a directory named ``~/kube-cluster`` in the home directory of
your local machine and ``cd``\ into it:

.. code:: sh

       $ mkdir ~/kube-cluster
       $ cd ~/kube-cluster

This directory will be your workspace for the rest of the tutorial and
will contain all of your Ansible playbooks. It will also be the
directory inside which you will run all local commands.

Create a file named ``~/kube-cluster/hosts`` using ``nano`` or your
favorite text editor:

.. code:: bash

       $ nano ~/kube-cluster/hosts

Add the following text to the file, which will specify information about
the logical structure of your cluster:

::

       file: ~/kube-cluster/hosts

       [control_plane]
       control1 ansible_host=control_plane_ip ansible_user=user1 

       [workers]
       worker1 ansible_host=worker_1_ip ansible_user=user1
       worker2 ansible_host=worker_2_ip  ansible_user=user1

       [all:vars]
       ansible_python_interpreter=/usr/bin/python3

You may recall that `inventory
files <http://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html>`__
in Ansible are used to specify server information such as IP addresses,
remote users, and groupings of servers to target as a single unit for
executing commands. ``~/kube-cluster/hosts`` will be your inventory file
and you’ve added two Ansible groups (masters and workers) to it
specifying the logical structure of your cluster.

In the masters group, there is a server entry named “master” that lists
the master node’s IP (``control_plane_ip``) and specifies that Ansible
should run remote commands as the root user.

Similarly, in the workers group, there are two entries for the worker
servers (``worker_1_ip`` and ``worker_2_ip``) that also specify the
``ansible_user`` as ``nita-u-user``.

.. note:: 

   Note: The ``nita-u-user`` is the user that NITA-U will have
   provisioned. In this case it will be ``user1`` since it is what was
   provided to us in the `server
   docs <https://drive.google.com/file/d/1eVnlGbCqjW_5U1B3KnmFgaro-Dd9bOOq/view?usp=sharing>`__

The last line of the file tells Ansible to use the remote servers’
``Python 3`` interpreters for its management operations.

Save and close the file after you’ve added the text.

::

   ctrl + x

Having set up the server inventory with groups, let’s move on to
installing operating system level dependencies and creating
configuration settings.

Setup Authentication for the VMS
--------------------------------

In this section you will setup the required files for ansible to be able
to authenticate with the servers and perform tasks on the servers like
installing dependencies and changing settings.

Create two files in your ``~/kube-cluster/`` folder called ``password``
and ``sudopassword`` and put the password for the ``ssh`` user in those
files.

.. code:: bash

       $ echo "myverysecurepass" >> password
       $ echo "myverysecurepass" >> sudopassword

Ansible will use the ``password`` file contents for ssh authentication
and the ``sudopassword`` file contents for privilege escalation during
installation of dependencies and configuration management.

   It is okay if the contents of the two files are the same. As they
   were the same at the time of bootstraping the cluster.

Now that the preliminary setup is done. Let us install the kubernetes
dependencies.

Installing Kubernetetes Dependencies
-------------------------------------

In this section, you will install the operating-system-level packages
required by Kubernetes with Ubuntu’s package manager. These packages
are:

-  Docker - a container runtime. It is the component that runs your
   containers. Support for other runtimes such as rkt is under active
   development in Kubernetes.
-  ``kubeadm`` - a CLI tool that will install and configure the various
   components of a cluster in a standard way.
-  ``kubelet`` - a system service/program that runs on all nodes and
   handles node-level operations.
-  ``kubectl`` - a CLI tool used for issuing commands to the cluster
   through its API Server.

Create a file named ``~/kube-cluster/kube-dependencies.yml`` in the
workspace:

::

    $ nano ~/kube-cluster/kube-dependencies.yml

Add the following plays to the file to install these packages to your
servers:

.. code:: yaml

   file: ~/kube-cluster/kube-dependencies.yml
   ---
   - hosts: all
   become: yes
   tasks:
   - name: create Docker config directory
       file: path=/etc/docker state=directory

   - name: changing Docker to systemd driver
       copy:
       dest: "/etc/docker/daemon.json"
       content: |
           {
           "exec-opts": ["native.cgroupdriver=systemd"]
           }

   - name: install Docker
       apt:
       name: docker.io
       state: present
       update_cache: true

   - name: install APT Transport HTTPS
       apt:
       name: apt-transport-https
       state: present

   - name: add Kubernetes apt-key
       apt_key:
       url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
       state: present

   - name: add Kubernetes' APT repository
       apt_repository:
       repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
       state: present
       filename: 'kubernetes'

   - name: install kubelet
       apt:
       name: kubelet=1.22.4-00
       state: present
       update_cache: true

   - name: install kubeadm
       apt:
       name: kubeadm=1.22.4-00
       state: present

   - hosts: control_plane
   become: yes
   tasks:
   - name: install kubectl
       apt:
       name: kubectl=1.22.4-00
       state: present
       force: yes

The first play in the playbook does the following:

-  Installs Docker, the container runtime.
-  Installs apt-transport-https, allowing you to add external HTTPS
   sources to your APT sources list.
-  Adds the Kubernetes APT repository’s apt-key for key verification.
-  Adds the Kubernetes APT repository to your remote servers’ APT
   sources list.
-  Installs kubelet and kubeadm.

The second play consists of a single task that installs kubectl on your
master node.

Save and close the file when you are finished.

Next, execute the playbook by locally running:

.. code:: sh

       ansible-playbook -i hosts --conn-pass-file password --become-pass-file sudo_password ~/kube-cluster/kube-dependencies.yml

This command should be executed at the root directory of
``~/kube-cluster/`` such that all the files you created can be picked up
by ``ansible``

On completion, the output should be like this

::

   Output
   PLAY [all] ****

   TASK [Gathering Facts] ****
   ok: [worker1]
   ok: [worker2]
   ok: [master]

   TASK [install Docker] ****
   changed: [master]
   changed: [worker1]
   changed: [worker2]

   TASK [install APT Transport HTTPS] *****
   ok: [master]
   ok: [worker1]
   changed: [worker2]

   TASK [add Kubernetes apt-key] *****
   changed: [master]
   changed: [worker1]
   changed: [worker2]

   TASK [add Kubernetes' APT repository] *****
   changed: [master]
   changed: [worker1]
   changed: [worker2]

   TASK [install kubelet] *****
   changed: [master]
   changed: [worker1]
   changed: [worker2]

   TASK [install kubeadm] *****
   changed: [master]
   changed: [worker1]
   changed: [worker2]

   PLAY [master] *****

   TASK [Gathering Facts] *****
   ok: [master]

   TASK [install kubectl] ******
   ok: [master]

   PLAY RECAP ****
   master                     : ok=9    changed=5    unreachable=0    failed=0   
   worker1                    : ok=7    changed=5    unreachable=0    failed=0  
   worker2                    : ok=7    changed=5    unreachable=0    failed=0  

After execution, Docker, ``kubeadm``, and ``kubelet`` will be installed
on all of the remote servers. ``kubectl`` is not a required component
and is only needed for executing cluster commands. Installing it only on
the master node makes sense in this context, since you will run
``kubectl`` commands only from the master. Note, however, that
``kubectl`` commands can be run from any of the worker nodes or from any
machine where it can be installed and configured to point to a cluster.

All system dependencies are now installed. Let’s set up the master node
and initialize the cluster.

Setting Up the Master Node
--------------------------

In this section, you will set up the master node. Before creating any
playbooks, however, it’s worth covering a few concepts such as Pods and
Pod Network Plugins, since your cluster will include both.

A pod is an atomic unit that runs one or more containers. These
containers share resources such as file volumes and network interfaces
in common. Pods are the basic unit of scheduling in Kubernetes: all
containers in a pod are guaranteed to run on the same node that the pod
is scheduled on.

Each pod has its own IP address, and a pod on one node should be able to
access a pod on another node using the pod’s IP. Containers on a single
node can communicate easily through a local interface. Communication
between pods is more complicated, however, and requires a separate
networking component that can transparently route traffic from a pod on
one node to a pod on another.

This functionality is provided by pod network plugins. For this cluster,
you will use `Flannel <https://github.com/coreos/flannel>`__, a stable
and performant option.

Create an Ansible playbook named ``control-plane.yml`` on your local
machine:

.. code:: bash

       $ nano ~/kube-cluster/master.yml

Add the following play to the file to initialize the cluster and install
Flannel:

.. code:: yaml

       file: ~/kube-cluster/master.yml
       ---
       - hosts: control_plane
       become: yes
       tasks:
           - name: disable swap
             shell: swapoff -a

           - name: initialize the cluster
             shell: kubeadm init --pod-network-cidr=10.244.0.0/16 >> cluster_initialized.txt
           args:
               chdir: $HOME


           - name: create .kube directory
             become: yes
             become_user: user1
             file:
               path: $HOME/.kube
               state: directory
               mode: 0755

           - name: copy admin.conf to user's kube config
             copy:
               src: /etc/kubernetes/admin.conf
               dest: /home/user1/.kube/config
               remote_src: yes
               owner: user1

           - name: install Pod network
             become: yes
             become_user: user1
             shell: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml >> pod_network_setup.txt
             args:
               chdir: $HOME
               creates: pod_network_setup.txt

Here’s a breakdown of this play:

-  The first task initializes the cluster by running ``kubeadm init``.
   Passing the argument ``--pod-network-cidr=10.244.0.0/16`` specifies
   the private subnet that the pod IPs will be assigned from. Flannel
   uses the above subnet by default; we’re telling ``kubeadm`` to use
   the same subnet.
-  The second task creates a ``.kube`` directory at ``/user1/ubuntu``.
   This directory will hold configuration information such as the admin
   key files, which are required to connect to the cluster, and the
   cluster’s API address.
-  The third task copies the ``/etc/kubernetes/admin.conf`` file that
   was generated from ``kubeadm init`` to your non-root user’s home
   directory. This will allow you to use ``kubectl`` to access the
   newly-created cluster.
-  The last task runs ``kubectl apply`` to install ``Flannel``.
   ``kubectl apply -f descriptor.[yml|json]`` is the syntax for telling
   ``kubectl`` to create the objects described in the
   ``descriptor.[yml|json]`` file. The ``kube-flannel.yml`` file
   contains the descriptions of objects required for setting up
   ``Flannel`` in the cluster.

Save and close the file when you are finished.

Execute the playbook locally by running:

.. code:: sh

       $ ansible-playbook -i hosts --conn-pass-file password --become-pass-file sudo_password ~/kube-cluster/control-plane.yml

On completion, you will see output similar to the following:

::

   Output

   PLAY [master] ****

   TASK [Gathering Facts] ****
   ok: [master]

   TASK [initialize the cluster] ****
   changed: [master]

   TASK [create .kube directory] ****
   changed: [master]

   TASK [copy admin.conf to user's kube config] *****
   changed: [master]

   TASK [install Pod network] *****
   changed: [master]

   PLAY RECAP ****
   master                     : ok=5    changed=4    unreachable=0 

To check the status of the master node, SSH into it with the following
command:

::

   $ ssh ubuntu@control_plane_ip

Once inside the master node, execute:

::

   $ kubectl get nodes 

You will now see the following output

::

   Output
   NAME      STATUS    ROLES     AGE       VERSION
   master    Ready     master    1s        v1.14.0

The output states that the ``master`` node has completed all
initialization tasks and is in a ``Ready`` state from which it can start
accepting worker nodes and executing tasks sent to the API Server. You
can now add the workers from your local machine.

Setting Up the Worker Nodes
---------------------------

Adding workers to the cluster involves executing a single command on
each. This command includes the necessary cluster information, such as
the IP address and port of the master’s API Server, and a secure token.
Only nodes that pass in the secure token will be able join the cluster.

Navigate back to your workspace and create a playbook named
``workers.yml``:

::

   $ nano ~/kube-cluster/workers.yml

Add the following text to the file to add the workers to the cluster:

.. code:: yaml

       file: ~/kube-cluster/workers.yml
       ---
       - hosts: control_plane
       become: yes
       gather_facts: false
       tasks:
           - name: get join command
           shell: kubeadm token create --print-join-command
           register: join_command_raw

           - name: set join command
           set_fact:
               join_command: "{{ join_command_raw.stdout_lines[0] }}"


       - hosts: workers
       become: yes
       tasks:
           - name: disable swap
           shell: swapoff -a

           - name: join cluster
           shell: "{{ hostvars['control1'].join_command }} --ignore-preflight-errors FileAvailable--etc-kubernetes-pki-ca.crt >> node_joined3.txt"
           args:
               chdir: $HOME
               creates: node_joined3.txt

Here’s what the playbook does:

-  The first play gets the join command that needs to be run on the
   worker nodes. This command will be in the following
   format:``kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>``.
   Once it gets the actual command with the proper token and hash
   values, the task sets it as a fact so that the next play will be able
   to access that info.
-  The second play has two tasks that run disabling swap and the join
   command on all worker nodes. On completion of this task, the two
   worker nodes will be part of the cluster.

Save and close the file when you are finished.

Execute the playbook by locally running:

.. code:: bash

       $ ansible-playbook -i hosts --conn-pass-file password --become-pass-file sudo_password ~/kube-cluster/workers.yml

On completion, you will see output similar to the following:

::

   Output
   PLAY [master] ****

   TASK [get join command] ****
   changed: [master]

   TASK [set join command] *****
   ok: [master]

   PLAY [workers] *****

   TASK [Gathering Facts] *****
   ok: [worker1]
   ok: [worker2]

   TASK [join cluster] *****
   changed: [worker1]
   changed: [worker2]

   PLAY RECAP *****
   master                     : ok=2    changed=1    unreachable=0    failed=0   
   worker1                    : ok=2    changed=1    unreachable=0    failed=0  
   worker2                    : ok=2    changed=1    unreachable=0

With the addition of the worker nodes, your cluster is now fully set up
and functional, with workers ready to run workloads. Before scheduling
applications, let’s verify that the cluster is working as intended.

Verifying the Cluster
---------------------

A cluster can sometimes fail during setup because a node is down or
network connectivity between the control plane and workers is not
working correctly. Let’s verify the cluster and ensure that the nodes
are operating correctly.

You will need to check the current state of the cluster from the control
plane node to ensure that the nodes are ready. If you disconnected from
the control plane node, you can SSH back into it with the following
command:

.. code:: bash

   $ ssh ubuntu@control_plane_ip

Then execute the following command to get the status of the cluster:

.. code:: bash

   kubectl get nodes

You will see output similar to the following:

.. code:: bash

   Output
   NAME     STATUS   ROLES                  AGE     VERSION
   control1   Ready    control-plane,master   3m21s   v1.22.0
   worker1  Ready    <none>                 32s     v1.22.0
   worker2  Ready    <none>                 32s     v1.22.0

If all of your nodes have the value ``Ready`` for ``STATUS``, it means
that they’re part of the cluster and ready to run workloads.

If, however, a few of the nodes have ``NotReady`` as the ``STATUS``, it
could mean that the worker nodes haven’t finished their setup yet. Wait
for around five to ten minutes before re-running ``kubectl get nodes``
and inspecting the new output. If a few nodes still have ``NotReady`` as
the status, you might have to verify and re-run the commands in the
previous steps.

Now that your cluster is verified successfully, let’s schedule an
example Nginx application on the cluster.

Accessing the cluster from your machine
---------------------------------------

Maker sure you have ``kubectl``
`installed <https://kubernetes.io/docs/tasks/tools/>`__ on your local
machine.

To access the cluster from your local machine, you need to get the
contents of the ``~/.kube/config`` file in the control plane and add it
to your local machine’s ``~/.kube/config`` file

An exampple ``config`` file looks like this

.. code:: yaml

   apiVersion: v1
   clusters:
     - cluster:
         certificate-authority-data: I3URHI3UR324IH3U4HI2UH2IUH52984U52893458923==
         server: https://11.252.6.18:6443
       name: kubernetes
   contexts:
     - context:
         cluster: kubernetes
         user: kubernetes-admin
       name: kubernetes-admin@kubernetes
   current-context: kubernetes-admin@kubernetes
   kind: Config
   preferences: {}
   users:
     - name: kubernetes-admin
       user:
         client-certificate-data: rb3ur3ur23y324uy234y32==
         client-key-data: njwnfurfb4u413xxxxxded2u==

It contains the following.

-  ``apiVersion``
-  ``clusters`` which is a list of all clusters that can be accessed by
   your ``kubectl`` installation
-  ``contexts`` which is a corresponding list of cluster contexts
-  ``current-context`` which is the current context that is queried when
   you run kubectl
-  ``kind`` is the type of kubernetes resource which is ``config``
-  ``preferences``
-  ``users`` which is a list of users that can authenticate onto a
   particular cluster with ``name``

After getting either by manually copying it or downloading the
``~/.kube/config`` file from the ``control_plane``, open it in your
favorite text editor and copy the sections one by one as you append it
to the ``~/.kube/config`` file that is on your local machine.

Be careful to maintain the indentation as ``yaml`` is strict on
indentation.

Replace the value of ``current-context`` with the name of your cluster
which should correspond to the name you supplied in your entry under
``contexts.``

Verifying that you have access.
-------------------------------

To verify that you have access to your cluster, run the following
command.

.. code:: bash

   kubectl get nodes

You should see the following output

.. code:: bash

   Output
   NAME     STATUS   ROLES                  AGE     VERSION
   control1   Ready    control-plane,master   3m21s   v1.22.0
   worker1  Ready    <none>                 32s     v1.22.0
   worker2  Ready    <none>                 32s     v1.22.0

Deploying UNMC to the cluster.
--------------------------------

Ingress-nginx setup
---------------------

The UNMC system uses ``ingress-nginx`` for routing requests between the
services.

To install ``ingress-nginx``, run the following ``kubectl`` command

.. code:: bash

   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.1/deploy/static/provider/baremetal/deploy.yaml

This should work on almost every cluster, but it will typically use a
port in the range 30000-32767.

To deploy UNMC to the cluster, you need to have the source code on your
local machine. Clone it using this command.

Make sure to add your ``public key`` to the repository before you clone.

.. code:: bash

   git clone git@gitlab.outbox.co.ug:uganda-nurses-and-midwives/unmc-k8s.git

Navigate into the directory

.. code:: bash

   cd unmc-k8s

Then deploy using skaffold. If you don’t have skaffold install it from
`here <https://skaffold.dev/docs/install/>`__

.. code:: bash

   skaffold run

This command will build all the images afresh and deploy everything to
the new cluster.

Configure Container Registry
------------------------------

`Link to
Article <https://medium.com/hackernoon/today-i-learned-pull-docker-image-from-gcr-google-container-registry-in-any-non-gcp-kubernetes-5f8298f28969>`__

Create a secret for all namespaces

.. code:: bash

   kubectl create secret docker-registry gcr-json-key --docker-server=us.gcr.io --docker-username=_json_key --docker-password="$(cat /Users/elijahokello/Downloads/outboxwebsite-12d67f7d7aa8.json)" --docker-email=elijah.okello@outbox.co.ug 

Patch the image pull secret.

.. code:: bash

       kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'

Set environment variable for GOOGLE_APPLICATION_CREDENTIALS in the
deployment manifest

Setup Postgres
--------------

To allow requests from other servers Install uuid extenstion on the
postgres database server

Migrate the required database from an old db using pd_dump. orgunits-db,
product-db, Make sure the new database management system is of the same
version as the old so prevent errors.

https://www.digitalocean.com/community/tutorials/how-to-move-a-postgresql-data-directory-to-a-new-location-on-ubuntu-20-04

EXPOSING NATS
---------------

for monitoring nats kubectl expose deployment nats-depl –type=NodePort
–port 8222 –target-port 8222 For exposing nats kubectl expose deployment
nats-depl –type=NodePort –port 4222 –target-port 4222 –name nats-pub

Change NATS environment variables in unmc_django_upload server in
event_bus_root.py. These are the variables NATS_URL, NATS_CLUSTER_ID,
NATS_CLIENT_ID

-  Get the base url(private IP) and the svc port from k get svc and add
   it to the envs
-  Start the nats connection nohup python manage.py
   register_event_listeners &

DEPLOYING
-----------

Get the latest built image from GCP Container Regristry

https://console.cloud.google.com/gcr/images/hi-innovator-354112/us/unmc_react_client?project=hi-innovator-354112

Then locate the deployment and copy it’s latest image url. then run the
following commands

kubectl set image deployment/unmc-react-client-depl
unmc-react-client=us.gcr.io/hi-innovator-354112/unmc_react_clit@sha256:f4cfe6de9d22ba9a6bdc126237c2a3bdf4e6a80d824a0896f188e67f678900f1

kubectl rollout restart deployment unmc-react-client-depl

ACCESSING NATS
----------------

Connect to VPN Then visit this url
http://10.255.6.18:31160/streaming/clientsz?offset=0&subs=1

MIGRATING TO A NEW GCP Project.
---------------------------------

Obtain a service account key to be used by the builders and the logger
in the services.

Replace the service account in all places where it is used in the
services build pipeline.

Change the project ids in logger settings in express services.

OPTIMIZING GITLAB RUNNER BUILDS
---------------------------------

Open the ``config.toml`` file of the host machine for the runners. Then
change the ``concurrency`` from ``1`` to ``5``.

Email Notifier Port Opening
-----------------------------

Ask the NITA-U people to open port ``587`` of the servers. Both Ingress
and Egress

Setting up Staging environment.
---------------------------------

-  Create a ``staging`` namespace in the kubernetes cluster
-  Create all the required secrets in the staging namespace.
-  Build images and upload them to the container registry
-  Update the ``ImagePullSecret`` with the service account ``json`` key
   file to be used to authenticate with google cloud for the ``staging``
   namespace
-  The run ``kubectl apply -f k8s/`` in the ``staging-infra`` folder

Firebase Setup
----------------

-  Setup firebase project
-  Update the rules and configure the indexes

.. code:: bash

   https://console.firebase.google.com/v1/r/project/outboxwebsite/firestore/indexes?create_composite=ClVwcm9qZWN0cy9vdXRib3h3ZWJzaXRlL2RhdGFiYXNlcy8oZGVmYXVsdCkvY29sbGVjdGlvbkdyb3Vwcy9wYXltZW50UmVzcG9uc2UvaW5kZXhlcy9fEAEaCgoGdXNlcklkEAEaDQoJY3JlYXRlZEF0EAIaDAoIX19uYW1lX18QAg

KNOWN ISSUES
----------------

Google API Logging timeout
^^^^^^^^^^^^^^^^^^^^^^^^^^

-  Sometimes services may be restarted when the cloud logging api takes
   long to respond to the service
-  Sometimes services unsubscribe from nats.

Issues to fix
---------------

-  Firebase rules to allow access to firestore.

Nats URL
---------

http://10.255.6.18:31160/streaming/clientsz?offset=0&subs=1

Deploying the upload server
-----------------------------

When changes are made to the source code of the upload project, to
deploy the changes do the following

-  ssh into the file server
-  ``cd`` into the ``unmc-django-upload`` directory
-  run ``git pull origin master`` assuming that the production changes
   are in the master branch
- run ``python3 -m pip install -r requirements.txt`` to install the
  required dependencies
-  run ``sudo systemctl restart gunicorn``
-  run ``nohup python manage.py register_event_listeners &``

Logging using Google Cloud Logging Stack Driver and Fluentbit
---------------------------------------------------------------

-  Official documentation from Google Cloud

`Tutorial <https://cloud.google.com/community/tutorials/kubernetes-engine-customize-fluentbit>`__

-  This tutorial is for fluentbit setup

`Tutorial <https://docs.fluentbit.io/manual/pipeline/outputs/stackdriver>`__

