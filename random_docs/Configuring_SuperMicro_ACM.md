# Supermicro servers and automatic deployments with ACM/HIVE/ZTP

This tutorial tries to help about SuperMicro servers to make deployments with ACM/HIVE/ZTP. The main problem is the Supermicro Redfish API implementation. 

In these deployments, an installation ISO is created and mounted into the Virtualmedia. All the process is done using the BMC Redfish implementation of Supermicro. 

Metal3 will be, usually, in charge of managing this process from the installation image creation/customization and server booting. Until, the installation starts. 

## SBI-4129P-C2N (I think X10 series)

### Boot mode issue

One the first things done by Metal3 is to configure the server to boot (once) with the CD. But here appears the first error:

```log
  errorMessage: 'Image provisioning failed: Deploy step deploy.deploy failed: Redfish
    exception occurred. Error: Setting boot mode to uefi failed for node 3c33dfa7-6950-4a90-a939-d3b71c546863.
    Error: HTTP PATCH https://10.19.10.71:9000/redfish/v1/Systems/1 returned code
    400. Base.1.0.0.GeneralError: The value UEFI for the property BootSourceOverrideMode
    is not in the list of acceptable values. Extended information: [{''MessageId'':
    ''Base.1.0.0.PropertyValueNotInList'', ''Severity'': ''Warning'', ''Resolution'':
    ''Choose a value from the enumeration list that the implementation can support
    and resubmit the request if the operation failed.'', ''Message'': ''The value
    UEFI for the property BootSourceOverrideMode is not in the list of acceptable
    values.'', ''MessageArgs'': [''UEFI'', ''BootSourceOverrideMode''], ''RelatedProperties'':
    [''Boot'']}].'
```

What Metal3 is trying to do is:

```bash
>  curl -s -k -X PATCH -H 'Content-Type: application/json'     -d '{
      "Boot": {
          "BootSourceOverrideMode": "UEFI"
      }
    }'   "${bmc}/redfish/v1/Systems/${bmc_server}" | jq
{
  "error": {
    "code": "Base.1.0.0.GeneralError",
    "Message": "A general error has occurred. See ExtendedInfo for more information.",
    "@Message.ExtendedInfo": [
      {
        "MessageId": "Base.1.0.0.PropertyValueNotInList",
        "Severity": "Warning",
        "Resolution": "Choose a value from the enumeration list that the implementation can support and resubmit the request if the operation failed.",
        "Message": "The value UEFI for the property BootSourceOverrideMode is not in the list of acceptable values.",
        "MessageArgs": [
          "UEFI",
          "BootSourceOverrideMode"
        ],
        "RelatedProperties": [
          "Boot"
        ]
      }
    ]
  }
}
```

Metal3 is trying to change the "BootSourceOverrideMode" from "Legacy" to "UEFI". The error says, it is because "UEFI" is not a valid option, but this is not true. 

```bash
> curl -s -k ${bmc}/redfish/v1/Systems/${bmc_server} | jq .Boot
{
  "BootSourceOverrideEnabled": "Once",
  "BootSourceOverrideMode": "Legacy",
  "BootSourceOverrideMode@Redfish.AllowableValues": [
    "Legacy",
    "UEFI"
  ],
  "BootSourceOverrideTarget": "None",
  "BootSourceOverrideTarget@Redfish.AllowableValues": [
    "None",
    "Pxe",
    "Hdd",
    "Cd",
    "BiosSetup",
    "Usb",
    "Floppy",
    "Cd"
  ]
}
```

UEFI is a valid option.

This seem to be an issue in the Redfish API implementation. Which requires to set, at the same time, "BootSourceOverrideEnabled: Once" and "BootSourceOverrideTarget: Cd".
This other request will success about setting the boot options.

```bash
$> curl -s -k -X PATCH -H 'Content-Type: application/json'     -d '{
      "Boot": {
          "BootSourceOverrideEnabled": "Once",
          "BootSourceOverrideMode": "UEFI",
          "BootSourceOverrideTarget": "Cd"
      }
    }'   "https://root:100Mgmt-@10.19.143.195/redfish/v1/Systems/${bmc_server}" | jq
{
  "Success": {
    "code": "Base.1.0.0.Success",
    "Message": "Successfully Completed Request."
  }
}
```

Metal3 is not aware of that, it will try to change first the BootSourceOverrideMode, and it will fail. 

#### Solution

We can use [fakefish project](https://github.com/openshift-metal3/fakefish), which acts as a proxy of Redfish requests. It allows you to customize the requests according to your server. It already includes [scripts](https://github.com/openshift-metal3/fakefish/tree/main/supermicro_scripts) for Super Micro servers, but these were failing for me. So, I created my own with the scripts for this server [here](https://github.com/jgato/fakefish/tree/main/supermicro_scripts)

>  99% the same, just a change on using the word Cd instead of CD. 

Witht this proxy, 
you can see on the ['bootfromcdonce.sh'](https://github.com/jgato/fakefish/blob/main/supermicro_scripts/bootfromcdonce.sh#L10) script, how it sets the boot three parameters at once.

I have also created a custom [container image](https://quay.io/repository/jgato/fakefish-supermicro) that already includes the supermicro scripts. 

So, run fakefish container on a server in between your ACM and the BMC. Just changing the IP of the BMC

```bash
podman run --name fakefish --rm -p 9000:9000 quay.io/jgato/fakefish-supermicro:latest --listen-port 9000 --remote-bmc 10.19.143.195
```

> You have to one containers for each BMC.  

Then, in your ACM/ZTP deployments, you point to the endpoint generated by fakefish, instead of the usual BMC Address. Something like: 'redfish-virtualmedia://:/redfish/v1/Systems/1' or 'https://<fake_fish_host>:<fake_fish_port>/redfish/v1/Systems/1'

Try now the deployment, and you will see how the boot mode error disappear.

### CD not correctly mounted issue [Not solved]

So now, the boot order seems to work oka, or at least, no errors. But there are other issues not solved.

* boot order oka, CD configuration pointing to an http iso oka, but the CD is not mounted

* boot order oka, CD configuration pointing to an http iso oka, but the CD is mounted by an strange Applet

In both cases, when Metal3 boots the server, this will not boot from the CD Virtualmedia and the installation ISO.

But mostly, everything was configured correctly:

* Boot order is oka, because of the fakefish work

* Virtualmedia is configured to use the http iso:



```bash
$ curl -s -X GET  -k -u ''"${BMC_USERNAME}"'':''"${BMC_PASSWORD}"'' https://${BMC_ENDPOINT}/redfish/v1/Managers/1/VM1/CfgCD | jq
{
  "@odata.context": "/redfish/v1/$metadata#IsoConfig.IsoConfig",
  "@odata.type": "#IsoConfig.IsoConfig",
  "@odata.id": "/redfish/v1/Managers/1/VM1/CfgCD",
  "Id": "1",
  "Name": "VM1",
  "Host": "http://10.19.10.75:6180",
  "Path": "/redfish/boot-cbb7c9a0-f71f-4a7d-8523-367acfb2aea3.iso",
  "Username": "",
  "Password": "********",
  "Actions": {
    "#IsoConfig.Mount": {
      "target": "/redfish/v1/Managers/1/VM1/CfgCD/Actions/IsoConfig.Mount"
    },
    "#IsoConfig.UnMount": {
      "target": "/redfish/v1/Managers/1/VM1/CfgCD/Actions/IsoConfig.UnMount"
    }
  }
}
```

* The Virtualmedia can be mounted:

```bash
$ curl -X POST -s -k -u ''"${BMC_USERNAME}"'':''"${BMC_PASSWORD}"'' https://${BMC_ENDPOINT}/redfish/v1/Managers/1/VM1/CfgCD/Actions/IsoConfig.UnMount -d ""
{"Success":{"code":"Base.1.0.0.Success","Message":"Successfully Completed Request."}}[jgato@provisioner supermicro_scripts]$ curl -X POST -s -k -u ''"${BMC_USERNAME}"'':''"${BMC_PASSWORD}"'' https://${BMC_ENDPOINT}/redfish/v1/Managers/1/VM1/CfgCD/Actions/IsoConfig.UnMount -d "" | jq
{
  "Success": {
    "code": "Base.1.0.0.Success",
    "Message": "Successfully Completed Request."
  }
}
```

* But after that, if you check how it was mounted. You receive an empty reply. So, not really mounted. And it will not boot correctly.

```bash
$ curl -s  -k -u ''"${BMC_USERNAME}"'':''"${BMC_PASSWORD}"'' https://${BMC_ENDPOINT}/redfish/v1/Managers/1/VM1/CD1 |jq
```

* Or you see that it has been mounted an 'strange' iso by an Applet



```bash
$ curl -s  -k -u ''"${BMC_USERNAME}"'':''"${BMC_PASSWORD}"'' https://${BMC_ENDPOINT}/redfish/v1/Managers/1/VM1/CD1 |jq
{
  "@odata.context": "/redfish/v1/$metadata#VirtualMedia.VirtualMedia",
  "@odata.type": "#VirtualMedia.VirtualMedia",
  "@odata.id": "/redfish/v1/Managers/1/VM1/CD1",
  "Id": "CD1",
  "Name": "Virtual Removable Media",
  "MediaTypes": [
    "CD",
    "DVD"
  ],
  "ImageName": "mymedia-read-only.iso",
  "Inserted": true,
  "WriteProtected": true,
  "ConnecteVia": "Applet"
}
```

#### Solution (but manual work)

In the case the CD1 is never mounted, but CD configuration is oka:

* Do your regular stuff about defining the cluster (SiteConfigs, or usual ACM CRs). 

* The installation ISO is generated and the host is "ready". Ready, because most of the steps have been done (with the previous fixes).

* The server will be boot but the ISO is not correctly mounted. 

* Until here, everything is automatic.

* Now, manually, interrupt the server boot and enter in the boot menu when Bios is loading (F11). Choose to boot from CD. That is has been automatically configured to boot with the installation ISO.

The installation should be done correctly.

If the CD1 is always mounted with that strange applet issue. You cannot do the previous workaround. Because, the CD is configured to boot from the ISO mounted by that applet.

## Different firmware versions

I have played with two different firmware versions.:

Older:

```
Firmware Revision : 3.72.13     
Firmware Build Time : 10/30/2020     
BIOS Version : 3.3a    
BIOS Build Time : 03/16/2020    
Redfish Version : 1.0.1    
CPLD Version : 02.b3.07    
```

newer:

```
Firmware Revision : 3.72.19     
Firmware Build Time : 06/25/2021     
BIOS Version : 3.4b    
BIOS Build Time : 10/08/2021    
Redfish Version : 
CPLD Version : 02.b3.07
```

Nothing to say specially about the different issue. None has worked out of the box. But the newer is showing the 'Applet issue" above.

## Applet issue

Some strange situation, It could be that there is an applet console which mounted an iso and it is not released:

```
$> curl -s -k -X GET -u ''"${BMC_//${BMC_ENDPOINT}/redfish/v1/Managers/1/VM1/CD1 | jq

{
  "@odata.context": "/redfish/v1/$metadata#VirtualMedia.VirtualMedia",
  "@odata.type": "#VirtualMedia.VirtualMedia",
  "@odata.id": "/redfish/v1/Managers/1/VM1/CD1",
  "Id": "CD1",
  "Name": "Virtual Removable Media",
  "MediaTypes": [
    "CD",
    "DVD"
  ],
  "ImageName": "mymedia-read-only.iso",
  "Inserted": true,
  "WriteProtected": true,
  "ConnecteVia": "Applet"
}
```

Not solved yet.