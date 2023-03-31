# ScriptCed

#! /bin/bash

# Load balance VMs across availability zones

# Variable block

let "randomIdentifier=$RANDOM*$RANDOM"

location=francecentral
vNet="msdocs-vnet-lb-myvnet"
subnet="msdocs-subnet-lb-subnet"
loadBalancerPublicIp="msdocs-public-ip-lb-$randomIdentifier"
ipSku=Standard
zone="1 2 3"
loadBalancer="msdocs-load-balancer-loadbalancer"
frontEndIp="msdocs-front-end-ip-lb-$randomIdentifier"
backEndPool="msdocs-back-end-pool-lb-$randomIdentifier"
probe80="msdocs-port80-health-probe-lb-$randomIdentifier"
loadBalancerRuleWeb="msdocs-load-balancer-rule-port80-$randomIdentifier"
loadBalancerRuleSSH="msdocs-load-balancer-rule-port22-$randomIdentifier"
networkSecurityGroup="msdocs-network-security-group-lb-networksecuritygroup"
networkSecurityGroupRuleSSH="msdocs-network-security-rule-port22-lb-$randomIdentifier"
networkSecurityGroupRuleWeb="msdocs-network-security-rule-port80-lb-$randomIdentifier"
nic="msdocs-nic-lb-$randomIdentifier"
image=UbuntuLTS
mdbserv=mdbserv$randomIdentifier



read -p "Entrez le nom de votre Groupe de Ressource: " GroupeDeRessource

read -p "Entrez le nom de votre VM: " vm

read -p "Entrez votre nom d'Utilisateur: " azureuser

read -p "Entrez votre nom d'utilsateur pour MariaDB: " UserMDB

# read -p "Entrez votre mot de passe pour MariaDB: "


# Create a resource group
echo "Creating $GroupeDeRessource in France Central"

az group create \
    --name $GroupeDeRessource \
    --location $location


# Create a virtual network and a subnet.
echo "Creating $vNet and $subnet"

az network vnet create \
    --resource-group $GroupeDeRessource \
    --name $vNet \
    --location $location \
    --subnet-name $subnet

# Create a zonal Standard public IP address for load balancer.
echo "Creating $loadBalancerPublicIp"

az network public-ip create \
    --resource-group $GroupeDeRessource \
    --name $loadBalancerPublicIp \
    --sku $ipSku \
    --zone $zone

# Create an Azure Load Balancer.
echo "Creating $loadBalancer with $frontEndIP and $backEndPool"

az network lb create \
    --resource-group $GroupeDeRessource \
    --name $loadBalancer \
    --public-ip-address $loadBalancerPublicIp \
    --frontend-ip-name $frontEndIp \
    --backend-pool-name $backEndPool \
    --sku $ipSku

# Create an LB probe on port 80.
echo "Creating $probe80 in $loadBalancer"

az network lb probe create \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer \
    --name $probe80 \
    --protocol tcp \
    --port 80

# Create an LB rule for port 80.
echo "Creating $loadBalancerRuleWeb for $loadBalancer"

az network lb rule create \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer \
    --name $loadBalancerRuleWeb \
    --protocol tcp \
    --frontend-port 80 \
    --backend-port 80 \
    --frontend-ip-name $frontEndIp \
    --backend-pool-name $backEndPool \
    --probe-name $probe80
#    --disable-outbound-snat true \
#    --idle-timeout 15 \
#    --enable-tcp-reset true

# Create two NAT rules for port 22.
echo "Creating two NAT rules named $loadBalancerRuleSSH"

  for i in `seq 1 2`
  do
    az network lb inbound-nat-rule create \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer \
    --name $loadBalancerRuleSSH$i \
    --protocol tcp \
    --frontend-port 422$i \
    --backend-port 22 \
    --frontend-ip-name $frontEndIp
  done

# Create a network security group
echo "Creating $networkSecurityGroup"

az network nsg create \
    --resource-group $GroupeDeRessource \
    --name $networkSecurityGroup

# Create a network security group rule for port 22.
echo "Creating $networkSecurityGroupRuleSSH in $networkSecurityGroup for port 22"

az network nsg rule create \
    --resource-group $GroupeDeRessource \
    --nsg-name $networkSecurityGroup \
    --name $networkSecurityGroupRuleSSH \
    --protocol tcp \
    --direction inbound \
    --source-address-prefix '*' \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 22 \
    --access allow \
    --priority 1000


# Create a network security group rule for port 80.
echo "Creating $networkSecurityGroupRuleWeb in $networkSecurityGroup for port 22"

az network nsg rule create \
    --resource-group $GroupeDeRessource \
    --nsg-name $networkSecurityGroup \
    --name $networkSecurityGroupRuleWeb \
    --protocol tcp \
    --direction inbound \
    --priority 1001 \
    --source-address-prefix '*' \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 80 \
    --access allow \
    --priority 2000

# Create two virtual network cards and associate with public IP address and NSG.
echo "Creating two NICs named $nic for $vNet and $subnet"

  for i in `seq 1 2`
  do
    az network nic create \
    --resource-group $GroupeDeRessource \
    --name $nic$i \
    --vnet-name $vNet \
    --subnet $subnet \
    --network-security-group $networkSecurityGroup \
    --lb-name $loadBalancer \
    --lb-address-pools $backEndPool \
    --lb-inbound-nat-rules $loadBalancerRuleSSH$i
  done

# Create two virtual machines, this creates SSH keys if not present.
echo "Creating two VMs named $vm with $nic using $image"

  for i in `seq 1 2`
  do
    az vm create \
    --resource-group $GroupeDeRessource \
    --name $vm$i \
    --zone $i \
    --nics $nic$i \
    --image $image \
    --admin-username $azureuser \
    --generate-ssh-keys \
    --no-wait
  done

  for i in `seq 1 2`
  do
    az vm open-port \
    --port 80 \
    --resource-group $GroupeDeRessource \
    --name $vm$i
  done


# List the virtual machines

az vm list \
    --resource-group $GroupeDeRessource

# Test the load balancer


IpPublic=$(az network public-ip show \
    --resource-group $GroupeDeRessource \
    --name $loadBalancerPublicIp \
    --query ipAddress \
    --output tsv)

echo $IpPublic

az network nat gateway create \
    --resource-group $GroupeDeRessource \
    --name myNATgateway \
    --public-ip-addresses myNATgatewayIP \
    --idle-timeout 10

az network vnet subnet update \
    --resource-group $GroupeDeRessource \
    --vnet-name myVNet \
    --name myBackendSubnet \
    --nat-gateway myNATgateway

# Variables



read -p "Entrez le nom d'Utilisateur de votre Utilisateur MYSQL: " mysqladmin

mysqldb=guacamoledb
mysqlpassword=PIcciNO69200!MaRaTEa?
vnet=msdocs-vnet-lb-myvnet
snet=msdocs-subnet-lb-subnet
avset=guacamoleAvSet
networkSecurityGroup=msdocs-network-security-group-lb-networksecuritygroup
lbguacamolepip=lbguacamolepip
pipdnsname=loadbalancerguacamole
lbname=msdocs-load-balancer-loadbalancer
1=1
2=2
probe443="msdocs-port443-health-probe-lb-$randomIdentifier"

az mysql server create \
    --resource-group $GroupeDeRessource \
    --name $mysqldb \
    --location $location \
    --admin-user $mysqladmin \
    --admin-password $mysqlpassword \
    --sku-name B_Gen5_1 \
    --storage-size 51200 \
    --ssl-enforcement Disabled


az mysql server firewall-rule create \
    --resource-group $GroupeDeRessource \
    --server $mysqldb \
    --name AllowYourIP \
    --start-ip-address 0.0.0.0 \
    --end-ip-address 255.255.255.255



az network nsg rule create \
    --resource-group $GroupeDeRessource \
    --nsg-name $networkSecurityGroup \
    --name msdocs-network-security-rule-port80-lb-$randomIdentifier \
    --protocol tcp \
    --direction inbound \
    --source-address-prefix Internet \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 80 \
    --access allow \
    --priority 200


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i  \
    --command-id RunShellScript \
    --scripts "wget https://raw.githubusercontent.com/ricmmartins/apache-guacamole-azure/main/guac-install.sh -O /tmp/guac-install.sh" 
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i  \
    --command-id RunShellScript \
    --scripts "sudo sed -i.bkp -e 's/mysqlpassword/$mysqlpassword/g' \
    -e 's/mysqldb/$mysqldb/g' \
    -e 's/mysqladmin/$mysqladmin/g' /tmp/guac-install.sh"
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i \
    --command-id RunShellScript \
    --scripts "/bin/bash /tmp/guac-install.sh"
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i \
    --command-id RunShellScript \
    --scripts "sudo apt install \
    --yes nginx-core"
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i \
    --command-id RunShellScript \
    --scripts "cat <<'EOT' > /etc/nginx/sites-enabled/default
# Nginx Config
    server {
        listen 80;
        server_name _;

        location / {


                proxy_pass http://localhost:8080/;
                proxy_buffering off;
                proxy_http_version 1.1;
                proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
                proxy_set_header Upgrade \$http_upgrade;
                proxy_set_header Connection \$http_connection;
                access_log off;
        }
}
  EOT"
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i \
    --command-id RunShellScript \
    --scripts "sudo systemctl restart nginx"
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i \
    --command-id RunShellScript \
    --scripts "sudo systemctl restart tomcat8"
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i \
    --command-id RunShellScript \
    --scripts "sudo /bin/rm -rf /var/lib/tomcat7/webapps/ROOT/* && sudo /bin/cp -pr /var/lib/tomcat8/webapps/guacamole/* /var/lib/tomcat8/webapps/ROOT/"
  done


az network nic ip-config inbound-nat-rule add \
    --inbound-nat-rule ssh1 \
    --ip-config-name ipconfigGuacamole-VM1 \
    --nic-name $nic$1 \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer


az network nic ip-config inbound-nat-rule add \
    --inbound-nat-rule ssh2 \
    --ip-config-name ipconfigGuacamole-VM2 \
    --nic-name $nic$2 \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer


# You try to access the client at http://<loadbalancer-public-ip> or 
# http://<loadbalancer-public-ip-dns-name> and you should see the Guacamole's 
# login screen and use the default user and password (guacadmin/guacadmin) 
# to login:


# Adding SSL in 5 steps

  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i  \
    --command-id RunShellScript \
    --scripts "sudo sed -i.bkp -e 's/_;/myguacamolelab.com;/g' /etc/nginx/sites-enabled/default" 
  done    

export DOMAIN_NAME="myguacamolelab.com"
export EMAIL="admin@myguacamolelab.com"


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i  \
    --command-id RunShellScript \
    --scripts "sudo snap install core; sudo snap refresh core" 
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i  \
    --command-id RunShellScript \
    --scripts "sudo snap install --classic certbot" 
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i  \
    --command-id RunShellScript \
    --scripts "sudo certbot --nginx -d "${DOMAIN_NAME}" -m "${EMAIL}" --agree-tos -n" 
  done


  for i in `seq 1 2`
  do
    az vm run-command invoke -g $GroupeDeRessource -n $vm$i  \
    --command-id RunShellScript \
    --scripts "sudo systemctl restart nginx" 
  done


az network nsg rule create \
    --resource-group $GroupeDeRessource \
    --nsg-name $networkSecurityGroup \
    --name msdocs-network-security-rule-port443-lb-$randomIdentifier \
    --access Allow \
    --protocol Tcp \
    --direction Inbound \
    --priority 300 \
    --source-address-prefix Internet \
    --source-port-range "*" \
    --destination-address-prefix "*" \
    --destination-port-range 443



az network lb probe create \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer \
    --name healthprobe-https \
    --protocol "https" \
    --port 443 \
    --path / \
    --interval 15 


az network lb rule create \
    --resource-group $GroupeDeRessource \
    --lb-name $loadBalancer \
    --name lbrule-https \
    --protocol tcp \
    --frontend-port 443 \
    --backend-port 443 \
    --frontend-ip-name $frontEndIp \
    --backend-pool-name $backEndPool \
    --probe-name $probe443 \
    --load-distribution SourceIPProtocol


# Now you can access through https://<yourdomainname.com>


az mariadb server create \
    --resource-group $GroupeDeRessource \
    --name $mdbserv \
    --location francecentral \
    --admin-user $UserMDB \
    --admin-password PIcciNO69200!MaRaTEa? \
    --sku-name GP_Gen5_2 \
    --version 10.2

az mariadb server firewall-rule create \
    --resource-group $GroupeDeRessource \
    --server $mdbserv \
    --name AllowMyIP \
    --start-ip-address 0.0.0.0 \
    --end-ip-address 255.255.255.255

az mariadb server show \
    --resource-group $GroupeDeRessource \
    --name $mdbserv
