Typical Application:
	App1 --> WebServer --> VM(OS) --> IP

Why Azure Load Balancer:
> Load Balance internal/external 
	traffic(Request/Response) to Azure VM
> Increase availability distribute resources
	within + across = Zones
> Configure outbound connectivity to LB resources
	Internet --> Incoming traffic --> VM  =Inbound
	Internet <-- Outgoing traffic <-- VM  =Outbound
> Use health probes to monitor LB resources(VMs)
> Port Forwarding

Project Steps: 
==============
1) Subscription --> Resource Group
	# az group create \
		--name CreatePubLBQS-rg \
		--location eastus
2) VNet(10.1.0.0/16) --> myPubIP
	# az network vnet create \
	--resource-group CreatePubLBQS-rg \
	--location eastus \
	--name myVNet \
	--address-prefixes 10.1.0.0/16 \
	--subnet-name myBackendSubnet \
	--subnet-prefixes 10.1.0.0/24
	
	Full CIDR Block: 10.1.0.0/26
	Network Type:  10.1.0.
	IPV4 : X.X.X.X = 8.8.8.8 =   32 - 26 =2^6 = 64 
	Start IP Address:	10.1.0.0
	End IP Address 	:	10.1.0.63

	Full CIDR Block: 10.1.0.0/28
	Network Type:  10.1.0.
	IPV4 : X.X.X.X = 8.8.8.8 =   32 - 28 =2^4 = 16 
	Start IP Address:	10.1.0.0
	End IP Address 	:	10.1.0.15
	
	# az network public-ip create \
		--resource-group CreatePubLBQS-rg \
		--name myPublicIP \
		--sku Standard \
		--zone 1

3) LB --> HP --> LB Route
	1. Load Balancer:
		Frontend Public IP
		Backend Public IP
		# az network lb create \
			--resource-group CreatePubLBQS-rg \
			--name myLBMujahed \
			--sku Standard \
			--public-ip-address myPublicIP \
			--frontend-ip-name myFrontEnd \
			--backend-pool-name myBackEndPool
	
	2. Health Probe
		# az network lb probe create \
			--resource-group CreatePubLBQS-rg \
			--lb-name myLBMujahed \
			--name myHealthProbe \
			--protocol tcp \
			--port 80
	3. LB Rule
		# az network lb rule create \
		--resource-group CreatePubLBQS-rg \
		--lb-name myLBMujahed \
		--name myHTTPRule \
		--protocol tcp --frontend-port 80 \
		--backend-port 80 \
		--frontend-ip-name myFrontEnd \
		--backend-pool-name myBackEndPool \
		--probe-name myHealthProbe \
		--disable-outbound-snat true \
		--idle-timeout 15 --enable-tcp-reset true
	
4) NSG --> NSG Rule
	# az network nsg create \
		--resource-group CreatePubLBQS-rg \
		--name myNSG
	# az network nsg rule create \
		--resource-group CreatePubLBQS-rg \
		--nsg-name myNSG \
		--name myNSGRuleHTTP \
		--protocol '*' --direction inbound \
		--source-address-prefix '*' \
		--source-port-range '*' \
		--destination-address-prefix '*' \
		--destination-port-range 80 \
		--access allow --priority 200
	
5) Bastion Configuration
	Bastion PubIP  - MyBastionIP
	# az network public-ip create \
		--resource-group CreatePubLBQS-rg \
		--name myBastionIP \
		--sku Standard --zone 1 2 3
	Bastion Subnet - AzBastionSubnet
	# az network vnet subnet create \
	    --resource-group CreatePubLBQS-rg \
	    --name AzureBastionSubnet \
	    --vnet-name myVNet \
	    --address-prefixes 10.1.1.0/27
	Create Bastion Host - myBastionHost
	# az network bastion create \
	    --resource-group CreatePubLBQS-rg \
	    --name myBastionHost \
	    --public-ip-address myBastionIP \
	    --vnet-name myVNet \
	    --location eastus
	    
6) Backend Subnet:
    Create NIC  with name as MyNicVM1, MyNicVM2
    # array=(myNicVM1 myNicVM2)
    for vmnic in "${array[@]}"
    do
        az network nic create \
            --resource-group CreatePubLBQS-rg \
            --name $vmnic \
            --vnet-name myVNet \
            --subnet myBackEndSubnet \
            --network-security-group myNSG
    done  
    
	VM1
	# az vm create \
	    --resource-group CreatePubLBQS-rg \
	    --name myVM1 \
	    --nics myNicVM1 \
	    --image win2019datacenter \
	    --admin-username azureuser \
	    --zone 1 --no-wait
	    Localhost@1234567
	    
	VM2
	# az vm create \
	    --resource-group CreatePubLBQS-rg \
	    --name myVM2 \
	    --nics myNicVM2 \
	    --image win2019datacenter \
	    --admin-username azureuser \
	    --zone 2 --no-wait

	LB <-- VM1,VM2 <-- MyNicVM1,MyNicVM2
	# array=(myNicVM1 myNicVM2)
    for vmnic in "${array[@]}"
    do
        az network nic ip-config address-pool add \
            --address-pool myBackendPool \
            --ip-config-name ipconfig1 \
            --nic-name $vmnic \
            --resource-group CreatePubLBQS-rg \
            --lb-name myLBMujahed
    done
    
7) NAT Gateway
	MyNATGWIp
	# az network public-ip create \
	    --resource-group CreatePubLBQS-rg \
	    --name myNATgatewayIP \
	    --sku Standard \
	    --zone 1 2 3
	Create MyNATGW
	# az network nat gateway create \
	    --resource-group CreatePubLBQS-rg \
	    --name myNATgateway \
	    --public-ip-addresses myNATgatewayIP \
	    --idle-timeout 10
    Update MyNATGW --> BackendSubnet
    # az network vnet subnet update \
        --resource-group CreatePubLBQS-rg \
        --vnet-name myVNet \
        --name myBackendSubnet \
        --nat-gateway myNATgateway 
    
8) WebServer(IIS) <-- HelloWorld <-- VM1,VM2
    array=(myVM1 myVM2)
    for vm in "${array[@]}"
    do
        az vm extension set \
            --publisher Microsoft.Compute \
            --version 1.8 --name CustomScriptExtension \
            --vm-name $vm --resource-group CreatePubLBQS-rg \
            --settings '{"commandToExecute":"powershell Add-WindowsFeature Web-Server; powershell Add-Content -Path \"C:\\inetpub\\wwwroot\\Default.htm\" -Value $($env:computername)"}'
    done
9) Test
    # az network public-ip show \
        --resource-group CreatePubLBQS-rg \
        --name myPublicIP \
        --query ipAddress \
        --output tsv
    # curl http://PublicIP/Default.htm
        myVM2
    # Stop VM1 and check again
        myVM1
    # curl http://PublicIP/Default.htm
        myVM2
    
10) Delete All Resources in Resource Group
    # az group delete --name CreatePubLBQS-rg



























