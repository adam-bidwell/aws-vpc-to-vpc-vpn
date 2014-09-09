# Amazon Web Services VPC to VPC VPN tunnel

This repository is some snippets of code to configure a tunnel between two VPC networks in AWS.

## Usage

The script in the src/vpc.json file is a good starting point. It will create a VPC, with a public and private subnet in it. It will then place a VPN gateway in the public subnet and a normal EC2 instance in the private subnet.

In the example below, we'll create two VPC networks with this script. We'll then connect the two using openswan, so that the two private instances can talk to each other as if their networks were local to each other.

## Example

We're going to build two VPC, one with CIDR 10.100.0.0/16 and the other with CIDR 10.200.0.0/16, then connect them together via a VPN.

1. Copy the file src/vpc.parameters.json to src/vpcA.parameters.json. Edit this file before building the first VPC.
  * CIDRBLOCK - The subnet range of the VPC. e.g. 10.100.0.0/16
  * PublicCIDR - The subnet range for the public subnet in the VPC. e.g. 10.100.1.0/24
  * PrivateCIDR - The subnet range for the private subnet in the VPC. e.g. 10.100.100.0/24
  * KeyName - The name of an existing EC2 KeyPair that you will use to access any instances that are created
  * VpcName - A name for the VPC. Nothing fancy required here. 'VPCA' would do.
  * RemoteCIDRBLOCK - The subnet range of the other VPC you will connect to. e.g. 10.200.0.0/16

2. Build the VPC with the cloudformation command line.
  * aws cloudformation create-stack --stack-name VPCA --template-body file://src/vpc.json --parameters file://src/vpcA.parameters.json

3. Copy the file src/vpc.parameters.json to src/vpcB.parameters.json. Edit this file, before building the second VPC.
  * CIDRBLOCK - e.g. 10.200.0.0/16
  * PublicCIDR - e.g. 10.200.1.0/24
  * PrivateCIDR - e.g. 10.200.100.0/24
  * KeyName - The name of an existing EC2 KeyPair that you will use to access any instances that are created
  * VpcName - e.g. 'VPCB'
  * RemoteCIDRBLOCK - The CIDR block you used for the first VPC. e.g. 10.100.0.0/16

4. Build the VPC with the cloudformation command line.
  * aws cloudformation create-stack --stack-name VPCB --template-body file://src/vpc.json --parameters file://src/vpcB.parameters.json

5. Make a note of the output parameters from both cloudformation stacks
  * VPN GATEWAY A IP = Public IP address for VPNGateway in VPCA
  * VPCA CIDRBLOCK = CIDRBLOCK parameter value for VPCA
  * VPN GATEWAY B IP = Public IP address for VPNGateway in VPCB
  * VPCB CIDRBLOCK = CIDRBLOCK parameter value for VPCB
    
  Also create a VPN KEY, for testing this can perhaps be something simple, like ABC123. In production this obviously would be something more robust.

6. On the first VPNGateway (the public instance in VPCA) install an ipsec tunnel.
```
    sudo yum -y install openswan
    
    sudo vi /etc/ipsec.conf
    
    (uncomment, or add the line `include /etc/ipsec.d/*.conf`)
    
    sudo vi /etc/ipsec.d/vpcA-to-vpcB.conf
    
    (enter the following..)
    
    conn vpcA-to-vpcB
        type=tunnel
        authby=secret
        left=%defaultroute
        leftid=<VPN GATEWAY A IP>
        leftnexthop=%defaultroute
        leftsubnet=<VPCA CIDRBLOCK>
        right=<VPN GATEWAY B IP>
        rightsubnet=<VPCB CIDRBLOCK>
        pfs=yes
        auto=start

    sudo vi /etc/ipsec.d/vpcA-to-vpcB.secrets
    
    (enter the following..)
    
    <VPN GATEWAY A IP> <VPN GATEWAY B IP> : PSK "<VPN KEY>"
```

7. On the second VPNGateway (the public instance in VPCB) install an ipsec tunnel. This is almost identical to the previous step, just reversing mention of VPC A and VPC B values.
```
    sudo yum -y install openswan
    
    sudo vi /etc/ipsec.conf
    
    (uncomment, or add the line `include /etc/ipsec.d/*.conf`)
    
    sudo vi /etc/ipsec.d/vpcB-to-vpcA.conf
    
    (enter the following..)
    
    conn vpcB-to-vpcA
        type=tunnel
        authby=secret
        left=%defaultroute
        leftid=<VPN GATEWAY B IP>
        leftnexthop=%defaultroute
        leftsubnet=<VPCB CIDRBLOCK>
        right=<VPN GATEWAY A IP>
        rightsubnet=<VPCA CIDRBLOCK>
        pfs=yes
        auto=start

    sudo vi /etc/ipsec.d/vpcB-to-vpcA.secrets
    
    (enter the following..)
    
    <VPN GATEWAY B IP> <VPN GATEWAY A IP> : PSK "<VPN KEY>"
```
8. Start the service on both VPNGateways.
```
    sudo service ipsec start
    
    sudo chkconfig ipsec on
```
9. Configure packet forwarding on both VPNGateways.
```
    sudo vi /etc/sysctl.conf
    
    (find the line about `net.ipv4.ip_forward`, set it to `net.ipv4.ip_forward = 1`)
    
    sudo service network restart
```
10. On each VPNGateway, you should now be able to ping the private IP address of the other VPNGateway.

11. You should now also be able to SSH to the private instance, from the public instance. From the private instance you should be able to ping the IP address of the VPNGateway and the private instance in the other VPC.

