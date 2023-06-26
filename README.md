# 03-AdvancedSiteToSiteVPN
In this lab I created and tested a VPN connection between an AWS VPC and an On Premise Customer network. <p> 
Architecture: The AWS VPC has 2 subnets in 2 AZs with an EC1 instance in each subnet. 
On the Customer side there are 2 physical routers and 2 servers.
Connection is implemented using a Transit Gateway on the AWS side with 2 VPN attachments. 
This provides us 2 endpoints on AWS side will connect to the two Customer Routers. 
This provides resiliency in case of failure of a customer router or failure of an AZ on the AWS side.

For practicality the customer network is simulated and is hosted on AWS. On the customer side the IPsec tunnels will be setup and configured using StrongSwan. StrongSwan is mature and widely adopted open-sourced IPsec based solution that provides secure communication and VPN capabilities. It supports various authentication methods, such as X.509 certificates, pre-shared keys, and username/password authentication. It adheres to industry standard protocols and specifications including IPsec RFCs, IKE RFCs and other relevant standards. 
Dynamic routing with BGP is set up using Free Range Routing (FRR), an open source routing software suite. Here FRR is used in conjunction with Strongswan to enhance the routing capabilities of this VPN solution. 

A Site to Site VPN has the following components:
AWS side: 1. A VPC
          2. A Virtual Private Gateway on the AWS side - this is a logical gateway which would be the target of one or more route tables.
Customer Side: 
          1. A Customer Network (On Prem)
          2. A Customer Gateway on the Customer side, which can be the logical component within AWS and the physical router on the Customer side that this logical piece connects to. . 
Common to both sides: The tunnel through which data flows. 

Steps : 
1. Deploy the AWS VPC and a simulated On Prem Customer network using one click CF deployment. Afterb this is setup, there is no connection between the AWS VPC and the On Prem environment which we will verify prior to setting up the VPN connection. [Details](#Step1)
2. Create the Customer Gateway which is the logical connection that represent the physical Customer Router. These connections tell AWS how to connect to the routers using public IP addressing. Since there are 2 Physical Routers on the Customer side, there needs to be 2 Customer Gateways. 
3. Verify there is no connection between the VPC and Customer network.
4. Create the VPN attachments which provide 2 endpoints. There is already 1 Transit Gateway attachment that connects the TGW to the AWS VPC. For the 2 new ones we will select the accelerated VPN  option. This creates 2 accelerated endpoints per connection. These endpoints provide transit back to the AWS network via the AWS global network. The effect of this is that we will have two Site to Site VPN connections which can be used to connect the VPN endpoints to the On Premises router using IPsec tunnels, 2 per connection.
5. Download the configuration parameters for the Site To Site VPN connections. Each of these correspond to one Customer Gateway. This is needed to set up the Strongswan VPN appliance. Extract IP address, shared secret and other relevant information from the configuration files. This is needed to configure the VPN tunnels and to configure BGP. Tunnels are created using the same pre shared key. This key is one of the configuration parameters in the downloaded file. Inside the tunnels - here the internal IP addresses are used. This is where BGP traffic runs and this is the tunnel through which data is transmitted.
To emphasise: Outside tunnel refers to the encrypted data stream between both parties. Inside Tunnel refers to the routing and the raw data that is transmitted. 
Each Customer Gateway connects to two endpoints on the AWS side for high availability which has one tunnel each. Extract values from the configuration file. The inside IP, outside IP, shared secret etc.
6. Configure Strongswan using these parameters. Make sure both VPN connections are available.
The resources (ONPREM-ROUTER1 and ONPREM-ROUTER2) that were deployed using the one click deployment script already came with scripts used to configure Strongswan. In the real world we would have to install a VPN solution ourselves.
Modify the files and copy them to the /etc directory where they will be read by the strongswan software. Restart strongswan.
After the setup we should be able to see two virtual tunnel interfaces. This means that the tunnel interfaces are active and are connected to aws. Do the same for ONPREM-ROUTER2. After this is complete we should see that IPSEC is UP for both tunnels in Both Site-to-Site VPNS.
7. Configure BGP to run on top of the IPsec tunnels and establish connectivity between the two parties. This is done using FRR, Free Range Routing which provides a solution for routing protocol implementations, in our case BGP. This is how we are configuring dynamic routing between the networks. FRR exchanges routing information with other routers in the network, enabling efficicient and dynamic route propagation. 
8. Once the BGP configuration is done and we can ping each other from both sides. 

Now that the high level steps and the purpose of each step are clear, let's implement it.

#Step1: 
When the one click deployment is complete we have the VPC created, with CIDR 10.16.0.0/16
The On Prem CIDR is 192.168.8.0/21	

#Step 2:
Create Customer Gateway Objects 
![Alt text](../03-AdvancedSiteToSiteVPN/images/CustomerGatewayAdded.png) 
#Step 3:
Verify connectivity. There is no connectivity. 
![Alt text](../03-AdvancedSiteToSiteVPN/images/PingBeforeConnect.png) 
#Step 4:
Create VPN attachments 
![Alt text](../03-AdvancedSiteToSiteVPN/images/TransitGatewayAttachment.png) 
#Step 5: 
Download connection parameters from the VPn Site To Site Connection for the appropriate attachment. 
![Alt text](../03-AdvancedSiteToSiteVPN/images/DownloadConfiguration.png) 
We have completed the AWS side. We've configured the endpoints that the Customer can connect to. We've downloaded the configuration information for those endpoints. 
The on Prem has 2 layers  - the IP sec and the BGP layer.
We are now going to configure StrongSwan
The on prem router that came with the CF template in this exercise came preinstalled with Strongswan software. 
In real world we will need to install the IPN solution ourselves and configure it based on the downloaded file. There are different Vendors we can choose from. 

#Step 6: We configure it in the following step by modifying the files. 
ipsec.conf contains the actual configuration for the ipsec tunnels
ipsec.secrets has the authentication information that this linux server will use to authenticate against aws
the ipsec-vti.sh script file - this will enable and disable the ipsec tunnel whenever the system detects traffic.
Now we edit these config files with parameters from the downloaded config files. This needs to be done carefully where we replace the AWS inside ip address, AWS outside IP address, shared secret for the phase 1 IPSEC tunnel etc.
THen the Phase 1 tunnel gets setup and the ipsec-vti.sh script will enable and disable the ipsec tunnel whenever the system detects traffic.
![Alt text](../03-AdvancedSiteToSiteVPN/images/restartStrongSwan.png) 
restart strongswan will bring both the tunnels from the Customer side to the AWS side.
After everything has been configured and after Strongswan has been rebooted, two vti interfaces whould show up on ifconfig.
![Alt text](../03-AdvancedSiteToSiteVPN/images/StrongSwanConfigComplete.png) 
Now if we check Site to Site VPN we should see that their IPSEc IS UP. This means these tunnels are active and that connectivity from Router1 and Router2 to aws is working and we can use these tunnels to establish BGP sessions and route traffic through them. 

![Alt text](../03-AdvancedSiteToSiteVPN/images/ipSecIsUp.png) 
#Step 7: Now we will configure BGP to run on top of these tunnels. 
Step 1  install FRR usng a script - which is in Router1 and Router 2. Then we will configure BGP which will share routing information to aws servers. This needs to be done on both Routers. SSH into the router, then install FRR.
FRR 
The router that came with the one click deployment came already setup with the script. 
Again in real world scenario, we would need to do install this ourselves. 
![Alt text](../03-AdvancedSiteToSiteVPN/images/installffrouting.png) 
After the script is run, we see that this is complete. 
![Alt text](../03-AdvancedSiteToSiteVPN/images/installFrrComplete.png) 
#Step 8: 
Now we proceed to configure BGP. 
![Alt text](../03-AdvancedSiteToSiteVPN/images/configureBGP.png) 
Once the Router is back it will be functioning as both an IPSEC endpoint and a BGP endpoint. 
It will now start exchanging routes with the Transit Gateway that we have setup. 
Now when we do ```show ip route``` we see that it now has the ip address of our VPC. 
![Alt text](../03-AdvancedSiteToSiteVPN/images/bgpConfigComplete.png) 

Now to make sure pings are working from both sides: Log on to VPC EC2 server and ping on prem server: 
![Alt text](../03-AdvancedSiteToSiteVPN/images/pingVPCFromOnPrem.png) 
Log on to On prem server and ping VPC EC2:
![Alt text](../03-AdvancedSiteToSiteVPN/images/pingOnPremFromVPC.png) 

The other server works as well. 
















