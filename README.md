# 1. AWS_Host_Static_Website_in_S3_bucket

### Pre-requisite:

Dmain Name (can be purchased from AWS)


## I. Create Static Website

1) Log on to aws console and ceate new S3 bucket:

   name: jjkkexpress.online (it should be smililar to your Dmain Name)

   select acls enable

   region: US East 2

   uncheck block public access


2) Upload website codes (all folders & files) to the newly created bucket (download attached "static-website-example-master.zip" file to your desktop)

   allow public access
   
3) Go to bucket properties 

   enable static web site hosting
   
   index page - index.html
   
   error page - error.html

4) Go to permission and add below bucket policy

   aws s3 static site policy :
   
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::jjkkexpress.online/*"
        }
    ]
}
```

5) Goto properties again, copy website URL and browse it

http://jjkkexpress.online.s3-website.us-east-2.amazonaws.com


## II. Create DNS zone and records

6) Create hosted zone similar to S3 bucket

7) Create A record as jjkkexpress.online

   destination : S3 bucket URL (http://jjkkexpress.online.s3-website.us-east-2.amazonaws.com)

8) Copy name servers records from newly created hosted zone and update it in the system from where purchased the Domain Name (ignore this step if DN purchsed from AWS)

   ns-687.awsdns-21.net

   ns-329.awsdns-41.com

   ns-1399.awsdns-46.org
 
   ns-1862.awsdns-40.co.uk


10) Wait for few minutes to get DNS populated

11) Test site is accessible from outside by accessing URL http://jjkkexpress.online


Note: Delete all resources created on Azure as soon as test complete to avoid charges


# 2. Hosting Internal HTTPS Static Websites with ALB, S3 and PrivateLink

This solution leverages your existing private connection to the VPC and an Internal ALB to present the TLS certificate of the custom S3 bucket domain to the end-user. The ALB leverages AWS Certificate Manager (ACM) to present a valid certificate for the end-user, while maintaining a secure TLS connection to the trusted Amazon S3 VPC Endpoint. This enables the use of custom domain names for your static website.

### Prerequisites:

1. An AWS account
   
2. A private or imported certificate for the domain imported in ACM in the Region of your choice

3. An Amazon S3 bucket named with the domain that matches your ACM certificate (such as “portal.example.com”). You can read here to get started with Amazon S3, and read about working with Amazon S3 buckets here.
   *You **do not need** to enable static website hosting on the bucket, as this is only for public website endpoints. Requests to the bucket will be going through a private REST API instead.*

4. A VPC with at least two private subnets

5. An existing Direct Connect, Site-to-Site VPN, or Client VPN connection with correct routing into your VPC. This will be referred to as your private inbound connection.

6. An private hosted zone for your custom domain name

7. A resource that can access your VPC network, such as an Amazon Elastic Compute Cloud (Amazon EC2) instance running inside of the VPC

8. A static website containing an index.html entry page populated into your S3 bucket

### Step 1: Create your Amazon S3 VPC Endpoint

1. Log in to your VPC Dashboard.

2. On the left-hand menu, navigate to the “Endpoints” page.

3. Select “Create Endpoint”.

4. Search for “s3” in the Service List, and select the Amazon S3 Interface Endpoint service.Screenshot of the VPC Create Endpoints page, with S3 Interface endpoint selected

5. Select the VPC which contains your private inbound connection, and at least two subnets to which the endpoint will belong. It’s recommended that you select subnets belonging to two or more different Availability Zones (AZs) to leverage fault isolation on your interface endpoint.

6. Select the security group that you would like to protect the VPC Endpoint. This security group must allow access on ports 80 and 443 from the security group for your ALB at minimum.
   
7. Select “Full Access” for the VPC Endpoint policy. This policy makes sure that all AWS principals working in the VPC can access the VPC Endpoint for any S3 bucket. This doesn’t bypass the security policy we’ll define on the S3 bucket. However, you can choose to limit this policy to only allow access to the ALB later after you’ve created it.

8. Select “Create endpoint”.

9. Select your new VPC Endpoint ID to navigate to the new VPC Endpoint.

10. On the bottom tabs, go to “Subnets”.

*11. Note the IPv4 Addresses of your VPC Endpoint, as you’ll need them later!Screenshot of the IPv4 Addresses of the VPC Endpoint*

### Step 2: Allow the VPC Endpoint to your S3 bucket

1. Navigate to your S3 bucket, and go to the “Permissions” tab.

2. Scroll down to “Bucket policy”, and Select “Edit”.

3. Add your policy based on the provided documentation. For convenience, you can also use this provided policy to make sure that only your VPC Endpoint is explicitly allowed:
   
```
{
   "Version": "2012-10-17",
   "Id": "Policy1415115909152",
   "Statement": [
     {
       "Sid": "Access-to-specific-VPCE-only",
       "Principal": "*",
       "Action": "s3:GetObject",
       "Effect": "Allow",
       "Resource": ["arn:aws:s3:::yourbucketname",
                    "arn:aws:s3:::yourbucketname/*"],
       "Condition": {
         "StringEquals": {
           "aws:SourceVpce": "vpce-1a2b3c4d"
         }
       }
     }
   ]
}

```
### Step 3: Set up your Internal ALB

1. Navigate to your AWS Console EC2 Dashboard.
   
2. On the left-hand menu, select “Load Balancers”.
   
3. Select “Create load balancer”.

4. Select “Create” under the “Application Load Balancer” box.

5. Name your ALB and pick the “internal” scheme.

6. Switch the listener protocol to “HTTPS”.

7. Select your private subnets that the ALB will serve. Select “Next”.

8. Select the ACM certificate that will be served to your clients. Note that it must match your static S3 bucket domain/name. Select “Next”.

9. Select or create a security group that will allow your existing private connection to connect to port 443 on the load balancer. Select “Next”.
   *If you haven’t done so already, then make sure that you update the security group for your VPC Endpoint to allow access to the ALB security group that you’ve defined here.*

10. Create a new target group that will target IPs using the HTTPS protocol. Use HTTP protocol under the health checks. Under “Advanced Health Check settings”, ensure that the Port Override is set to 80 to match the HTTP protocol.Screenshot of the Target Group page with options entered

11. ALB health checks Host headers will not contain a domain name, so S3 will return a non-200 HTTP response code. Add “307,405” to the health check success codes. Select “Next”.

12. Register the VPC Endpoint IP addresses that you noted from Step 1 into the target group. Select “Next”.Screenshot of the VPC Endpoint IP addresses populated into the ALB target group page

### Step 4: Configure extra listener rules

The Amazon S3 PrivateLink Endpoint is a **REST API Endpoint**, which means that trailing slash requests will return XML directory listings by default. To work around this, you’ll create a redirect rule to point all requests ending in a trailing slash to index.html.

1. Navigate to your Internal ALB. Select it, and open the “Listeners” tab.
   
2. On the right side of the table for the HTTPS listener, select “View/edit rules”.

3. Select the “+” icon near the top to enable the insertion of a new rule.

4. Under “IF”, select “Add Condition”, then select “Path…”.

5. Enter “*/” under the Path Value.

6. Under “THEN”, select “Add Action”, then select “Redirect to…”.

7. Enter “#{port}” under “Enter port”.

8. Pick “Custom host, path, query” under the dropdown.

9. Modify “Path” to “/#{path}index.html”.

10. Select “Save” in the top-right corner.Screenshot of the ALB Listener rules with options populated as described above

### Step 5: Configure your DNS and test your ALB
Configure your on-premises or private DNS entries to point to the Internal ALB. You can use Route53 private hosted zones (PHZs) to set up a private alias record, and associate the PHZ with your VPC. You can also forward inbound DNS queries from on-premises to your VPC.

### Step 6: Test your ALB
You must use a resource with private access to your VPC to reach the Internal ALB. Connect to your resource and try navigating to your new private DNS entry. If you only have console access to your resource, then you can also use a cURL command to validate the private static website.

### Cleanup
To clean up, you can delete or revert the resources created in this guide in the following order:

Route53 PHZ DNS entries, ALB, Load Balancer Target Group, The Amazon S3 VPC Endpoint, Any related Security Groups that you created, Your S3 bucket policy


