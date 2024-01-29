az group create --name CreatePubLBQS-rg --location eastus

az network vnet create --resource-group CreatePubLBQS-rg --location eastus --name myVNet --address-prefixes 10.1.0.0/16 --subnet-name myBackendSubnet --subnet-prefixes 10.1.0.0/24

az network public-ip create --resource-group CreatePubLBQS-rg --name myPublicIP --sku Standard --zone 1 2 3

az network lb create --resource-group CreatePubLBQS-rg --name myLoadBalancer --sku Standard --public-ip-address myPublicIP --frontend-ip-name myFrontEnd --backend-pool-name myBackEndPool

az network lb probe create --resource-group CreatePubLBQS-rg --lb-name myLoadBalancer --name myHealthProbe --protocol tcp --port 80

az network lb rule create --resource-group CreatePubLBQS-rg --lb-name myLoadBalancer --name myHTTPRule --protocol tcp --frontend-port 80 --backend-port 80 --frontend-ip-name myFrontEnd --backend-pool-name myBackEndPool --probe-name myHealthProbe --disable-outbound-snat true --idle-timeout 15 --enable-tcp-reset true

az network nsg create --resource-group CreatePubLBQS-rg --name myNSG

az network nsg rule create --resource-group CreatePubLBQS-rg --nsg-name myNSG --name myNSGRuleHTTP --protocol '*' --direction inbound --source-address-prefix '*' --source-port-range '*' --destination-address-prefix '*' --destination-port-range 80 --access allow --priority 200

az network public-ip create --resource-group CreatePubLBQS-rg --name myBastionIP --sku Standard --zone 1 2 3

az network vnet subnet create --resource-group CreatePubLBQS-rg --name AzureBastionSubnet --vnet-name myVNet --address-prefixes 10.1.1.0/27

az network bastion create --resource-group CreatePubLBQS-rg --name myBastionHost --public-ip-address myBastionIP --vnet-name myVNet --location eastus

array=(myNicVM1 myNicVM2)
for vmnic in "${array[@]}"
do
    az network nic create --resource-group CreatePubLBQS-rg --name $vmnic --vnet-name myVNet --subnet myBackEndSubnet --network-security-group myNSG
done

az vm create --resource-group CreatePubLBQS-rg --name myVM1 --nics myNicVM1 --image win2019datacenter --admin-username azureuser --zone 1 --no-wait
Localhost@1234567

az vm create --resource-group CreatePubLBQS-rg --name myVM2 --nics myNicVM2 --image win2019datacenter --admin-username azureuser --zone 2 --no-wait
Localhost@1234567

array=(myNicVM1 myNicVM2)
for vmnic in "${array[@]}"
do
    az network nic ip-config address-pool add --address-pool myBackendPool --ip-config-name ipconfig1 --nic-name $vmnic --resource-group CreatePubLBQS-rg --lb-name myLoadBalancer
done

az network public-ip create --resource-group CreatePubLBQS-rg --name myNATgatewayIP --sku Standard --zone 1 2 3

az network nat gateway create --resource-group CreatePubLBQS-rg --name myNATgateway --public-ip-addresses myNATgatewayIP --idle-timeout 10

az network vnet subnet update --resource-group CreatePubLBQS-rg --vnet-name myVNet --name myBackendSubnet --nat-gateway myNATgateway

array=(myVM1 myVM2)
for vm in "${array[@]}"
do
    az vm extension set --publisher Microsoft.Compute --version 1.8 --name CustomScriptExtension --vm-name $vm --resource-group CreatePubLBQS-rg --settings '{"commandToExecute":"powershell Add-WindowsFeature Web-Server; powershell Add-Content -Path \"C:\\inetpub\\wwwroot\\Default.htm\" -Value $($env:computername)"}'
done

az network public-ip show --resource-group CreatePubLBQS-rg --name myPublicIP --query ipAddress --output tsv
-----------------------------
az group delete --name CreatePubLBQS-rg




  
