..
    This work is licensed under a Creative Commons Attribution 3.0 Unported
    License.

    http://creativecommons.org/licenses/by/3.0/legalcode

    Convention for heading levels in Neutron devref:
    =======  Heading 0 (reserved for the title in a document)
    -------  Heading 1
    ~~~~~~~  Heading 2
    +++++++  Heading 3
    '''''''  Heading 4
    (Avoid deeper levels because they do not render well.)

=========================================================
Kuryr Kubernetes ocp-route And Ingress Integration Design
=========================================================

Purpose
-------
The purpose of this document is to present how openshift route and Kubernetes ingress are supported
by the kuryr integration.

Overview
----------
An OpenShift route and Kubernetes ingress are used to give services externally-reachable URLs,
load balance traffic, terminate SSL, offer name based virtual hosting, and more.
Each route/ingress consists of a name, service identifier, and (optionally) security configuration.
A defined route/ingress and the endpoints identified by its service are consumed by a L7-router 
to provide named connectivity that allows external clients to reach your applications.

Proposed Solution
-----------------
An OpenShift/Kubernetes administrator can deploy L7 router in an OpenShift/Kubernetes cluster,
which enable ingress/ocp-routes created by developers to be used by external clients.
The Router should perfrom L7 routing layer based on L7 rules database, where the Ingress/OCP-Route
controllers are responsible for updating the L7 rules database.
The layer 7 loadbalacing capability will be composed from :
1. L7 Router
2. Ingress/OCP-Route controllers.

The L7 Router
~~~~~~~~~~~~~
The L7 Router is based on neutron LbaaS L7 policy capability,
L7 router is an extranlly reachable loadbalancer, for achieving external conenctivity
a floating IP (allocated from 'external_svc_subnet') is bounded to the Router loadbalancer.
The following parameters should be configured in kuryr.conf file to enable L7 Router::

         [neutron_defaults]
         external_svc_subnet=  external_subnet_id
         [kubernetes]
         l7_router_driver= neutron_l7_policy 
         
After the L7 router was created, we should retrieve the Router's FIP, 
and point (at DNS) external traffic to L7 Router(FIP).
The Router's FIP could be retrieved from node annotation's as appears below.
.. code-block:: yaml

    apiVersion: v1
    kind: Node
    metadata:
      annotations:
        openstack.org/kuryr-l7-router-state: '{"versioned_object.data": {"fip": "172.24.4.14",
          "router_lb": {"versioned_object.data": {"id": "90732f0a-651a-4b17-a14e-9b0e01fbe774",
          "ip": "10.0.0.154", "name": "kuryr-l7-router", "port_id": "5c71a29a-0dc1-461e-81ee-2258a7e3842d",
          "project_id": "868307936d384c21824e5eb0425a3f42", "subnet_id": "9f6d8c9f-d22d-480e-80f5-867daa050ff8"},
          "versioned_object.name": "LBaaSLoadBalancer", "versioned_object.namespace":
          "kuryr_kubernetes", "versioned_object.version": "1.0"}}, "versioned_object.name":
          "L7RouterState", "versioned_object.namespace": "kuryr_kubernetes", "versioned_object.version":
          "1.0"}'
        volumes.kubernetes.io/controller-managed-attach-detach: "true"
      creationTimestamp: 2017-11-17T19:52:54Z

The next diagram illustrates data flow from external user to L7 loadbalancer:
.. image:: ../../images/vif_handler_drivers_design.png
    :alt: vif handler drivers design
    :align: center
    :width: 100%

    
apiVersion: v1
kind: Node
metadata:
  annotations:
    openstack.org/kuryr-l7-router-state: '{"versioned_object.data": {"fip": "172.24.4.14",
      "router_lb": {"versioned_object.data": {"id": "90732f0a-651a-4b17-a14e-9b0e01fbe774",
      "ip": "10.0.0.154", "name": "kuryr-l7-router", "port_id": "5c71a29a-0dc1-461e-81ee-2258a7e3842d",
      "project_id": "868307936d384c21824e5eb0425a3f42", "subnet_id": "9f6d8c9f-d22d-480e-80f5-867daa050ff8"},
      "versioned_object.name": "LBaaSLoadBalancer", "versioned_object.namespace":
      "kuryr_kubernetes", "versioned_object.version": "1.0"}}, "versioned_object.name":
      "L7RouterState", "versioned_object.namespace": "kuryr_kubernetes", "versioned_object.version":
      "1.0"}'
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: 2017-11-17T19:52:54Z


         
       l7_router_driver = neutron_l7_policy
which means that the L7 router is actually a loadbalancer
Since Router loadbalancer should be accessible from external network, a floating IP will 
be bounded to 
An OpenShift/Kubernetes administrator can deploy router in an OpenShift/Kubernetes cluster,
which enable ingress/ocp-routes created by developers to be used by external clients.
The Router should perfrom L7 routing layer based on L7 rules database, where the Ingress/OCP-Route
controllers are responsible for updating the L7 rules database.

    The L7 Router is actually a loadbalancer with URL mapping capabilities.
    A floating IP should be assigned to the L7 router.


The Router driver will be based on neutron Lbaas  L7  capability.

A single load balancer (i.e : ‘Router LB’) should be created,a specific IP address from service subnet should be reserved for the ‘Router LB’’s VIP.

Since ‘Router LB’ should be accessible from external network, a configurable floating IP should be defined for this purpose.
The floating IP should be bounded to the Router’s LB VIP as appears in below diagram:

Ingress/OCP-Route controllers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


    L7 Router
    =========
    The L7 Router is actually a loadbalancer with URL mapping capabilities.
    A floating IP should be assigned to the L7 router.

    Ingress/OCP-Route controllers
    =============================
    The ingress/ocp-route controller watches the apiserver's for updates to
    the Ingress/ocp-route resource. Its job is to satisfy requests
    for Ingresses/ocp-route.
    Each Ingress/ocp-route being translated to a new L7 policy in
    L7 router, and the rules on the Ingress/ocp-route become L7 (URL)
    mapping rules in that L7 policy.

    At first phase, only the ocp-route controller is implemented.

    Data Flow :
    ===========
    To achieve L7 loadbalancing through kuryr, we should first
    point (at DNS) the external traffic to the L7 Router(FIP),
    the L7 router parse the URL to see which  backend service
    should handle the request.
    """
    The L7 loadbalancing capability in kuryr is composed from :
    1. L7 router
    2. Ingress/OCP-Route controllers.

Kubernetes service in its essence is a Load Balancer across Pods that fit the
service selection. Kuryr's choice is to support Kubernetes services by using
Neutron LBaaS service. The initial implementation is based on the OpenStack
LBaaSv2 API, so compatible with any LBaaSv2 API provider.
In order to be compatible with Kubernetes networking, Kuryr-Kubernetes
makes sure that services Load Balancers have access to Pods Neutron ports.
This may be affected once Kubernetes Network Policies will be supported.
Oslo versioned objects are used to keep translation details in Kubernetes entities
annotation. This will allow future changes to be backward compatible.


defined a way to expose a service by giving it an externally-reachable hostname like www.example.com.

A defined route and the endpoints identified by its service can be consumed by a router to provide named connectivity 
that allows external clients to reach your applications.
Each route consists of a route name, service selector, and (optionally) security configuration.

Routers
An OpenShift administrator can deploy routers in an OpenShift cluster,
which enable routes created by developers to be used by external clients.
The routing layer in OpenShift is pluggable, and two available router plug-ins are provided and supported by default.

OpenShift routers provide external host name mapping and load balancing to services over protocols
that pass distinguishing information directly to the router; the host name must be present in the protocol
in order for the router to determine where to send it.

It can be configured to give services externally-reachable URLs, load balance traffic, terminate SSL,
offer name based virtual hosting, and more.
Users request ingress by POSTing the Ingress resource to the API server.
An Ingress controller is responsible for fulfilling the Ingress, usually with a loadbalancer,
though it may also configure your edge router or additional frontends to help handle the traffic in an HA manner.

VIF-Handler
-----------
VIF-handler is intended to handle VIFs. The main aim of VIF-handler is to get
the pod object, send it to Multi-vif driver and get vif objects from it. After
that VIF-handler is able to activate, release or update vifs. Also VIF-Handler
is always authorized to get main vif for pod from generic driver.
VIF-handler should stay clean whereas parsing of specific pod information
should be moved to Multi-vif driver.

Multi-vif driver
~~~~~~~~~~~~~~~~~
The main driver that is authorized to call other drivers. The main aim of
this driver is to get list of enabled drivers, parse pod annotations, pass
pod object to enabled drivers and get vif objects from them to pass these
objects to VIF-handler finally. The list of parsed annotations by Multi-vif
driver includes sriov requests, additional subnets requests and specific ports.
If the pod object doesn't have annotation which is required by some of the
drivers then this driver is not called or driver can just return.
Diagram describing VifHandler - Drivers flow is giver below:

.. image:: ../../images/vif_handler_drivers_design.png
    :alt: vif handler drivers design
    :align: center
    :width: 100%

Config Options
~~~~~~~~~~~~~~
Add new config option "enabled_vif_drivers" (list) to config file that shows
what drivers should be used in Multi-vif driver to collect vif objects. This
means that Multi-vif driver will pass pod object only to specified drivers
(generic driver is always used by default and it's not necessary to specify
it) and get vifs from them.
Option in config file might look like this:

.. code-block:: ini

    [kubernetes]

    enabled_vif_drivers =  sriov, additional_subnets


Additional Subnets Driver
~~~~~~~~~~~~~~~~~~~~~~~~~
Since it is possible to request additional subnets for the pod through the pod
annotations it is necessary to have new driver. According to parsed information
(requested subnets) by Multi-vif driver it has to return dictionary containing
the mapping 'subnet_id' -> 'network' for all requested subnets in unified format
specified in PodSubnetsDriver class.
Here's how a Pod Spec with additional subnets requests might look like:

.. code-block:: yaml

    spec:
      replicas: 1
      template:
        metadata:
          name: some-name
          labels:
            app: some-name
          annotations:
            openstack.org/kuryr-additional-subnets: '[
                "id_of_neutron_subnet_created_previously"
            ]'


SRIOV Driver
~~~~~~~~~~~~
SRIOV driver gets pod object from Multi-vif driver, according to parsed
information (sriov requests) by Multi-vif driver. It should return a list of
created vif objects. Method request_vif() has unified interface with
PodVIFDriver as a base class.
Here's how a Pod Spec with sriov requests might look like:

.. code-block:: yaml

    spec:
      containers:
      - name: vf-container
        image: vf-image
        resources:
          requests:
            pod.alpha.kubernetes.io/opaque-int-resource-sriov-vf-physnet2: 1


Specific ports support
----------------------
Specific ports support is enabled by default and will be a part of the drivers
to implement it. It is possile to have manually precreated specific ports in
neutron and specify them in pod annotations as preferably used. This means that
drivers will use specific ports if it is specified in pod annotations, otherwise
it will create new ports by default. It is important that specific ports can have
vnic_type both direct and normal, so it is necessary to provide processing
support for specific ports in both SRIOV and generic driver.
Pod annotation with requested specific ports might look like this:

.. code-block:: yaml

    spec:
      replicas: 1
      template:
        metadata:
          name: some-name
          labels:
            app: some-name
          annotations:
            spec-ports: '[
                "id_of_direct_precreated_port".
                "id_of_normal_precreated_port"
            ]'

Pod spec above should be interpreted the following way:
Multi-vif driver parses pod annotations and gets ids of specific ports.
If vnic_type is "normal" and such ports exist, it calls generic driver to create vif
objects for these ports. Else if vnic_type is "direct" and such ports exist, it calls
sriov driver to create vif objects for these ports. If certain ports are not
requested in annotations then driver doesn't return additional vifs to Multi-vif
driver.
