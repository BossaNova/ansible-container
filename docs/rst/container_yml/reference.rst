Ansible Container Compose Specification
=======================================

The orchestration document for Ansible Container is the ``container.yml`` file. It's much like a ``docker-compose.yml``
file, defining the services that make up the application, the relationships between each, and how they can be accessed
from the outside world.

However, the goal for ``container.yml`` is much more ambitious. It is to provide a single definition of the application
that builds and deploys in development and in a production cloud environment, providing one source of configuration that works
throughout the application lifecycle.

The structure and contents of the file are based on Docker Compose, supporting versions 1 and 2. Looking at a ``container.yml``
file you will recognize it as mostly Compose with a few notable additions. There are of course directives that are not
supported, because they do not translate to the supported clouds, and directives have been added to provide support for
multiple environments and multiple clouds.

The aim of this document to provide a complete reference to the contents of ``container.yml``.

.. contents:: Topics

Supported Directives
--------------------

.. |checkmark| unicode:: U+02713 .. check mark

The following tables provide a quick references to supported Compose directives. All of the Compose directives for
version 1, 2 and 2.1 are listed along with any directives added by Ansible Container. A |checkmark| in the *Supported?*
column indicates that the directive is supported. In some cases the directive name links to specific implementation notes
that provide details on how the directive is used.

Top level directives
````````````````````

===================== ======================================================== ============
Directive             Definition                                               Supported?
===================== ======================================================== ============
networks              Create named, persistent networks
:ref:`registries`     Registry definitions                                     |checkmark|
services              Services included in the app                             |checkmark|
version               Specifiy the version of Compose, '1' or '2'              |checkmark|
:ref:`volumes`        Create named, persistent volumes. The syntax differs     |checkmark|
                      from the Docker specification. View :ref:`volumes`
                      for details.
===================== ======================================================== ============

Service level directives
````````````````````````

===================== ======================================================== ============
Directive             Definition                                               Supported?
===================== ======================================================== ============
build                 Run Dockerfile based build
cap_add               Add container capabilities
cap_drop              Drop container capabilities
command               Command executed by the container at startup             |checkmark|
container_name        Custom container name
cpuset                CPUs in which to allow execution
cpu_shares            CPU shares (relative weight)
cpu_quota             Limit the CPU CFS (Completely Fair Scheduler) quota
devices               Map devices
:ref:`depends_on`     Express dependency between services                      |checkmark|
:ref:`dev_over`       Service level directives that apply only in development
dns                   Custom DNS servers
dns_search            Custom DNS search
domainname            Set the FQDN
enable_ipv6           Enable IPv6 networking
entrypoint            Override the default entrypoint                          |checkmark|
env_file              Add environment variables from a file
environment           Add environment variables                                |checkmark|
:ref:`expose`         Expose ports internally to other containers              |checkmark|
extends               Extend another service, in the current file or another,
                      optionally overriding configuration
external_links        Link to containers started outside this project
:ref:`extra_hosts`    Add hostname mappings                                    |checkmark|
from                  The base image to start from                             |checkmark|
hostname              Set the container hostname
ipc                   Configure IPC settings
isolation             Specify the container's isolation technology
:ref:`k8s`            k8s engine directives                                    |checkmark|
labels                Add meta data to the container                           |checkmark|
:ref:`links`          Link services                                            |checkmark|
link_local_ips        List of special, external IPs to link to
logging               Logging configuration
log_driver            Specify a log driver (V1 only)
log_opt               Specify logging options as key:value pairs (V1 only)
mac_address           Set the mac address
mem_limit             Memory limit
memswap_limit         Total memory limit (memory + swap)
net                   Network mode (V1 only)
network_mode          Network mode
networks              Networks to join
:ref:`openshift`      openshift engine directives                              |checkmark|
pid                   Sets the PID mode to the host PID mode, enabling between
                      container and host OS
:ref:`ports`          Expose ports externally to the host                      |checkmark|
privileged            Run in privileged mode                                   |checkmark|
read_only             Mount the container's file system as read only           |checkmark|
restart               Restart policy to apply when a container exits           |checkmark|
security_opt          Override default labeling scheme
shm_size              Size of /dev/shm
stdin_open            Keep stdin open                                          |checkmark|
tty                   Allocate a psuedo-tty
stop_signal           Sets an alternative signal to stop the container
tmpfs                 Mount a temporary volume to the container                |checkmark|
ulimits               Override the default ulimit
user                  Username or UID used to execute internal container       |checkmark|
                      processes
:ref:`volumes`        Mounts paths or named volumes                            |checkmark|
volume_driver         Specify a volume driver
:ref:`volumes_from`   Mount one or more volumes from one container into        |checkmark|
                      another
working_dir           Path to set as the working directory                     |checkmark|
===================== ======================================================== ============

Implementation
--------------

The following provides details about how specific directives are implemented.

.. _depends_on:

depends_on
``````````
Express a dependency between services, causing services to be started in order. Supported by ``build`` and ``run`` commands,
but will be ignored by ``deploy``.

.. _dev_over:

dev_overrides
`````````````
Use for directives that should only be applied during the execution of the ``run`` command, or development mode. For example,
consider the following ``container.yml`` file:

.. code-block:: yaml

    version: '2'
    services:
      web:
        from: centos:7
        command: [nginx]
        entrypoint: [/usr/bin/entrypoint.sh]
        ports:
          - 8000:8000
        dev_overrides:
          ports:
            - 8888:8000
          volumes:
            - ${PWD}:/var/lib/static


In this example, when ``ansible-container run`` is executed, the options found in *dev_overrides* will
take effect, and the running container will have its port 8000 mapped to the host's port 8888, and the host's working
directory will be mounted to '/var/lib/static' in the container.

The ``build`` and ``deploy`` commands ignore *dev_overrides*. When ``build`` executs, the running container
does not have the host's working directory mounted, and the container port 8000 is mapped to the host's port 8000. And
likewise, the ``deploy`` command will create a service using port 8000, and will not create any volumes for the container.

.. _expose:

expose
``````

For the ``build`` and ``run`` commands, this exposes ports internally, allowing the container to accept requests from other
containers.

In the cloud, an exposed port translates to a service, and ``deploy`` will create a service for each exposed port. The cloud
service will have the same name as the `container.yml` service, will listen on the specified port, and forward requests
to the same port on the pod.

.. _extra_hosts:

extra_hosts
```````````
For ``build`` and ``run``, adds a hosts entry to the container.

In the cloud, ``deploy`` will create an External IP service. See `Kubernetes external IPs <http://kubernetes.io/docs/user-guide/services/#external-ips for details>`_
for details.

.. _links:

links
`````

Links allow containers to communicate directly without having to define a network, and this is supported by the ``build``
and ``run`` commands.

In the cloud, *links* are not supported, and so they will be ignored by ``deploy``. However, containers can communicate
using services, so to enable communication between two containers, add the *expose* directive. See *expose* above.

.. _k8s:

k8s
```

Specify directives specific to the ``k8s`` engine. View :ref:`k8s_openshift_options` for a reference of available directives.


.. _openshift:

openshift
`````````

Specify directives specific to the ``openshift`` engine. View :ref:`k8s_openshift_options` for a reference of available directives.

.. _ports:

ports
`````
Connects ports from the host to the container, allowing the container to receive external requests. This is supported by
the ``build`` and ``run`` commands.

The ``deploy`` command supports it as well by mapping the same functionality to the cloud. In the case of Kubernetes it creates
a load balanced service that accepts external requests on the host port and relays them to the pod, which contains the
container, on the container port. In the case of OpenShift it creates a route and service, where the route accepts external
requests on the host port, and relays them to a service listening on the container port, which relays them to a pod also on
the container port.

.. _registries:

registries
``````````
Define registries that can be referenced by the ``push`` and ``deploy`` commands. For each registiry provide a *url* and
and optional namespace. If no namespace is provided, the username found in your .docker/config.json or specified on the
command line will be used.

The following is an example taken from a ``container.yml`` file:

.. code-block:: yaml

    registries:
      google:
        url: https://gcr.io
        namespace: my-project
      openshift
        url: https://192.168.30.14.xip.io

Use the following command to push images to the *google* registry:

.. code-block:: bash

     # Push images
     $ ansible-container push --push-to google

.. _volumes:

volumes
```````

Supported by ``build``, ``run`` and ``deploy`` commands. The volumes directive mounts host paths or named volumes to the container.
In version 2 of compose a named volume must be defined in the top-level volumes directive. In version 1, if a named volume does
not exist, it is automatically created.

In the cloud, host paths result in the creation of an `emptyDir <http://kubernetes.io/docs/user-guide/volumes/#emptydir>`_,
and a named volume will result in the creation of a persistent volume claim (PVC). The resulting emptyDir or PVC will then
be mounted to the container using the specified path.

Ansible Container follows the `Portable Configuration pattern <http://kubernetes.io/docs/user-guide/persistent-volumes/#writing-portable-configuration>`_,
which means:

- It does not create persistent volumes
- It does create persistent volume claims.

.. _volumes_from:

volumes_from
````````````

Mount all the volumes from another service or container. Supported by ``build`` and ``run`` commands, but not supported
in the cloud, and thus ignored by ``deploy``.


.. _k8s_openshift_options:

k8s and openshift options
-------------------------

When using the ``k8s`` and ``openshift`` engines, the following commands are available for managing cluster objects:

 - deploy
 - restart
 - run
 - stop
 - destroy

To impact how objects are created, a ``k8s`` or ``openshift`` section can be added to a specific service, and to a named volume within the top-level volumes directive. The following presents an``openshift`` example:


.. code-block:: yaml

    version: '2'
    services:
      web:
        from: centos:7
        command: [nginx]
        entrypoint: [/usr/bin/entrypoint.sh]
        ports:
          - 8000:8000
        volumes:
          volumes:
            - static-content:/var/www/static
        dev_overrides:
          ports:
            - 8888:8000
          volumes:
            - $PWD:/var/www/static
            - /home/myuser/directory-on-the-host:/var/www/static2
        openshift:
          state: present
          service:
            force: false
          deployment:
            force: false
            replicas: 2
            security_context:
              run_as_user: root
            strategy:
              type: Rolling
              rolling_params:
                timeout_seconds: 120
                max_surge: "20%"
                max_unavailable: "10%"
                pre: {}
                post: {}
          routes:
          - port: 8443
            tls:
            termination: passthrough
            force: false

     volumes:
       static-content:
         openshift:
            state: present
            force: false
            access_modes:
            - ReadWriteOnce
            requested_storage: 5Gi


Service level directives
````````````````````````

The following directives can be added to a ``k8s`` or ``openshift`` section within a service:

======================== ======================================================================================================
Directive                Definition
======================== ======================================================================================================
state                    Set to *present*, if the service should be deployed to the cluster, or *absent*, if it should not.
                         Defaults to *present*.
:ref:`service_sub`       Adds a mapping of Service object attributes.
:ref:`deployment_sub`    Adds a mapping of Deployment (or DeploymentConfig for OpenShift) object attributes.
:ref:`route_sub`         Adds a mapping of OpenShift Route object attributes.
======================== ======================================================================================================

.. _service_sub:

service
.......

Service objects expose container ports based on the ``expose`` and ``ports`` directives defined on the service. The ``expose`` directive will result in a Service exposing ports internally, enabling containers to communicate with one another, and ``ports`` will result in a service exposing ports externally, enabling access from outside of the cluster.

Any valid attributes of a Service object can be added to the ``service`` subsection, where they'll be passed through to the resulting Service definition. The only requirement is that attributes be added in snake_case, rather than camelCase. The following demonstrates setting *cluster_ip*, *load_balancer_ip*, *type*, and *annotations*:

.. code-block:: yaml

    openshift:
      service:
        force: false
        cluster_ip: 10.0.171.239
        load_balancer_ip: 78.11.24.19
        type: LoadBalancer
        metadata:
          annotations: service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012

By default, existing objects are patched when attributes differ from those specified in ``container.yml``. The patch process is additive, meaning that array and dictionary type values are added to rather than replaced. To override this behavior, and force an update of the object, set the ``force`` option to *true*.

.. _deployment_sub:

deployment
..........

Container objects are created by way of Deployments (or Deployment Configs on OpenShift), and each service will be translated into a Deployment that creates and manages the container.

Any valid attributes of a Deployment object can be added to the ``deployment`` subsection, where they'll be passed through to the resulting Deployment definition. The only requirement is that attributes be added in snake_case, rather than camelCase.

For example, the following shows setting *replicas*, *security_context*, *strategy*, and *triggers*:

.. code-block:: yaml

    openshift:
      deployment:
        force: false
        replicas: 2
        security_context:
          run_as_user: root
        strategy:
          type: Rolling
          rolling_params:
            timeout_seconds: 120
            max_surge: "20%"
            max_unavailable: "10%"
            pre: {}
            post: {}
        triggers:
        - type: "ImageChange"
          image_change_params:
            automatic: true
            from:
              kind: "ImageStreamTag"
              name: "test-mkii-web:latest"
            container_names:
              - "web"

By default, existing objects are patched when attributes differ from those specified in ``container.yml``. The patch process is additive, meaning that array and dictionary type values are added to rather than replaced. To override this behavior, and force an update of the object, set the ``force`` option to *true*.

.. _route_sub:

routes
......

Route objects are used by OpenShift to expose services externally, and Ansible Container generates routes based on the ``ports`` directive of a service.

Consider the following service defined in ``container.yml``:

.. code-block:: yaml

    services:
      web:
        from: centos:7
        entrypoint: ['/usr/bin/entrypoint.sh']
        working_dir: /
        user: apache
        command: [/usr/bin/dumb-init, httpd, -DFOREGROUND]
        ports:
        - 8000:8080
        - 4443:8443

For each port in the set of defined ``ports``, a Route object is generated, and the above will generate the following routes:

.. code-block:: yaml

    apiVersion: v1
    kind: Route
    metadata:
      name: web-8000
      namespace: test-mkii
      labels:
        app: test-mkii
        service: web
      spec:
        to:
          kind: Service
          name: web
        port:
          targetPort: port-8000-tcp

.. code-block:: yaml

    apiVersoin: v1
    kind: Route
    metadata:
      name: web-4443
      namespace: test-mkii
      labels:
        app: test-mkii
        service: web
      spec:
        to:
          kind: Service
          name: web
        port: 4443

To add additional options, such as configuring TLS, add the options to the service level `k8s` or `openshift`, as in the following example:

.. code-block:: yaml

    services:
      web:
        from: centos:7
        entrypoint: ['/usr/bin/entrypoint.sh']
        working_dir: /
        user: apache
        command: [/usr/bin/dumb-init, httpd, -DFOREGROUND]
        ports:
        - 8000:8080
        - 4443:8443
        openshift:
          routes:
          - port: 4443
            tls:
              termination: edge
              key: |-
                -----BEGIN PRIVATE KEY-----
                [...]
                -----END PRIVATE KEY-----
              certificate: |-
                -----BEGIN CERTIFICATE-----
                [...]
                -----END CERTIFICATE-----
              caCertificate: |-
                -----BEGIN CERTIFICATE-----
                [...]
                -----END CERTIFICATE-----
            force: false

Notice that ``routes`` is a list. To set the route attributes for a specific port, add a new entry to the list, and set the ``port`` to the host or external port value.

The host port value comes from the ``ports`` directive set at the service level, where a port is in the Docker format of ``host_port:container_port``. Looking back at the first example, the ``web`` service publishes container port 8443 to host port 4443, and thus the route port will be 4443.

With the new options, the route for port 4443 will be updated with the following:

.. code-block:: yaml

    apiVersoin: v1
    kind: Route
    metadata:
      name: web-4443
      namespace: test-mkii
      labels:
        app: test-mkii
        service: web
    spec:
      to:
        kind: Service
        name: web
      port: 4443
      tls:
        termination: edge
        key: |-
          -----BEGIN PRIVATE KEY-----
          [...]
          -----END PRIVATE KEY-----
        certificate: |-
          -----BEGIN CERTIFICATE-----
          [...]
          -----END CERTIFICATE-----
        caCertificate: |-
          -----BEGIN CERTIFICATE-----
          [...]
          -----END CERTIFICATE-----


.. volumes:

Volumes
```````

For Docker, the service level ``volumes`` directive works as expected. The top-level ``volumes`` directive, however, has been modified slightly. The following example ``container.yml`` shows the three forms of the service level ``volumes`` directive, and the new top-level ``volumes`` format:

.. code-block:: yaml

    version: '2'
    services:
      web:
        from: centos:7
        entrypoint: [/usr/bin/entrypoint.sh]
        working_dir: /
        user: apache
        command: [/usr/bin/dumb-init, httpd, -DFOREGROUND]
        ports:
        - 8000:8080
        - 4443:8443
        roles:
        - apache-container
        volumes:
          - /Users/chouseknecht/projects/test-mkii/static:/var/www/static
          - static-content:/var/www/static2
          - /var/www/static3

    volumes:
      static-content:
        docker: {}
        k8s:
          force: false
          state: present
          access_modes:
          - ReadWriteOnce
          requested_storage: 1Gi
          metadata:
            annotations: 'volume.beta.kubernetes.io/mount-options: "discard"'

The top-level directive is organized by volume name. In this case, a volume named ``static-content`` is mounted to the container as ``/var/www/static2``. The definition of the named volume is found in the top-level ``volumes`` directive under the name, where specific options are organized by container engine. In this case there are no options for ``docker``, and several options for ``openshift``.

Under ``docker``, add valid volume attributes including: driver, driver_opts and external. For additional information about Docker volumes see Docker's `volume configuration reference <https://docs.docker.com/compose/compose-file/#volume-configuration-reference>`_.


For ``openshift`` and ``k8s``, the following options are available:

======================== =============================================================================================================
Directive                Definition
======================== =============================================================================================================
metadata                 Provide a metadata mapping, as depicted above. In general, the only mapping value provided here would be
                         ``annotations``.
access_modes             A list of valid `access modes <http://kubernetes.io/docs/user-guide/persistent-volumes/#access-modes>`_.
match_labels             A mapping of key:value pairs used to filter matching volumes.
match_expressions        A list of expressions used to filter matching volumes.
                         See `Persistent Volume Claims <https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims>`_ for additional details.
requested_storage        The amount of storage being requested. Defaults to 1Gi.
                         See `compute resources <http://kubernetes.io/docs/user-guide/compute-resources/>`_ for abbreviations.
======================== =============================================================================================================


