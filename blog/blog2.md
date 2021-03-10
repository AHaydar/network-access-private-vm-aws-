# Building a secure network in AWS (PART 2)

Assume you've been asked to create a VM on AWS to run some critical operations for your business; it needs to access the internet, but only can be accessed by the maintainers (e.g. people/services who would want to install/upgrade the software). How would you do it?

This is a series of 2 posts. In the first post, we went over what happens when we create an EC2 instance (VM) in AWS, where we explained how the instance gets attached to the default VPC and the traffic gets routed.

In this second post, we will go over creating a full secure network where the newly created VM will operate.

_Why can't we just use the default vpc?_
The default VPC is available to get users started quickly without thinking about the underlying network. So it works great if you are doing some simple deployments, or If you are testing or experimenting in AWS.

The default VPC has multiple components associated with it, such as subnets, NACLs, Security Group and Route table. Some of these component are open publicly, which does not form the most solid security practice.

Moreover, using the default VPC will limit the user to using the `172.31.0.0/116` range. By having a custom VPC the customer will be able to tailor the network exactly and avoid overlapping IP addresses, including on-premise networks

It is recommended to configure a custom VPC, where the user has full control over the network model and its components. This makes it easier to scale the application and architecture within private subnets.

## Setup a Custom VPC

In this section we will start by creating the necessary resources progressively using cloud formation. At every step of this section, I suggest you upload the template to CloudFromation and have a look at the resources that get created with the stack. Use the following command at your convenience (or upload the template manually from the console):

To create the stack - `aws cloudformation create-stack --stack-name private-network --template-body file://infra/template-custom-vpc.yml`

To update the stack - `aws cloudformation update-stack --stack-name private-network --template-body file://infra/template-custom-vpc.yml`

### Create the VPC

Use the following template to create the custom VPC:

```
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 192.168.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: awesome-vpc
```

In this previous snippet, we have given a name for the VPC, a CidrBlock that defines the range of IPS that could be utilised (in this case from 192.168.0.0 to 192.168.255.255). When setting the EnableDnsSupport and EnableDnsHostnames flags to true, the instances with a public IP address get corresponding public DNS hostnames, and the Amazon Route 53 Resolver server can resolve Amazon-provided private DNS hostname. If you're curious to know more check the detailed AWS documentation: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html.

### Create the Internet Gateway

In the template file, under "Resources", add the following:

```
  InternetGateway:
    Type: 'AWS::EC2::InternetGateway'
    Properties:
      Tags:
      - Key: Name
        Value: awesome-internet-gateway
```

This would create the internet gateway, but it won't be attached to a VPC: ![Internet Gateway](A.Internet-Gateway.png)

Add the following resource to your template to attach the internet gateway to the created VPC:

```
  VpcInternetGatewayAttachment:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
```

### Subnets

Afterwards, we will create 2 subnets, one private and one public - the private one will contain our EC2 instance and the other can connect to the internet through the internet gateway.

```
  SubnetPrivate:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      CidrBlock: 192.168.0.0/17
      Tags:
        - Key: Name
          Value: subnet-private
  SubnetPublic:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      CidrBlock: 192.168.128.0/17
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: subnet-public
```

When creating a subnet we do not deicde whether it's private or public through a configuration on creation, so for this step these are the subnets that are intended to be private and public; we will configure them for that in future steps: ![Subnets](B.Subnets.png)Notice how we split the Cidr blocks of the VPC into 2 Cidr blocks (one for each subnet). The Availability zone property uses 2 CloudFormation intrinsic functions:

- GetAZs, which returns a list of all of the Availability zones for a specific region; specifying an empty string is equivalent to specifying the region in which the CloudFormation stack is created
- Select, which returns a single object from a list of objects by index

In this case, we are getting the first availability zone in the us-east-1 region, in which we're creating the stack.

The MapPublicIpOnLaunch property in the public subnet indicates whether instances launched in this subnet receive a public IPv4 address. This might be confusing as we discussed creating the EC2 instance in the private network, so why are we adding this configuration? Briefly, we will create another EC2 instance in the public subnet, which would be a utility to SSH into the private subnet instance later on. We will get back to more details about this instance in the coming steps.

### Route table

We will now create a route table and associate it with the subnet, in addition to a default route

```
  RouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
      - Key: Name
        Value: awesome-route-table
  RouteTableAssociationSubnetPublic:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref SubnetPublic
      RouteTableId:
        Ref: RouteTable
  RouteTableDefaultIPv4:
    Type: 'AWS::EC2::Route'
    DependsOn: VpcInternetGatewayAttachment
    Properties:
      RouteTableId:
        Ref: RouteTableWeb
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId:
        Ref: InternetGateway
```

Looking at the created route table, we will see it associated with the public subnet, and has the following routes: ![Route Table](C.Route-Table.png)

### NAT Gateway

The NAT(Network Address Translation) is the method of giving a private resource access to the internet. It always uses Elastic IPs (Static public IPs).

### Bastion host

In the subnets section we mentioned the creation of an EC2 instance in the public subnet. That would be the bastion host. It is a server whose purpose is to provide access to a private network from an external network (e.g. the internet); we will use it in our case to allow the maintainers SSH access to the private EC2 instance, which we will create in the next step.
