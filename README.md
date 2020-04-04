# jitsi-meet-cfn

An AWS CloudFormation Template for getting started with Jitsi Meet.

## Motivation

With a sudden need for everyone to work remotely, I've been testing out video-conferencing solutions
and recently came across [this article from AWS about setting up the open-source Jitsi Meet](https://aws.amazon.com/blogs/opensource/getting-started-with-jitsi-an-open-source-web-conferencing-solution/).

Their instructions show how to do this manually, but I wanted to automate most of it to make it easier to maintain.

## Limitations

* The CloudFormation script configures **a public server, which has no access control on it** (others can start
  meetings). You should consider if this suits your needs. There is a way to configure some access control
  in the jicofo module, [see the instructions here under **Secure Domain**](https://github.com/jitsi/jicofo#secure-domain).
* Running a server comes with its own costs and maintenance overheads. You should be prepared to update it
  and monitor its security correctly.

## Before you start

You will need:

* **A domain name you control**. This is needed so third-parties can connect securely with a valid certificate.
* **An AWS Account**. The template is deployed with CloudFormation, so you'll need access to an AWS account you can deploy it in.
* **AWS Console or CLI access**. I'll provide instructions for the CLI and some hints for the Console.

Also keep in mind this solution deploys its own server. You need to be comfortable running a server publicly
accessible on the Internet and ready for the associated cost (I've recommended a t3.large instance, which is
about $US60 / 30 days).

## Instructions

Instructions are below for generating a SSH key pair and deploying the template.

*NOTE: In the AWS CLI instructions, we have assumed you set up the aws-cli correctly with an access key pair for
your account and a default region. You can add the `--region` flag to any AWS CLI command to change the region
it applies to.*

### EC2 Key Pair

You will need an EC2 key-pair in th eregion you propose to deploy this to. You can skip this section if you
already have one.

1. List the keys you already have:

  *Console*: go to the EC2 Console and select **Key Pairs** under **Network and Security** 

  *CLI*:

  ```bash
  aws ec2 describe-key-pairs
  ```

  Note the name the key you wish to use, as you will need it later.

2. If you don't want to use one of those, you can generate a new key pair, selecting a name
for it (which you will need later):

  *Console*: Select the 'Create key pair' button. This will generate the key pair and automatically download
  the private key. Save it to your `$HOME/.ssh` directory.

  *CLI*:
  ```bash
  aws ec2 generate-key-pair --key-name <key_name> --query KeyMaterial --output text > $HOME/.ssh/<key_name>
  ```

  (replacing `<key_name>` with a name for your key pair)

3. (Linux / Mac): Update the permissions on the key pair for SSH

  ```bash
  chmod 0600 $HOME/.ssh/<key_name>
  ```

### Installing the template

Installing the template is relatively straightforword from the Console or CLI. This will deploy a CloudFormation stack
containing an EC2 instance and an Elastic IP address, with a security group setup to permit public internet access for
Jitsi Meet, but restricted access to SSH from the IP address you specify.

*Firstly, clone this repository onto your local machine*

*AWS Console*

1. Navigate the AWS CloudFormation console (assuming you're using the new console layout, which should be the default)
2. Select **Create Stack**
3. Select **Template is Ready** and **Upload Template File**. Download the [jitsi.yml](jitsi.yml) file from this repository
  and upload it.
4. Click **Next**
5. Enter a **Stack Name** and fill out the parameters as follows:
  * `DNSName` - this the domain name you will host your server at
  * `InstanceTypeParameter` - the size of the instance. t3.large is good enough for small setups (say 10 simultaneous users)
  * `KeyName` - this is the name of the key pair you generated / selected in the previous stage
  * `SSHLocation` - this an IPv4 CIDR specifying where you can SSH the host from. For security reasons, 
    you should restrict this to just your IP address (e.g. 1.2.3.4/32). You can update it later if it changes.
6. Click **Next**, and continue with the default values on the following screens until you return back to the stack listing.

**AWS CLI**

```bash
aws cloudformation deploy \
  --stack-name <stack_name> \
  --template-file ./jitsi.yml  \
  --parameter-overrides SSHLocation=<ssh_location> KeyName=<key_name> DNSName=<dns_name> InstanceTypeParameter=t3.large
```

substituting:
  * `<stack_name>` - the name of the CloudFormation stack to create (e.g. `jitsi`)
  * `<dns_name>` - this the domain name you will host your server at
  * `<key_name>` - this is the name of the key pair you generated / selected in the previous stage
  * `<ssh_location>` - this an IPv4 CIDR specifying where you can SSH the host from. For security reasons, 
    you should restrict this to just your IP address (e.g. 1.2.3.4/32). You can update it later if it changes.

### Obtain the EC2 Host Name and IP Address

Once the template is deployed, you should be able to get the output values.

`EC2HostName` is the host name assigned by EC2, and `IPAddress` is its IP Address. The template creates
an Elastic IP resources, which gives your server a fixed IP.

*AWS Console*: click on the stack name in the **Stacks** view and switch to the Outputs tab. 

*AWS CLI*: `aws cloudformation describe-stacks --stack-name jitsi-test --query Stacks[0].Outputs`

### DNS Setup

Create an 'A' record for the hostname you specified as the `DNSHostName` parameter to the template, pointing
to the IP Address recorded in the previous step.

Once this is set up, you should be able to access your host via `https://<hostname>`, but the certificate will
be wrong. We will set up a Let's Encrypt certificate in the following steps.

### SSH to Host

You should now be able to SSH to the host, which you will need to be able to do to install the certificates.

```bash
ssh -i ~/.ssh/<key_name> ubuntu@<hostname>
```

If you're having trouble, check that the key file name is correct and has the right permissions. You can also try and run `ssh` with the
`-v` parameter for more information about the connection negotiation.

### Install Let's Encrypt certificate

Once you have SSH'ed to the host, run the following command:

```bash
sudo /usr/share/jitsi-meet/scripts/install-letsencrypt-cert.sh
```

The script will prompt you for an email address and ask you to agree to the linked usage agreement. Your
DNS must be set up correctly for this script to run successfully.

Once this process has complete, you should now be able to navigate to your hostname in your
web browser and start a meeting.

