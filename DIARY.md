# Taking on the SteamCo DevOps Challenge

### By Tincho


## First Impressions of the task

- Glancing over all the tasks, they all made sense and understoof them, however the gritty details of how to complete them weren’t all that clear. 

- Prior experience with CloudFormation: 0

- in order to try and make the most effective use of learning, I thought YouTube would be the best spot to find some AWS resource about CF. Came across [this](https://www.youtube.com/watch?v=6R44BADNJA8) gem which only lasted 50 minutes, the same investment I would make for an episode of The Flash.

- _".. that return the instance id.."_  
Returning the Instance ID seem's pretty straight forward, though it will probably involve going through some docs to see if there is some CloudFormation function to get to it.

- _"Redirect any HTTP requests to HTTPS. Self-signed certificates are acceptable."_ 
It had been a while since I played around with self-signed certs, though I know it'll involve some calls to openssl with certian args and I'll be able to generate keys to then upload onto my instances

- _"Drive the deployment with Puppet."_
I know of Puppet, I know what it's used for and that it's a Ruby DSL for automating the provisioning of machines. I suspect that for provisioning whatever app I deploy should be pretty straight forward.

- _"Provide basic automated tests to cover included scripts, templates, manifests, recipes, code etc."_
I felt this would be my weakest point for the test, only that it would require a bit of research into what appropriate tools I can use in order to test everything. Puppet surely comes with their owns tests, however CloudFormation might be tricky.


## Getting Started

The AWS presenter in some video stated “The best CloudFormation config are one’s that are copied from someone”, which is a point of view I subscribe to aswell. So I looked for a config that seemed to have the the majority of what I needed, with the intention of then tweaking it.

I ended up coming across one which seemed to tick most boxes: [Auto Scaling Multi AZ with Notifications CF Template](https://s3.amazonaws.com/cloudformation-templates-us-east-1/AutoScalingMultiAZWithNotifications.template.)

I created a new KeyPair and ran the template inside an existing VPC I had. 

### First hiccup:

- Couldn't access the ELB through it’s domain name. 
- Traffic isn’t being forwarded onto the WebServers, however I was able to ssh into them
- Realised that HTTP connections weren't being allowed through the AWS 'firewall'
- I updated my template with a new security group thanks to  [markitx](https://github.com/markitx/cloud-formation-templates/blob/master/load-balancers.template#L102), and re-ran it all to ensure it worked fine. And it did.

![Able to hit our public domain name and get back the instance ID](img/instance_id_returned.png)
<p style="text-align: center;font-style: italic;">Success! Challenge completed. Do I get a job giving Pranav crap now? ;)</p>

### Self critisism over VPC

* I looked at the funky VPC that my StreamCo app running in, and it was actually not the best possible setup as it was something hacked together for a previous project.
- Furthermore, I thought it beneficial for future people (including myself), be able to recreate this environment properly including all aspects of it. Our goal here is to achieve some form of reproduble build for our infrastructure
* While looking for a template to start off with, came across this nice image, which was a perfect representation of what I wanted to achieve: ![](https://docs.aws.amazon.com/quickstart/latest/vpc/images/quickstart-vpc-design-fullscreen.png)
* Except 4 is a bit of overkill, we’ll just configure 2 of them
* Ran the aws-vpc.template formation with two AZ’s, in us-west-2a and us-west-2b, and with private subnets also being created (because we’ll use them later on for the extra credit tasks)


### Lets Encrypt Everything

- Found a nice guide for getting self-signed certs for Load balancers [here](https://medium.com/@chamilad/adding-a-self-signed-ssl-certificate-to-aws-acm-88a123a04301)
- Interestingly unable to use AWS::CertificateManager::Certificate as a parameter to CloudFormation. According to some documentation it's only available for public Certs that have been verified by email. So what I do is pass in the CRN as a string in CloudFormation parameter.
- We'll do the redirecting in Apache itself, allowing traffic to come from ELB straight through to our webservers on both 80 and 443

### Issues encountered

- Wasn't able to get my self signed certs into ACM, because I hadn't put in a FQDN. To check if they are properly uploaded, use the AWS cli `aws acm list-certificates`. Although they may appear in the web console, it doesn't count unless they appear in the cli output.


## How would I further automate the management of the infrastructure?

### Remove the use of the AWS web console

The web console, though nice to enter parameters and view wizards, maintains a human aspect to dealing with the infrastructure which we are aiming to remove. The first step I would do is make use of the aws cli, for e.g. creating a CloudFormation stack or applying changes to CF templates, and place these repetative tasks in some bash scripts. 

My desired goal from doing this with as many tasks as possible, is that I start to think of new use cases where automation insdie the infrastructure leads to clean solution. Some examples to illustrate my point:

- running weekly jobs which tear down test and dev environments, and create replicas of production
- creating staging environment through chat bots
- Have pre-planned DR scenarious which can be run by support remotely
- Have scripts for creating appropraite keys for new hires


### Monitoring and Notifications

In managing our infrastructure, I would want to focus a fair amoung of energy on setting up appropriate notifications for our support engineers to respond and action with. I'd draw a lot of inspiration from a paper by a Google SRE team memeber, ["My Philosophy on Alerting"](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/edit).

Monitoring would also include application monitoring, in our case monitoring that apache is serving requests and isn't being resource hungry. A nice side effect of this monitoring of resource usage, is to also drive down costs from instances that aren't being fully used. Perhaps for some services we could downgrade instance size and maintain performance? 



## How I would take on the extra tasks


### Drive deployment with Puppet

- Create a local vagrant image using the same image we are running on our AWS instances to ensure dev/prod environment parity
- Write and test my puppet manifests for this situation. Provision it with a new user for 'web'. Install httpd and configure it with a virtual host which has an index.html with the instance id (see cloudformation template for reference)
- Remove the existing `commands` sections in our cloudformation template and add a new command in CloudFormation which would be a shell provisioner for installing ruby and puppet
- Upload the puppet manifests to S3 and download them onto the new instance
- `puppet apply`

Improvements with more time/money
- Run a puppet master in one of our private subnets and run the deploys from there
- Keep a health check with the puppet master and agent to ensure that the application (httpd) is running fine too

### Provide basic automated tests to cover included scripts, templates, manifests, recipes, code etc

- Provide [rspec unit](http://rspec-puppet.com/) tests for the puppet manifest which would ensure that traffic is open on :80 and :443, and that HTTP is redirected to HTTPS
- For AWS, write some integration tests that would do a simple http request to our instance and see what is returned. Then compare that value to an Instance ID by querying the AWS api. These could be run on our puppet master.
- We could also put in our CloudFormation template to wait for the provisioning to run and to run the unit tests as a smoke test

Improvement with more time/money
- Run our CloudFormation scripts (both StreamCoDevOps && aws-vpc) to create a duplicate testing environment
- In that environment, we can provision our servers and test that they all performed correctly
- I would probably also add further tests to destroy some of the instances using the AWS api, to ensure that new instances deployed in the auto-scaled group are provisioned properly by our puppet master

### Redirect any 404 errors to a custom static page

- Add an [ErrorDocument](https://httpd.apache.org/docs/2.4/custom-error.html) in the httpd config for serving a customer error page at /404.html
- Modify our puppet scripts to create this file in our document root

Improvement with more time/money
- Set up CloudFront to sit infront of our Load Balancer and [add a rule](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/custom-error-pages.html#custom-error-pages-cache-behavior) for handling any 404 errors that occur in our web instances, to be served by CloudFront. 

### Add a Database to your automation and have your application serve the data stored in addition to the instance ID.

- Add to our puppets scripts to also install postgres on the web server instances 
- Write a 'db migration' script which will create our database, create a table called instance, and insert a single row with the Instance ID of EC2 instance the app is hosted on
- Write a PHP file to handle a request comming in from '/', query the database and return it. (don't judge me too much for choosing PHP, but you have to admit it's the Right Tool for the Job™ in this situation)

Improvement with more time/money/show-off AWS skillz
- Modify our CloudFormation template to create an RDS instance, and DNS entry for something like db.streamco.com. We'lll use that in our connection string in our application
- Have our application run it's db migration against the RDS instance and query it for the application data. 
