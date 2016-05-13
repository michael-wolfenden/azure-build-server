# ARM template to create a build server in Azure

## Introduction

Personal project to create an ARM template to spin-up a build server in Azure.

NOTE: This is a work in progress. The ultimate goal is that this will do everything from creating a new vm and sql server database and then install the holy trinity of TeamCity, Octopus Deploy and Seq.

## Instructions

Below are the parameters that the template expects

<table>
    <thead>
        <tr>
            <th>Name</th>
            <th>Type</th>
            <th>Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>vmAdminUsername</td>
            <td>string</td>
            <td>Administrator username for the virtual machine.</td>
        </tr>
        <tr>
            <td>vmAdmvmAdminPasswordinUsername</td>
            <td>string</td>
            <td>Administrator password for the virtual machine.</td>
        </tr>
        <tr>
            <td>vmStorageAccountType</td>
            <td>string</td>
            <td>
                The type of storage account
                <ul>
                    <li>Standard_LRS = Locally-redundant storage</li>
                    <li>Standard_GRS = Geo-redundant storage</li>
                    <li>Standard_ZRS = Zone-redundant storage</li>
                </ul>
            </td>
        </tr>
        <tr>
            <td>vmDnsName</td>
            <td>string</td>
            <td>unique public DNS name for the virtual machine. The fqdn will look something like '&lt;vmDnsName&gt;.&lt;resouce group location&gt;.cloudapp.azure.com'. Up to 62 chars, digits or dashes, lowercase, should start with a letter: must conform to '^[a-z][a-z0-9-]{1,61}[a-z0-9]$'.</td>
        </tr>
        <tr>
            <td>vmSize</td>
            <td>string</td>
            <td>VM size, for example "Standard_A2" (see https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-windows-sizes/).</td>
        </tr>
        <tr>
            <td>resourcePrefix</td>
            <td>string</td>
            <td>The name of all resources (except storage) wil be prefixed with this. Can be blank to not have a prefix.</td>
        </tr>
        <tr>
            <td>resourceSuffix</td>
            <td>string</td>
            <td>The name of all resource (except storage) wil be suffixed with this. Can be blank to not have a suffix.</td>
        </tr>
    </tbody>
</table>

## Deploying

Populate the `template.json` file will the parameter values you require and then run

> ./deploy.ps1 $subscriptionId $resourceGroupName $resourceGroupLocation $deploymentName

NOTE: If the resouce group does not exist, it will be created

## What gets created

Lets say you have set

* `resourcePrefix` = `prefix-`
* `resourceSuffix` = `-suffix`

The following resources will be created

* Virtual machine named  `prefix-vm-suffix` 
* Network interface named  `prefix-nic-suffix` 
* Network security group named `prefix-nsg-suffix`. The will allow inbound traffic on ports 3389 (RDP), 80 (http), 443 (https)
* Public IP address named `prefix-pip-suffix`. This will have the dns name provided in the parameters '&lt;vmDnsName&gt;.&lt;resouce group location&gt;.cloudapp.azure.com'
* Virtual Network named `prefix-vnet-suffix`.
* Storage account named `sto<unique based on subscription>`. For example `stoufa7qk2m4n6ye`

The reason the naming of storage acocunt differs is that it is actually a dns name and has a bunch of restrictions of characters and length. Using the same naming scheme as the other resources caused issue where the name wouldn't be unique or it was too long. This is the workaround.

## Vanity Urls

At the end of this process you have a dns name for the virtual machine, for instance:

http://mybuildserver.australiaeast.cloudapp.azure.com'

and three applications listening on different ports

* seq : http://mybuildserver.australiaeast.cloudapp.azure.com:5341
* TeamCity : http://mybuildserver.australiaeast.cloudapp.azure.com:8080
* Octopus Deploy : http://mybuildserver.australiaeast.cloudapp.azure.com:8081

If you own a domain, for instance

http://mydomain.com

Wouldn't it be nicer to have

* seq : https://seq.mydomain.com
* TeamCity : https://teamcity.mydomain.com
* Octopus Deploy : https://octopus.mydomain.com

To do this, firstly setup `CNAME` records as follows

```
CNAME | seq | mybuildserver.australiaeast.cloudapp.azure.com
CNAME | teamcity | mybuildserver.australiaeast.cloudapp.azure.com
CNAME | octopus | mybuildserver.australiaeast.cloudapp.azure.com
```

To make this work, we need a reverse proxy running on the virtual machine. You could use IIS, but a far better option is [Caddy](https://caddyserver.com) (a HTTP/2 web server with automatic HTTPS).

Download and extract to a folder, then in that folder create the following `Caddyfile`:

```
seq.mydomain.com {
    proxy / localhost:5341 {
        proxy_header Host {host}
    }
}

teamcity.mydomain.com {
    proxy / localhost:8080 {
        proxy_header Host {host}
        websocket
    }
}

octopus.mydomain.com {
    proxy / localhost:8081 {
        proxy_header Host {host}
    }
}
```

The beauty of this, is the first time Caddy runs, it will prompt you for some information and then automatically enable HTTPS for all your sites (using autogenerated certs from Let's Encrypt). It will also redirect all HTTP requests to their HTTPS equivalent.

Let me say that again, Caddy will automatically handle certificate registration and renewal. No more mucking around with java keystores to get SSL working with teamcity. BOOM!!!!!

Currently though, Caddy only runs as an exe (they are adding the option to run as a windows service). To get around this use [NSSM - the Non-Sucking Service Manager](https://nssm.cc/). Exract the exe into the same directory as Caddy and run `nssm install Caddy`. Configure the parameters and your good to go, you now have have Caddy running as a windows service.

NOTE: You need to add an inbound firewall rule for ports 80 and 443 for Caddy to work

