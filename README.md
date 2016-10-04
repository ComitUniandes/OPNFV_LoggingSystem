# Towards an Analyzing Tool for Audit Logs on OPNFV

## The Project
The following instructions are just the first step of a larger project that aims to create a tool capable of creating 
and managing security logs. 

Our goal is to develop a program that collects and analyzes the logs created by OPNFV in order to detect security 
issues and offer an easy-to-understand dashboard for network administrators.

This project is part of the Comit research group at Universidad de los Andes, Colombia.

## Log auditing

This step-by-step tutorial will explain how to enable logging in different OpenStack modules on OPNFV using CADF standard. We use OPNFV 
as it is the open-source implementation to NFV and it allows a virtual deployment for environment that do not fulfill the 
bare-metal requirements. 

OPNFV uses OpenStack, a cloud management software, as the main controller for the virtual and physical resources. OpenStack 
has a modular architecture, which brings the opportunity to design and deploy and NFV platform with the necessary services.
On the following sections we will explain how to install OpenStack and how to enable log-auditing on the modules.

### Environment setup
As we said earlier, it was necessary to do a virtual deployment for OpenStack. To do so, the Apex installer was used to install
the OpenStack's implementation of the Triple-O (OpenStack On OpenStack) project. This project uses a virtual machine with an
all-in-one deployment of Opentack called *undercloud* to deploy the OpenStack on the cloud environment, or *overcloud*.

Using the virtual environment installation, with no high availability capabilities, the installer will create three virtual machines
on a Jumphost or physical machine. In order to install the Jumphost, we followed the instructions provided by OPNFV that can be found
on the next link:

http://artifacts.opnfv.org/apex/brahmaputra/docs/installation-instructions/installation-instructions.pdf

Once the Jumphost installation is complete, run the following command to create the three virtual machines: the *instack* (or undercloud),
the controller and the compute node. 

```
opnfv-deploy --virtual --no-ha -n network_settings.yaml -d deploy_settings.yaml 
```

The parameters for this command are:

``--version``: Indicates it is a virtual deployment

``--no-ha``: Indicates to deploy a no-high availability environment, with this option, the script would deploy three virtual machines instead of five

``-n network_settings.yaml``: The ``-n`` option indicates the path of the ``network_settings.yaml``, which is normally located on the directory ``/etc/opnfv-apex``.
			The *network_settings.yaml* file stores the different types of network configuration (administrative, private, public, storage),
			and its characteristics such as usable ip range, dhcp range and gateway. Once you installed the Jumphost, you will find an example for the file, 
			edit that file with the needed configuration.
			
``-d deploy_settings.yaml``: The ``-d`` option indicates the path of the ``deploy_settings.yaml`` file, which defines the SDN controller to be used, or if the 
			admninistrator is going to use aditional services such as congress or tacker.
			
The ``opnfv-deploy`` script will take some time to complete the deployment, but once it has finished, it should have created the three virtual machines. To 
verify the creation of those machines, use the KVM client to list the virtual machines by running the following command:

```
virsh kist --all
```

It should list 6 virtual machines, three of them will hace state *shut off*, the other three will have a *running* state. Try to access to the instack machine
through ssh with the IP address 192.0.2.1, and then change to the *stack* user. Once inside the instack machine, use the *stackrc* credentials to use the 
undercloud machines and list the available servers with their IP address. Try to access one of the nodes via *ssh* to check if it is correctly deployed.

```
ssh root@192.0.2.1
source stackrc
openstack server list
```

Finally, validate that OpenStack is functioning correctly by interacting with its functions throught the dashboard. To do so, on the Jumphost machine
(the physical machine, not the virtual one), open an internet navigator and type the address: http://192.18.37.10/dashboard . The credentials to log in
can be found on the ``tripleo-overcloud-passwords`` file on the instack machine's (192.0.2.1) */home/stack* directory and follow the instructions described
in the section OPENSTACK VERIFICATION in the document:

http://artifacts.opnfv.org/apex/brahmaputra/docs/installation-instructions/installation-instructions.pdf

**Note:** Tou may have trouble downloading the *cirros* image from the dashboard: however, you can download the file directly from the Jumphost machine
and create the image with the *local file* option. To download the cirros image, you can use the following command:

```
wget -P /tmp/images http://download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img
```

###Enabling CADF Logging
The Cloud Audit Data Federation (CADF) creates a specification to define a normative data model with a compatible set of interfaces for federating events,
logs and reports. OpenStack offers two ways to generate logs, the first one is known as *Basic Notification Format* and the second one as *CADF Format*. They
differ from each other in the format and in the information shown in the logs.
Basic Notification Format:
``` 
{
    "event_type": "identity.<resource_type>.<operation>",
    "message_id": "<message_id>",
    "payload": {},
    "priority": "INFO",
    "publisher_id": "identity.<hostname>",
    "timestamp": "<timestamp>"
}
```
CADF Format:
```
{
    "typeURI": "http://schemas.dmtf.org/cloud/audit/1.0/event",
    "initiator": {
        "typeURI": "service/security/account/user",
        "host": {
            "agent": "curl/7.22.0(x86_64-pc-linux-gnu)",
            "address": "127.0.0.1"
        },
        "id": "<initiator_id>"
    },
    "target": {
        "typeURI": "<target_uri>",
        "id": "openstack:1c2fc591-facb-4479-a327-520dade1ea15"
    },
    "observer": {
        "typeURI": "service/security",
        "id": "openstack:3d4a50a9-2b59-438b-bf19-c231f9c7625a"
    },
    "eventType": "activity",
    "eventTime": "2014-02-14T01:20:47.932842+00:00",
    "action": "<action>",
    "outcome": "success",
    "id": "openstack:f5352d7b-bee6-4c22-8213-450e7b646e9f",
}
```

To enable CADF logging on OpenStack, you will have to edit the configuration file and the ``api-dist-paste`` file for each module on OpenStack. Remember that 
for this tutorial, OpenStack was deployed using the Apex installer, this means that the operating system used is CentOS, so some commands may vary on your
installation.

####Keystone:
Keystone provides the authentication service for OpenStack. It is important to enable CADF format notification in this module, since it is going to be used
by the other modules to generate the logs. The steps to enable logging are:

1. Edit the ``keystone.conf`` file that can be found on the directory ``/etc/keystone`` and write the following lines:
	* Under the option "*From keystone.notifications*" enable the notification format to CADF:
		```notification_format = cadf```
	* Under the "*From oslo.messaging*", change the notification driver to *log*. This makes the system to write all the CADF events into a log file. Some
	tutorials change the notification driver to *messagingv2*; however, if RabbitMQ, the messaging queue, is not working properly, then the log generation
	will fail.
		```notification_driver = log ```
		
2. After editing the files, restart the keystone service.
	```sudo systemctl restart openstack-keystone.service```
	
3. To check if the logging service is enabled correctly, open the web dashboard and try to create a new user. Then, check the log to see if keystone is 
generating the CADF events. The log can be found on the directory ``/var/log/keystone/keystone.log``. Make sure that the log *payload* contains the CADF message.

####Glance:
Glance's main function is to manage the images. This module offers two main services; the first one is for the image service registry, which stores the 
images' metadat, the second service managed the API calls; therefore, there are two configuration and dist files: ``glance-api.conf``, and ``glance-api-dist-paste.ini``
for the API service, and ``glance-registry.conf`` and ``glance-registry-dist-paste.ini`` for the registry service. It is necessary to modify the files
associate with the API service because this is the service in charge of image creation and deletion tasks.

The steps to enable CADF logging in Glance are:

1. Edit the ``glance-api-idst-paste.ini file``, which is located on the directory ``/usr/share/glance/`` and:

	* Create a new filter called audit:
  
		```
		[filter:audit]
		paste.filter_factory = keystonemiddleware.audit:filter_factory
		audit_map_file = /etc/pycadf/glance_api_audit_map.conf
		```
	**Note**: Some tutorials define the filter_factory as ``pyCADF.middleware.audit``; however, the pycadf middleware is going to be 
	deprecated soon to focus on the keystone middleware.
	
	* In order to use the audit filter and write the necessary logs, modify the pipelines and add the audit filter. For example:
  
		```
		[pipeline:glance-api-keystone]
		pipeline = healthcheck versionnegotiation osprofiler authtoken context audit rootapp
		```
    
	**Note**: It is important to add the audit filter after the authtoken and context filters to grab the keystone context.
	
2. It is also necessary to edit Glance’s configuration file to enable the notification drive to logging. To do so, edit the ``glance-api.conf`` on the directory 
``/etc/glance``, and modify the line notification_driver:

  ```
  notification_drive = log
  ```

3. Restart the service:

  ```
  sudo systemctl restart openstack-glance-api.service
  ```

4. Now test the implementation by opening the dashboard and creating a new image. After doing so, check the log if any activity was registered.
The log directory is ``/var/log/glance/api.log``.

####Neutron:
Neutron module manages the network configuration for OpenStak. Before editing the configuration and initialization files, it is necessary to re-write the 
audit map file for Neutron (located on the directory: ``/etc/pycadf/neutron_api_audit_map.conf``). This is because, the first time the changes were made 
and the audit logging was tested, a syntax exception was raised. After studying the file, there was not any mistake, so it was necessary to re-write the 
file with the one found on the following link: https://github.com/openstack/pycadf/blob/master/etc/pycadf/neutron_api_audit_map.conf.

After re-writing the audit map file, modify the following files:

1. The initialization file, which is located on the directory ``/usr/share/neutron/api-paste.ini`` and add the filter just like in Glance:

  ```
  [filter:audit]
  paste.filter_factory = keystonemiddleware.audit:filter_factory
  audit_map_file = /etc/pycadf/neutron_api_audit_map.conf
  ```

  Also, modify the only composite section to add the filter. As in Glance, it is important to add the filter after the authotoken and keystonecontext filters. 

  ```
  [composite:neutronapi_v2_0]
  use = call:neutron.auth:pipeline_factory
  noauth = request_id catch_errors extensions neutronapiapp_v2_0
  keystone = request_id catch_errors authtoken keystonecontext audit extensions neutronapiapp_v2_0
  ```

2. Modify the configuration file: ``/etc/neutron/neutron/neutron.conf`` and add or modify the option *notification_driver* to:

  ```
  notification_driver = log
  ```

3. Restart the neutron service with the folloowing command:

  ```
  sudo systemctl restart neutron-server.service
  ```

4. Open the web dashboard and create, delete or modify a network. Then, check for activity registration on the log located on the directory: 
``/var/log/server.log``

####Nova:
Nova manages the instances in OpenStack. Following the steps for Glance and Neutron, it is necessary to modify the initialization file and the 
configuration file. Like Glance, Nova offers several services; however, for this tutorial, only the API service is necessary, because through this 
service virtual machines can be created, modified or deleted. 

Ahead, there are the necessary files to modify.

1. Edit the ``/etc/nova/api-paste.ini`` file. Take into account that in the case of nova, the initialization file is on the ``/etc`` directory rather 
than the ``/usr/share directory``. As with the other projects, it is necessary to create an audit filter and add the filter to the pipelines and composites.

  ```
  [filter:audit]
  paste.filter_factory = keystonemiddleware.audit:filter_factory
  audit_map_file = /etc/pycadf/nova_api_audit_map.conf
  ```

2. Edit the configuration file to use log as the notification driver and to enable the verbose option, so events with level INFO, like the CADF events, can be logged.

  ```
  notification_driver = log
  verbose = True
  ```

3. Restart nova service:

  ```
  sudo systemctl restart openstack-nova-api.service
  ```

4. Create an instance from the web dashboard and check if the activities log are created on the directory: ``/var/log/nova/nova-api.log``.
