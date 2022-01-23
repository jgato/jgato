# Using Sushy tools to manage a BMC Redfish interface

This document is a short tutorial as introduction to use Sushy tools to manage a BMC server with a Redfish interface.

What is covered in this tutorial? 

* Understand how to install and use Sushy to create virtualized Redfish interfaces.

* Some basics about the Redfish  BMC (REST) interface and how to interact with it. 

* How to manage a BMC server using its Redfish interface.

* Boot a server with a virual media iso for easiness server's provisioning

A virtual environment will be created for this tutorial, in order to facilitate the environment availability. This means:

* A virtual machine as a server.

* A virtual Redfish nterface, created with Sushy tools, as a BMC.

This virtual environment should give us enough background to later interact with real Redfish interfaces. But, it should also help us to develop some quick tests, before going for a real physical environment.

## Creating the virtual environment

First of all we create a VM without an OS installed. This VM will act as the Server.

```bash
kcli create vm -P uefi_legacy=true -P start=false -P nets=[default] -P memory=8192 -P numcpus=2 agent1
```

Now we use Sushy tools to emulate the BMC Redfish interface. 

*This is a set of simple simulation tools aiming at supporting the
development and testing of the Redfish protocol.*

The package ships two simulators :

- Simple REST API server which responds the same things to client queries. It is effectively read-only.

- The virtual Redfish BMC resembles the real Redfish-controlled bare-metal machine to some extent. Backed by libvirt or openstack cloud.

For this tutorial we will use the Virtual Redfish BMC to manage the previously create VM. Using libvirt as the backend.

```
$> dnf -y install pkgconf-pkg-config libvirt-devel gcc python3-libvirt python3 git python3-netifaces
pip3 install sushy-tools
```

Lets create a configuration file:

```
cat << EOF > ./sushy.conf
SUSHY_EMULATOR_LISTEN_IP = u'0.0.0.0'
SUSHY_EMULATOR_LISTEN_PORT = 8000
SUSHY_EMULATOR_SSL_CERT = None
SUSHY_EMULATOR_SSL_KEY = None
SUSHY_EMULATOR_OS_CLOUD = None
SUSHY_EMULATOR_LIBVIRT_URI = u'qemu:///system'
# If authentication is desired, set this to an htpasswd file.
SUSHY_EMULATOR_AUTH_FILE = './auth.conf'
#SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = True
SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = False
SUSHY_EMULATOR_BOOT_LOADER_MAP = {
    u'UEFI': {
        u'x86_64': u'/usr/share/OVMF/OVMF_CODE.secboot.fd',
        u'aarch64': u'/usr/share/AAVMF/AAVMF_CODE.fd'
    },
    u'Legacy': {
        u'x86_64': None,
        u'aarch64': None
    }
}
EOF
```

* SUSHY_EMULATOR_LIBVIRT_URI = u'qemu:///system' : the backend for redfish is libvirt, so it will interact with the VMs in the host. This variable would point to other hosts.
* SUSHY_EMULATOR_AUTH_FILE: it is an htpasswd file with the authentication.

Create the auth file

```
$> htpasswd -cB ${PWD}/auth.conf admin
New password: 
Re-type new password: 
Adding password for user admin
```

Now you can run the service with the created configuration.

```
$> /usr/local/bin/sushy-emulator --config ${PWD}/sushy.conf
 * Serving Flask app 'sushy_tools.emulator.main' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://10.39.194.148:8000/ (Press CTRL+C to quit)
```

The sushy-emulator will create a local REST interface as a bridge with the available Redfish BMC interface. The backend is implemented by Libvirt (as it was configured in SUSHY_EMULATOR_LIBVIRT_URI). Therefore, the systems to be managed will depend on the number of VMs you have in your host. 

```bash
$> export bmc=admin:admin@localhost:8000
$> curl ${bmc}/redfish/v1/Managers
{
    "@odata.type": "#ManagerCollection.ManagerCollection",
    "Name": "Manager Collection",
    "Members@odata.count": 7,
    "Members": [

          {
              "@odata.id": "/redfish/v1/Managers/04d4b2dc-1cdc-4f6e-900a-03007e9ed5c9"
          },

          {
              "@odata.id": "/redfish/v1/Managers/0604fdde-d410-40f0-8dd0-0f49f2202a94"
          },

          {
              "@odata.id": "/redfish/v1/Managers/184ca2d3-5fec-413f-a8b4-6823773add6d"
          },

          {
              "@odata.id": "/redfish/v1/Managers/3a877855-8bb2-4907-be67-ca91ff6d6bc7"
          },

          {
              "@odata.id": "/redfish/v1/Managers/9f0b1a48-7eb9-45ed-95cf-4ac0877cab0d"
          },

          {
              "@odata.id": "/redfish/v1/Managers/c3605a63-90f2-4aad-a227-4e49cf257d8a"
          },

          {
              "@odata.id": "/redfish/v1/Managers/dd1ef1cf-7c9c-4e1a-8907-dc2ce25f4a39"
          }

    ],
    "Oem": {},
    "@odata.context": "/redfish/v1/$metadata#ManagerCollection.ManagerCollection",
    "@odata.id": "/redfish/v1/Managers",
    "@Redfish.Copyright": "Copyright 2014-2017 Distributed Management Task Force, Inc. (DMTF). For the full DMTF copyright policy, see http://www.dmtf.org/about/policies/copyright."
}
$> kcli list vms
+--------------+--------+-----+--------------------------------------------------------+-------+---------+
|     Name     | Status | Ips |                         Source                         |  Plan | Profile |
+--------------+--------+-----+--------------------------------------------------------+-------+---------+
|    agent1    |  down  |     |                                                        | kvirt |  kvirt  |
|   bifrost    |  down  |     | CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2 | kvirt | centos8 |
|   bifrost2   |  down  |     | CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2 | kvirt | centos8 |
|   minikube   |  down  |     |                                                        |       |         |
| minikube-m02 |  down  |     |                                                        |       |         |
| minikube-m03 |  down  |     |                                                        |       |         |
|   support    |  down  |     | CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2 | kvirt | centos8 |
+--------------+--------+-----+--------------------------------------------------------+-------+---------+
```

In my case, I have 7 VMs. So it returns 7 Managers (BMCs) or 7 Systems (Servers)

```bash
{
    "@odata.type": "#ComputerSystemCollection.ComputerSystemCollection",
    "Name": "Computer System Collection",
    "Members@odata.count": 7,
    "Members": [

            {
                "@odata.id": "/redfish/v1/Systems/9f0b1a48-7eb9-45ed-95cf-4ac0877cab0d"
            },

            {
                "@odata.id": "/redfish/v1/Systems/3a877855-8bb2-4907-be67-ca91ff6d6bc7"
            },

            {
                "@odata.id": "/redfish/v1/Systems/c3605a63-90f2-4aad-a227-4e49cf257d8a"
            },

            {
                "@odata.id": "/redfish/v1/Systems/0604fdde-d410-40f0-8dd0-0f49f2202a94"
            },

            {
                "@odata.id": "/redfish/v1/Systems/dd1ef1cf-7c9c-4e1a-8907-dc2ce25f4a39"
            },

            {
                "@odata.id": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d"
            },

            {
                "@odata.id": "/redfish/v1/Systems/04d4b2dc-1cdc-4f6e-900a-03007e9ed5c9"
            }

    ],
    "@odata.context": "/redfish/v1/$metadata#ComputerSystemCollection.ComputerSystemCollection",
    "@odata.id": "/redfish/v1/Systems",
    "@Redfish.Copyright": "Copyright 2014-2016 Distributed Management Task Force, Inc. (DMTF). For the full DMTF copyright policy, see http://www.dmtf.org/about/policies/copyright."
}
┌─
```

## Interacting with a Redfish interface

As we see in the outputs above, the Refish interface manages urls in the way of:

```
/redfish/v1/Systems/UUID/   # For the Server
/redfish/v1/Managers/UUID/   # For the BMC
```

In order to interact with the VM we created in the previous step, we have to find out what is its UUID.

```bash
$> sudo virsh domuuid  agent1
184ca2d3-5fec-413f-a8b4-6823773add6d
```

So, lets interact with our VM with the Redfish interface:

```bash
$> export bmc_server=184ca2d3-5fec-413f-a8b4-6823773add6d
$> curl -s ${bmc}/redfish/v1/Systems/${bmc_server} | jq
{
  "@odata.type": "#ComputerSystem.v1_1_0.ComputerSystem",
  "Id": "184ca2d3-5fec-413f-a8b4-6823773add6d",
  "Name": "agent1",
  "UUID": "184ca2d3-5fec-413f-a8b4-6823773add6d",
  "Manufacturer": "Sushy Emulator",
  "Status": {
    "State": "Enabled",
    "Health": "OK",
    "HealthRollUp": "OK"
  },
  "PowerState": "Off",
  "Boot": {
    "BootSourceOverrideEnabled": "Continuous",
    "BootSourceOverrideTarget": "Hdd",
    "BootSourceOverrideTarget@Redfish.AllowableValues": [
      "Pxe",
      "Cd",
      "Hdd"
    ],
    "BootSourceOverrideMode": "UEFI",
    "UefiTargetBootSourceOverride": "/0x31/0x33/0x01/0x01"
  },
  "ProcessorSummary": {
    "Count": 2,
    "Status": {
      "State": "Enabled",
      "Health": "OK",
      "HealthRollUp": "OK"
    }
  },
  "MemorySummary": {
    "TotalSystemMemoryGiB": 8,
    "Status": {
      "State": "Enabled",
      "Health": "OK",
      "HealthRollUp": "OK"
    }
  },
  "Bios": {
    "@odata.id": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d/BIOS"
  },
  "Processors": {
    "@odata.id": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d/Processors"
  },
  "Memory": {
    "@odata.id": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d/Memory"
  },
  "EthernetInterfaces": {
    "@odata.id": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d/EthernetInterfaces"
  },
  "SimpleStorage": {
    "@odata.id": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d/SimpleStorage"
  },
  "Storage": {
    "@odata.id": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d/Storage"
  },
  "IndicatorLED": "Lit",
  "Links": {
    "Chassis": [
      {
        "@odata.id": "/redfish/v1/Chassis/15693887-7984-9484-3272-842188918912"
      }
    ],
    "ManagedBy": [
      {
        "@odata.id": "/redfish/v1/Managers/184ca2d3-5fec-413f-a8b4-6823773add6d"
      }
    ]
  },
  "Actions": {
    "#ComputerSystem.Reset": {
      "target": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d/Actions/ComputerSystem.Reset",
      "ResetType@Redfish.AllowableValues": [
        "On",
        "ForceOff",
        "GracefulShutdown",
        "GracefulRestart",
        "ForceRestart",
        "Nmi",
        "ForceOn"
      ]
    }
  },
  "@odata.context": "/redfish/v1/$metadata#ComputerSystem.ComputerSystem",
  "@odata.id": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d",
  "@Redfish.Copyright": "Copyright 2014-2016 Distributed Management Task Force, Inc. (DMTF). For the full DMTF copyright policy, see http://www.dmtf.org/about/policies/copyright."
}
```

Interesting info:

* "Name": "agent1", which is the name we gave to the VM

* "PowerState": "off",  this is ok, the VM is off

* "ProcessorSummary.Count": 2, again, as we specified when creating VM

Some BMC info for this Server:

```bash
> curl -s ${bmc}/redfish/v1/Managers/${bmc_server} | jq
{
  "@Redfish.Copyright": "Copyright 2014-2017 Distributed Management Task Force, Inc. (DMTF). For the full DMTF copyright policy, see http://www.dmtf.org/about/policies/copyright.",
  "@odata.context": "/redfish/v1/$metadata#Manager.Manager",
  "@odata.id": "/redfish/v1/Managers/184ca2d3-5fec-413f-a8b4-6823773add6d",
  "@odata.type": "#Manager.v1_3_1.Manager",
  "DateTime": "2022-56-21T13:56:01+00:00",
  "DateTimeLocalOffset": "+00:00",
  "Description": "Contoso BMC",
  "FirmwareVersion": "1.00",
  "Id": "184ca2d3-5fec-413f-a8b4-6823773add6d",
  "Links": {
    "ManagerForChassis": [],
    "ManagerForServers": [
      {
        "@odata.id": "/redfish/v1/Systems/184ca2d3-5fec-413f-a8b4-6823773add6d"
      }
    ]
  },
  "ManagerType": "BMC",
  "Model": "Joo Janta 200",
  "Name": "agent1-Manager",
  "PowerState": "On",
  "ServiceEntryPointUUID": null,
  "Status": {
    "Health": "OK",
    "State": "Enabled"
  },
  "UUID": "184ca2d3-5fec-413f-a8b4-6823773add6d",
  "VirtualMedia": {
    "@odata.id": "/redfish/v1/Managers/184ca2d3-5fec-413f-a8b4-6823773add6d/VirtualMedia"
  }
}
```

To power on the server, look at the system "Actions.#ComputerSystem.Reset.target" to get the url and the list of actions supported: on, off, GracefulShutdown, etc

```bash
# rest the server
$> curl -s -H "Content-Type: application/json"  -X POST ${bmc}/redfish/v1/Systems/${bmc_server}/Actions/ComputerSystem.Reset -d '{"ResetType": "On"}'
# check current status
$>  curl -s ${bmc}/redfish/v1/Systems/${bmc_server} | jq .PowerState
"On"
# switch off
$> curl -H "Content-Type: application/json"  -X POST ${bmc}/redfish/v1/Systems/${bmc_server}/Actions/ComputerSystem.Reset -d '{"ResetType": "ForceOff"}'
# reboot
$> curl -H "Content-Type: application/json"  -X POST ${bmc}/redfish/v1/Systems/${bmc_server}/Actions/ComputerSystem.Reset -d '{"ResetType": "ForceRestart"}'

```

## Virtual media management and booting with an ISO

Finally, we want to provision the VM with an ISO. Something we usually do in a Baremetal lab with a console and GUI for the BMC. This is ok, but it is usually slow and the way you share the iso can have some restrictions. 

Now it is booting from HDD:

```bash
$> curl -s ${bmc}/redfish/v1/Systems/${bmc_server} | jq .Boot.BootSourceOverrideTarget
{
  "BootSourceOverrideEnabled": "Continuous",
  "BootSourceOverrideTarget": "Hdd",
  "BootSourceOverrideTarget@Redfish.AllowableValues": [
    "Pxe",
    "Cd",
    "Hdd"
  ],
  "BootSourceOverrideMode": "UEFI",
  "UefiTargetBootSourceOverride": "/0x31/0x33/0x01/0x01"
}

```

We change that to use first Cd ( to make this having effect, in your sushy conf you have to have SUSHY_EMULATOR_IGNORE_BOOT_DEVICE=false).

```bash
$> curl -s -X PATCH -H 'Content-Type: application/json'     -d '{
      "Boot": {
          "BootSourceOverrideTarget": "Cd",
          "BootSourceOverrideEnabled": "Continuous"
      }
    }'   "${bmc}/redfish/v1/Systems/${bmc_server}" 
$> curl -s ${bmc}/redfish/v1/Systems/${bmc_server} | jq .Boot
{
  "BootSourceOverrideEnabled": "Continuous",
  "BootSourceOverrideTarget": "Cd",
  "BootSourceOverrideTarget@Redfish.AllowableValues": [
    "Pxe",
    "Cd",
    "Hdd"
  ],
  "BootSourceOverrideMode": "UEFI",
  "UefiTargetBootSourceOverride": "/0x31/0x33/0x01/0x01"
}

```

Next it will boot from CD (only once).

Now we use Virtual Media to attach it to the CD with an ISO.

There is no iso linked to the virtual media

```bash
$> curl -s ${bmc}/redfish/v1/Managers/${bmc_server}/VirtualMedia/Cd/ | jq  '[{iso_connected: .Inserted}]'
[
  {
    "iso_connected": false
  }
]
```

We create that connection

```bash
export ISO_IMG=http://0.0.0.0:7800/CentOS-8.5.2111-x86_64-dvd1.iso
$> curl -s -d '{ \
    "Image":"'"${ISO_IMG}"'", \
    "Inserted": true \
}' -H "Content-Type: application/json" \
   -X POST ${bmc}/redfish/v1/Managers/${bmc_server}/VirtualMedia/Cd/Actions/VirtualMedia.InsertMedia


$> curl -s ${bmc}/redfish/v1/Managers/${bmc_server}/VirtualMedia/Cd/ | jq  '[{iso_connected: .Inserted}]'
[
  {
    "iso_connected": true
  }
]
```

*How to serve the iso is not covered here, but you can just check [here]([How do you set up a local testing server? - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn/Common_questions/set_up_a_local_testing_server))*

The iso is connected. Lets power on the server

```bash
$> curl -s -H "Content-Type: application/json"  -X POST ${bmc}/redfish/v1/Systems/${bmc_server}/Actions/ComputerSystem.Reset -d '{"ResetType": "ForceOn"}'
```

![](./assets/2022-01-21-17-10-45-image.png)



## Undo boot from virtual media

Just eject action and boot from hdd



```bash
$> curl -X POST ${bmc}/redfish/v1/Managers/${bmc_server}/VirtualMedia/Cd/Actions/VirtualMedia.EjectMedia


$> curl -s ${bmc}/redfish/v1/Managers/${bmc_server}/VirtualMedia/Cd/ | jq  '[{iso_connected: .Inserted}]'
[
  {
    "iso_connected": false
  }
]

$> curl -s -X PATCH -H 'Content-Type: application/json'     -d '{
      "Boot": {
          "BootSourceOverrideTarget": "Hdd",
          "BootSourceOverrideEnabled": "Continuous"
      }
    }'   "${bmc}/redfish/v1/Systems/${bmc_server}" 


$> curl -s -H "Content-Type: application/json"  -X POST ${bmc}/redfish/v1/Systems/${bmc_server}/Actions/ComputerSystem.Reset -d '{"ResetType": "ForceOff"}'
$> curl -s -H "Content-Type: application/json"  -X POST ${bmc}/redfish/v1/Systems/${bmc_server}/Actions/ComputerSystem.Reset -d '{"ResetType": "ForceOn"}'

```