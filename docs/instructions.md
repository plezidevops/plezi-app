## My notes while working on this project. 

### Remarks

I used three security groups. One for the load balancer, application tier, and application tier. There are room for improvements such adding CloudFront, automate the entire environment using cloudFormation, terraform, ansible, etc. I choose to run all the application services on Ubuntu 20.04.

### Flow of Execution
1. Register a dmain with godaddy
2. Login to AWS
2. Create a public certificate using certificate manager
	- Domain: *.plezidevops.com
	- Use for the HTTPS connections on the load balancer
3. Add DNS record to godaddy that uses the AWS public certificate
	- Create a dmain
	- Create a dns record using the AWS CNAME name/value 
4. From AWS create key pairs
5. Create Security groups
	 - SG for the load-balancer
	 	- 80, 443
	 - SG for the Application tier
	 	- 8080
	 - SG for the DB Tier
	 	- MySql 3306
		- Memcache 1211
		- RabbitMQ	5672
6. Launch 3 ubuntu instances
	- Mysql DB instance
		- connect to the instance
		- connect to the database
			- mysqql -u root -p
		- Make sure account table created
			- show databases;
		- curl http://169.254.169.254/latest/user-data
		
	- Memcached Ubuntu server
		- Distributed memory object caching system that alleviates database 
		  load to speed up dynamic Web applications.
		- sudo systemctl status memcached
		- sudo ss -tunpl | grep 11211
		
	- Rabbit MQ ubuntu server
		- Login to the server
		- sudo ss -tunpl | grep 5672
		- curl http://169.254.169.254/latest/user-data
		
	- Create Hosted zone
		- From Route 53 create private hosted zone
			- Domain Name: plezidevops.priv
			- Region: US-East-1
			- VPC: Default VPC
			
		- Creating DNS records (A-Route traffic to an IPv4 address, Simple Route)
			- mc01 172.31.90.206
			- db01 172.31.81.101
			- rmq01 172.31.82.121
			
7. Build the application locally
	- I am using Ubuntu 20.04
	- sudo apt install openjsk-8-jdk -y
	- sudo apt install maven -y
	- make changes to the application.properties file
	- mvn install
	- the war file is located in the target folder

8. Create an iam account with programmatic access 
	- the iam needs s3 access
	- aws s3 mb s3://plezi-artifact-storage
	- aws s3 cp plezidevops-v2.war s3://plezi-artifact-storage/

9. Create an IAM role
	- This role would allow the tomcat server permission to access the war file on the s3 bucket
	- Attach the role to the tomcat EC2 instance
	- Connect to the App server and verify that you have access to the s3 bucket

10. Deployed the application
	- sudo systemctl stop tomcat9
	- /var/lib/tomcat9/webapps
	- sudo rm -rf ROOT
	- sudo aws s3 cp s3://plezi-artifact-storage/plezidevops-v2.war ROOT.war
	- sudo systemctl start tomcat9
	
11. Test and make sure the application are listening for request
	- telnet db01.plezidevops.priv 3306
	- telnet mc01.plezidevops.priv 11211
	- telnet rmq01.plezidevops.priv 5672

12. Setting up the application load balancer
	- Creating target group
		Target group name: plezi-app-TG
		Protocol: HTTP
		Port: 8080
		VPC: choose one
		Health check protocol: HTTP
		Health check path: /login
		Port Override: 8080
		Healthy threshold: 3
		Unhealthy threshold: 2
		Register targets: plezidevops-app01
		Include as pending below
		
	- Creating load balancer
		Load balancer name: plezi-app-lb
		VPC: choose one
		Mappings (AZ): select all
		Security groups: plezidevops-load-balancer-SG
		Default SSL certificate: *.plezidevops.com
		Listeners and routing:
			HTTP:80 using plezi-app-TG
			HTTPS:443 using plezi-app-TG
		record the DNS name
		
	- Create CNAME record on GoDdday
		Name: pleziapp
		Value: Load balancer DNS Name
		Fire up your browser: https://pleziapp.plezidevops.com
		
13. Adding autoscaling to the system
	- Create an image of plezidevops-app01 (application server)
	- create launch configuration
		Name: plezi-app-LC
		AMI: name-of-ami-just-created
		Instance type: t2.micro
		IAM instance profile: plezi-artifact-storage-role
		choose an an existing security groups: plezidevops-application-SG
		Choose an existing key pair: keyPairName
	- create autoscaling group
		Auto Scaling group name: plezi-app-ASG
		Launch configuration: Switch to launch template
		Launch configuration: plezi-app-LC
		Choose VPC:
		Availability Zones and subnets: choose all
	- Accessing the application
		https://pleziapp.plezidevops.com
		
References:
- Part 1: RabbitMQ for beginners - What is RabbitMQ?
	- https://www.cloudamqp.com/blog/part1-rabbitmq-for-beginners-what-is-rabbitmq.html
	