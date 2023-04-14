# AWS_Host_Static_Website_in_S3_bucket

### Pre-requisite:

Dmain Name (can be purchased from AWS)


## I. Create Static Website

1) Log on to aws console and ceate new S3 bucket:

   name: jjkkexpress.online (it should be smililar to your Dmain Nsme)

   select acls enable

   region: US East 2

   uncheck block public access


2) Upload website codes (all folders & files) to the newly created bucket (download attached "static-website-example-master.zip" file to your desktop)

   allow public access
   
3) Go to bucket properties 

   enable static web site hosting
   
   index page - index.html
   
   erroe page - error.html

4) Go to permission and add below bucket policy

   aws s3 static site policy :
   
```ruby
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


9) Wait for few minutes to get DNS populated

10) Test site is accessible from outside by accessing URL http://jjkkexpress.online


Note: Delete all resources created on Azure as soon as test complete to avoid charges
