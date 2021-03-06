.. spelling::

   Quickstart
   subdirectory
   Todo
   usernames

Quickstart Guide
================
To quickly set up a testing cluster using MetalK8s_, you need 3 machines running
CentOS_ 7.4 to which you have SSH access (these can be VMs). Each machine
acting as a Kubernetes_ node (all of them, in this example) also need to have at
least one disk available to provision storage volumes.

.. todo:: Give some sizing examples

.. _MetalK8s: https://github.com/scality/metal-k8s/
.. _CentOS: https://www.centos.org
.. _Kubernetes: https://kubernetes.io

Defining an Inventory
---------------------
To tell the Ansible_-based deployment system on which machines MetalK8s should
be installed, a so-called *inventory* needs to be provided. This inventory
contains a file listing all the hosts comprising the cluster, as well as some
configuration.

.. _Ansible: https://www.ansible.com

First, create a directory, e.g. :file:`inventory/quickstart-cluster`, in which
the inventory will be stored. For our setup, we need to create two files. One
listing all the hosts, aptly called :file:`hosts`:

.. code-block:: ini

    node-01 ansible_host=10.0.0.1 ansible_user=centos
    node-02 ansible_host=10.0.0.2 ansible_user=centos
    node-03 ansible_host=10.0.0.3 ansible_user=centos

    [kube-master]
    node-01
    node-02
    node-03

    [etcd]
    node-01
    node-02
    node-03

    [kube-node]
    node-01
    node-02
    node-03

    [k8s-cluster:children]
    kube-node
    kube-master

Make sure to change IP-addresses, usernames etc. according to your
infrastructure.

In a second file, called :file:`kube-node.yml` in a :file:`group_vars`
subdirectory of our inventory, we declare how to setup storage (in the
default configuration) on hosts in the *kube-node* group, i.e. hosts on which
Pods will be scheduled:

.. code-block:: yaml

    metal_k8s_lvm:
      vgs:
        kubevg:
          drives: ['/dev/vdb']

In the above, we assume every *kube-node* host has a disk available as
:file:`/dev/vdb` which can be used to set up Kubernetes *PersistentVolumes*. For
more information about storage, see :doc:`../architecture/storage`.

Entering the MetalK8s Shell
---------------------------
To easily install a supported version of Ansible and its dependencies, as well
as some Kubernetes tools (:program:`kubectl` and :program:`helm`), we provide a
:program:`make` target which installs these in a local environment. To enter this
environment, run :command:`make shell` (this takes a couple of seconds on first
run)::

    $ make shell
    Creating virtualenv...
    Installing Python dependencies...
    Downloading kubectl...
    Downloading Helm...
    Launching metal-k8s shell environment. Run 'exit' to quit.
    (metal-k8s) $

Now we're all set to deploy a cluster::

    (metal-k8s) $ ansible-playbook -i inventory/quickstart-cluster -b metal-k8s.yml

Grab a coffee and wait for deployment to end.

Inspecting the cluster
----------------------
Once deployment finished, a file containing credentials to access the cluster is
created: :file:`inventory/quickstart-cluster/artifacts/admin.conf`. We can
export this location in the shell such that the :program:`kubectl` and
:program:`helm` tools know how to contact the cluster *kube-master* nodes, and
authenticate properly::

    (metal-k8s) $ export KUBECONFIG=`pwd`/inventory/quickstart-cluster/artifacts/admin.conf

Now, assuming port *6443* on the first *kube-master* node is reachable from your
system, we can e.g. list the nodes::

    (metal-k8s) $ kubectl get nodes
    NAME        STATUS    ROLES            AGE       VERSION
    node-01     Ready     master,node      1m        v1.9.5+coreos.0
    node-02     Ready     master,node      1m        v1.9.5+coreos.0
    node-03     Ready     master,node      1m        v1.9.5+coreos.0

or list all pods::

    (metal-k8s) $ kubectl get pods --all-namespaces
    NAMESPACE      NAME                                                   READY     STATUS      RESTARTS   AGE
    kube-ingress   nginx-ingress-controller-9d8jh                         1/1       Running     0          1m
    kube-ingress   nginx-ingress-controller-d7vvg                         1/1       Running     0          1m
    kube-ingress   nginx-ingress-controller-m8jpq                         1/1       Running     0          1m
    kube-ingress   nginx-ingress-default-backend-6664bc64c9-xsws5         1/1       Running     0          1m
    kube-ops       alertmanager-kube-prometheus-0                         2/2       Running     0          2m
    kube-ops       alertmanager-kube-prometheus-1                         2/2       Running     0          2m
    kube-ops       es-client-7cf569f5d8-2z974                             1/1       Running     0          2m
    kube-ops       es-client-7cf569f5d8-qq4h2                             1/1       Running     0          2m
    kube-ops       es-data-cd5446fff-pkmhn                                1/1       Running     0          2m
    kube-ops       es-data-cd5446fff-zzd2h                                1/1       Running     0          2m
    kube-ops       es-exporter-elasticsearch-exporter-7df5bcf58b-k9fdd    1/1       Running     3          1m
    ...

Similarly, we can list all deployed Helm_ applications::

    (metal-k8s) $ helm list
    NAME                    REVISION        UPDATED                         STATUS          CHART                           NAMESPACE
    es-exporter             3               Wed Apr 25 23:10:13 2018        DEPLOYED        elasticsearch-exporter-0.1.2    kube-ops
    fluentd                 3               Wed Apr 25 23:09:59 2018        DEPLOYED        fluentd-elasticsearch-0.1.4     kube-ops
    heapster                3               Wed Apr 25 23:09:37 2018        DEPLOYED        heapster-0.2.7                  kube-system
    kibana                  3               Wed Apr 25 23:10:06 2018        DEPLOYED        kibana-0.2.2                    kube-ops
    kube-prometheus         3               Wed Apr 25 23:09:22 2018        DEPLOYED        kube-prometheus-0.0.33          kube-ops
    nginx-ingress           3               Wed Apr 25 23:09:09 2018        DEPLOYED        nginx-ingress-0.11.1            kube-ingress
    prometheus-operator     3               Wed Apr 25 23:09:14 2018        DEPLOYED        prometheus-operator-0.0.15      kube-ops

.. _Helm: https://www.helm.sh

Access to dashboard, Grafana and Kibana
---------------------------------------
Once the cluster is running, you can access the `Kubernetes dashboard`_,
Grafana_ metrics and Kibana_ logs from your browser.

To access the Kubernetes dashboard, first create a secure tunnel into your
cluster by running ``kubectl proxy``. Then, while the tunnel is up and running,
access the dashboard at
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/.

Grafana can be accessed at :samp:`http://{node-ip}/_/grafana`, with *node-ip*
replaced by the IP-address of one of the hosts in the *kube-node* group.

Similarly, Kibana can be accessed at :samp:`http://{node-ip}/_/kibana`. When
accessing this service for the first time, set up an *index pattern* for the
``logstash-*`` index, using the ``@timestamp`` field as *Time Filter field
name*.

See :doc:`../architecture/cluster-services` for more information about these
services and their configuration.

.. _Kubernetes dashboard: https://github.com/kubernetes/dashboard
.. _Grafana: https://grafana.com
.. _Kibana: https://www.elastic.co/products/kibana/
