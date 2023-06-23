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
1. Deploy the AWS VPC and a simulated On Prem Customer network using one click CF deployment. There is no connection between the AWS VPC and this environment which we will verify prior to setting up the connection. 
2. Create the Customer Gateway which is the logical connection that represent the pysical Customer Router. These connections tell AWS how to connect to the routers using public IP addressing. Since there are 2 Physical Routers on the Customer side, there needs to be 2 Customer Gateways. 
3. Verify there is no connection between the VPC and Customer network.
4. Create the VPN attachments which provide 2 endpoints. There is already 1 Transit Gateway attachment that connects the TGW to the AWS VPC. For the 2 new ones we will select the accelerated VPN  option. This creates 2 accelerated endpoints per connection. These endpoints provide transit back to the AWS network via the AWS global network. 
5. The effect of this is that we will have two Site to Site VPN connections which can be used to connect the VPN endpoints to the On Premises router using IPsec tunnels, 2 per connection.
6. Download the configuration parameters for the Site To Site VPN connections. Each of these correspond to one Customer Gateway. This is needed to set up the Strongswan VPN appliance.
7. Extract IP address, shared secret and other relevant information from the configuration files. This is needed to configure the VPN tunnels and to configure BGP. Tunnels are created using the same pre shared key. This key is one of the configuration parameters in the downloaded file. 
9. Inside the tunnels - here the internal IP addresses are used. This is where BGP traffic runs and this is the tunnel through which data is transmitted.
To emphasise: Outside tunnel refers to the encrypted data stream between both parties. Inside Tunnel refers to the routing and the raw data that is transmitted. 
Each Customer Gateway connects to two endpoints on the AWS side for high availability which has one tunnel each. 
10. Extract values from the configuration file. The inside IP, outside IP, shared secret etc.
11. Configure Strongswan using these parameters. Make sure both VPN connections are available.
The resources (ONPREM-ROUTER1 and ONPREM-ROUTER2) that were deployed using the one click deployment script already came with scripts used to configure Strongswan. If not we would have had to install them ourselves.
Modify the files and copy them to the /etc directory where they will be read by the strongswan software. Restart strongswan.
After the setup we should be able to see two virtual tunnel interfaces. This means that the tunnel interfaces are active and are connected to aws.
Do the same for ONPREM-ROUTER2
After this is complete we should see that IPSEC is UP for both tunnels in Both Site-to-Site VPNS.
12. Configure BGP to run on top of the IPsec tunnels. and establish connectivity between the two parties.
13. Once done the VPN connection is done and we can ping each other from both sides. 

Now that the high level steps and the purpose of each step being clear, let's implement it. 














