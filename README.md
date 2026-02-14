# Assignment-2---IaC-Practices-with-AWS-CloudFormation - Lester John Abad

* I used our template from the class as starting point and modified the VPC to match the assignment requirements.
* I also created 2 public subnets on a different AZs because it is required in Application Load Balancer.
* I created a WebInstanceRole to allow access and copy file from s3 bucket.
  ```
  "WebInstanceRole": {
      "Type": "AWS::IAM::Role",
      "Properties": {
        "AssumeRolePolicyDocument": {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Principal": { "Service": "ec2.amazonaws.com" },
              "Action": "sts:AssumeRole"
            }
          ]
        },
        "Policies": [
          {
            "PolicyName": "AllowReadSingleS3Object",
            "PolicyDocument": {
              "Version": "2012-10-17",
              "Statement": [
                {
                  "Sid": "AllowGetSpecificObject",
                  "Effect": "Allow",
                  "Action": ["s3:GetObject"],
                  "Resource": "arn:aws:s3:::seis665-public/index.php"
                }
              ]
            }
          }
        ]
      }
    },

    "WebInstanceProfile": {
      "Type": "AWS::IAM::InstanceProfile",
      "Properties": {
        "Roles": [{ "Ref": "WebInstanceRole" }]
      }
    },

* I created web1 and web2 EC2 instances with AMI ID: ami-01cc34ab2709337aa
  * Web1 is in PublicSubnet1
  * Web2 is in PublicSubnet2
* I also added UserData information on both WebServer Instances using this code
  ```
  "UserData": {
        "Fn::Base64": {
          "Fn::Join": [
            "",
            [

              "#!/bin/bash\n",
              "yum update -y\n",
              "yum install -y git httpd php\n",
              "service httpd start\n",
              "chkconfig httpd on\n",
              "aws s3 cp s3://seis665-public/index.php /var/www/html/\n"
            ]
          ]
        }
      }
* I created security group with a logical name and group name of WebserversSG in the VPC.
Allow incoming requests on port 22 from your workstation (based on the YourIp
parameter) and Allow incoming requests on port 80 from 0.0.0.0/0
```
"WebserverSG": {
      "Type": "AWS::EC2::SecurityGroup",
      "Properties": {
        "VpcId": {
            "Ref": "EngineeringVpc"
        },
        "GroupDescription": "Security group rules for webserver host.",
        "SecurityGroupIngress": [
          {
            "IpProtocol": "tcp",
            "FromPort": "80",
            "ToPort": "80",
            "CidrIp": "0.0.0.0/0"
          },
		      {
            "IpProtocol": "tcp",
            "FromPort": "22",
            "ToPort": "22",
            "CidrIp": {"Ref": "YourIP"}		  
		      }
        ]
      }
    },
```
* An application load balancer named EngineeringLB and target group
named EngineeringWebservers
  * Load balance incoming requests on port 80 and send to instance port 80 using the
http protocol.
  * Load balancer health check via http on port 80 to the "/" url location.
  * Also with the Load Balancing Listener
  ```
  "EngineeringLB": {
      "Type": "AWS::ElasticLoadBalancingV2::LoadBalancer",
      "Properties": {
        "Name": "EngineeringLB",
        "Scheme": "internet-facing",
        "Type": "application",
        "Subnets": [
          { "Ref": "PublicSubnet1" },
          { "Ref": "PublicSubnet2" }
        ],
        "SecurityGroups": [
          { "Ref": "WebserverSG" }
        ]
      }
    },
    "EngineeringWebservers": {
      "Type": "AWS::ElasticLoadBalancingV2::TargetGroup",
      "Properties": {
        "Name": "EngineeringWebservers",
        "TargetType": "instance",
        "Port": 80,
        "Protocol": "HTTP",
        "VpcId": { "Ref": "EngineeringVpc" },
        "HealthCheckProtocol": "HTTP",
        "HealthCheckPort": "80",
        "HealthCheckPath": "/",
        "Matcher": {
          "HttpCode": "200"
        },
        "Targets": [
                {
                    "Id": {
                        "Fn::GetAtt": [
                            "Web1",
                            "InstanceId"
                        ]
                    },
                    "Port": 80
                },
                {
                    "Id": {
                        "Fn::GetAtt": [
                            "Web2",
                            "InstanceId"
                        ]
                    },
                    "Port": 80
                }
            ]
      }
    },
    "EngineeringListener": {
      "Type": "AWS::ElasticLoadBalancingV2::Listener",
      "Properties": {
        "LoadBalancerArn": { "Ref": "EngineeringLB" },
        "Port": 80,
        "Protocol": "HTTP",
        "DefaultActions": [
          {
            "Type": "forward",
            "TargetGroupArn": { "Ref": "EngineeringWebservers" }
          }
        ]
      }
    }
    
  },



