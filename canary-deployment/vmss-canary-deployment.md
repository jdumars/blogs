# Canary Deployment for Virtual Machine Scale Sets (VMSS)

Canary deployment is a pattern that rolls out releases to a subset of users or servers. It deploys the changes to a
small set of servers, which allows you to test and monitor how the new release works before rolling the changes to the
rest of the servers.

Virtual machine scale sets (VMSS) are an Azure compute resource that you can use to deploy and manage a set of identical VMs. 
With all VMs configured the same, scale sets are designed to support true autoscale, and no pre-provisioning of VMs 
is required. So it's easier to build large-scale services that target big compute, large data, and containerized workloads.

VMSS allow you to manage large amounts of identical VMs with simple instructions, yet allow you to update every single VM
along. You can build your VMSS with a customized image or publicly available OS images, and some VM extension scripts so that
they will be executed after the provision of the VM and setup all required environments. When it comes to update existing
VMSS, you need to update its configuration with new image or extension scripts, and then manually trigger the update of the
VMSS instances, either all in one instruction, or selectively pick some VMs to be updated.

The ability to update individual VMs in VMSS allows us to control the amount of VMs that will be updated to the new releases,
i.e., allows us to do canary deployment:

1. (Existing) Create the initial VMSS which hosts your services.
1. Update the VMSS configuration, either point to new customized image, or update the extension scripts, which contains the
   new release of your services.
1. Selectively update individual instances to the new release according to the configuration changes.
1. Verify the new release works.
1. Update the rest of instances to the new release.

## Nginx Canary Deployment Example

Here we demonstrate the canary deployment for VMSS using the Nginx binary release.

### Prepare the VMSS

We use the public Ubuntu Server 16.04 LTS together with an extension script which installs the Nginx service to setup the
VMSS. In your project, you can customize the extension script to install the service on your demand, or create a customized
image (reference: [Packer / Azure Resource Manager Builder](https://www.packer.io/docs/builders/azure.html)).

#### Prepare variable configurations

First we setup some variables that will be used in the following preparation steps. You need to update them on your demand.

```sh
# resource group and VMSS name
resource_group=the-resource-group-name
location=the-resource-location
vmss_name=the-vmss-name

# admin user and SSH login credentials setup
admin_user=azureuser
ssh_pubkey="$(readlink -f ~/.ssh/id_rsa.pub)"

# the storage account that will be created to store the init scripts. Note that '-' is not allowed in a storage account name.
export AZURE_STORAGE_ACCOUNT=the-storage-account-name
```

#### Create VMSS from public Ubuntu image

We create a VMSS with 3 instances, using the public image "UbuntuLTS".

```sh
az group create --name "$resource_group" --location "$location"

# create the VMSS with 3 instances using the public Ubuntu LTS image
az vmss create --resource-group "$resource_group" --name "$vmss_name" \
    --image UbuntuLTS \
    --admin-username "$admin_user" \
    --ssh-key-value "$ssh_pubkey" \
    --vm-sku Standard_D2_v3 \
    --instance-count 3 \
    --lb "${vmss_name}LB"
```

#### Prepare the init scripts

In order to use the custom script extension to configure the VMSS, we need to store the script at some location that's accessible
via HTTP(s). Here we create a storage account for the script storage, and expose the scripts publicly to allow the script extension
to pick it up.

The custom script is fairly simple in this case. It just install the Nginx package from the Ubuntu Apt source. In your project,
you may update the script to fetch dependencies, install, configure and start services, etc.

```sh
# create the storage account and container that store the extension scripts
az storage account create --name "$AZURE_STORAGE_ACCOUNT" --location "$location" --resource-group "$resource_group" --sku Standard_LRS
export AZURE_STORAGE_KEY="$(az storage account keys list --resource-group "$resource_group" --account-name "$AZURE_STORAGE_ACCOUNT" --query '[0].value' --output tsv)"
az storage container create --name init --public-access container

# upload the init script to the blob container
cat <<EOF >install_nginx.sh
#!/bin/bash

sudo apt-get update
sudo apt-get install -y nginx
EOF
az storage blob upload --container-name init --file install_nginx.sh --name install_nginx.sh
init_script_url="$(az storage blob url --container-name init --name install_nginx.sh --output tsv)"
```

#### Install the custom script extension

```sh
# prepare the script config
cat <<EOF >script-config.json
{
  "fileUris": ["$init_script_url"],
  "commandToExecute": "./install_nginx.sh"
}
EOF
# install the CustomScript extension to the VMSS
az vmss extension set \
    --publisher Microsoft.Azure.Extensions \
    --version 2.0 \
    --name CustomScript \
    --resource-group "$resource_group" \
    --vmss-name "$vmss_name" \
    --settings @script-config.json
# update all the instances so that they will have nginx installed
az vmss update-instances --resource-group "$resource_group" --name "$vmss_name" --instance-ids \*
```

#### Update load balancer endpoint

We need to create a load balancer rule to route the public traffic to the Nginx services running in the VMSS
backend.

```sh
# create load balancer rule to allow public access to the backend Nginx service
az network lb probe create \
    --resource-group "$resource_group" \
    --lb-name "${vmss_name}LB" \
    --name nginx \
    --port 80 \
    --protocol Http \
    --path /

az network lb rule create \
    --resource-group "$resource_group" \
    --lb-name "${vmss_name}LB" \
    --name nginx \
    --frontend-port 80 \
    --backend-port 80 \
    --protocol Tcp \
    --backend-pool-name "${vmss_name}LBBEPool" \
    --probe nginx
```

#### Verify everything works

Check that we can access the Nginx service from the public endpoint of the load balancer.

```sh
# check that the Nginx service is working properly
lb_ip=$(az network lb show --resource-group "$resource_group" --name "${vmss_name}LB" --query "frontendIpConfigurations[].publicIpAddress.id" --output tsv | head -n1 | xargs az network public-ip show --query ipAddress --output tsv --ids)
curl -s "$lb_ip" | grep title
#>> <title>Welcome to nginx!</title>
```

### Deploy New Release in Canary Deployment Pattern

In the new release we update the Nginx landing page a bit, and deploy it to 1 instance in the early stage. So after the
deployment, we should have 1 instance serving the updated landing page, and 2 instances serving the original page.

First, we need to update and upload the new custom script. Some points to be aware here:

* The custom script will be executed on a fresh VM after it is created from the given OS image. It is not an incremental
   update process based on the existing VM. So we need to install all the dependencies and services again,
   with the changes in the new release included.
* Any updates you make to your application are not exposed to the Custom Script Extension unless that install script changes.
   To force VMSS to pick up the custom script changes, you need to change the script name so that it results in a different
   file URI.
* This will not affect the existing instances until we manually update those instances.

```sh
# prepare the updated nginx service
cat <<EOF >install_nginx.v1.sh
#!/bin/bash

sudo apt-get update
sudo apt-get install -y nginx
sudo sed -i -e 's/Welcome to nginx/Welcome to nginx on Azure VMSS/' /var/www/html/index*.html
EOF
az storage blob upload --container-name init --file install_nginx.v1.sh --name install_nginx.v1.sh
init_script_url="$(az storage blob url --container-name init --name install_nginx.v1.sh --output tsv)"

# prepare the script config
cat <<EOF >script-config.v1.json
{
  "fileUris": ["$init_script_url"],
  "commandToExecute": "./install_nginx.v1.sh"
}
EOF
# install the CustomScript extension to the VMSS
az vmss extension set \
    --publisher Microsoft.Azure.Extensions \
    --version 2.0 \
    --name CustomScript \
    --resource-group "$resource_group" \
    --vmss-name "$vmss_name" \
    --settings @script-config.v1.json
```

Now that the custom script configuration is update for the VMSS, we can update 1 instance to pick up the new custom script.

```sh
# pick up the first instance ID
instance_id="$(az vmss list-instances --resource-group "$resource_group" --name "$vmss_name" --query '[].instanceId' --output tsv | head -n1)"
# update the instance VM
az vmss update-instances --resource-group "$resource_group" --name "$vmss_name" --instance-ids "$instance_id"
# (optional) check the latest model applied status
az vmss list-instances --resource-group "$resource_group" --name "$vmss_name" | grep latest
#    "latestModelApplied": true,
#    "latestModelApplied": false,
#    "latestModelApplied": false,
```

Check visit the load balancer public endpoint and we should see the old version and new version interleaved.

```sh
curl -s "$lb_ip" | grep title
#>> <title>Welcome to nginx!</title>
curl -s "$lb_ip" | grep title
#>> <title>Welcome to nginx on Azure VMSS!</title>
curl -s "$lb_ip" | grep title
#>> <title>Welcome to nginx!</title>
curl -s "$lb_ip" | grep title
#>> <title>Welcome to nginx!</title>
```

Now you can do more checks to verify if the new version works as expected. If you need to test the updated instance specifically,
you can open an tunnel through the SSH channel, and check the service in detail.

```sh
# obtain the NAT SSH port for the updated instance
ssh_port="$(az network lb inbound-nat-rule show --resource-group "$resource_group" --lb-name "${vmss_name}LB" --name "${vmss_name}LBNatPool.${instance_id}" --query frontendPort --output tsv)"
# map the localhost:8080 endpoint to the remote 80 port through the SSH channel
ssh -L localhost:8080:localhost:80 -p "$ssh_port" azureuser@"$lb_ip"
```

After this you can visit the web page through `http://localhost:8080` and it will show you the page served by the updated instance.

### Complete the Release of the New Version

At this point you have 1 instance serving the updated web page and 2 instances serving the original page in the VMSS.
When you have verified that the new version works, you can update the rest of the instances to the new version.

```sh
az vmss update-instances --resource-group "$resource_group" --name "$vmss_name" --instance-ids \*
```

Note that this will update all the instances whose model is not aligned with the latest state, in parallel. So all the outdated
instances will be brought down, updated, and brought up again. It will not cause service downtime as long as the load balancer noticed
some of the backends are down, as we have at least 1 instance updated in the previous steps. However, during the update window of
the outdated instances, all the client traffic will be routed to up-to-date instances, which will increase the load and latency
on those instances.

The better approach may be querying the outdated instances list first, and then update them with smaller granularity:

```sh
az vmss list-instances --resource-group "$resource_group" --name "$vmss_name" --query '[?latestModelApplied==`false`].instanceId' --output tsv
#>> 2
#>> 4
az vmss update-instances --resource-group "$resource_group" --name "$vmss_name" --instance-ids 2
# wait for instance 2 to be up and running
az vmss update-instances --resource-group "$resource_group" --name "$vmss_name" --instance-ids 4
```

In this way, only a small amount of instances are being updated at a given point of time. The rest of the instances are not
touched and will serve the traffic together.