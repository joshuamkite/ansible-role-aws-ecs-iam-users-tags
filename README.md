aws-ecs-iam-users-tags
=========

Provides IAM based individual ssh acccess. This role is intended for deployment with Packer to an AWS ECS base host AMI. It draws heavily on [widdix/aws-ec2-ssh](https://github.com/widdix/aws-ec2-ssh) -see files/widdix_license- and much of the readme below is taken from there.

* On reboot, all IAM users are imported and local UNIX users are created
* The import also runs every 10 minutes (via cron - calls import_users.sh)
* You can control which IAM users get a local UNIX user and are therefore able to login
	* all (default)
	* only those in specific IAM groups
* You can control which IAM users are given sudo access
	* none (default)
	* all
	* only those in a specific IAM group
* You can specify the local UNIX groups for the local UNIX users
* You can assume a role before contacting AWS IAM to get users and keys (e.g. if your IAM users are in another AWS account)
* On every SSH login, the EC2 instance tries to fetch the public key(s) from IAM using sshd's AuthorizedKeysCommand
* As soon as the public SSH key is deleted from the IAM user a login is no longer possible

## IAM user names and Linux user names

Allowed characters for IAM user names are:
> alphanumeric, including the following common characters: plus (+), equal (=), comma (,), period (.), at (@), underscore (_), and hyphen (-).

Allowed characters for Linux user names are (POSIX ("Portable Operating System Interface for Unix") standard (IEEE Standard 1003.1 2008)):
> alphanumeric, including the following common characters: period (.), underscore (_), and hyphen (-).

Therefore, characters that are allowed in IAM user names but not in Linux user names:
> plus (+), equal (=), comma (,), at (@).

This solution will use the following mapping for those special characters when creating users:
* `+` => `plus`
* `=` => `equal`
* `,` => `comma`
* `@` => `at`

So instead of `name@email.com` you will need to use `nameatemail.com` when login via SSH.

Linux user names may only be up to 32 characters long.


Requirements
------------

This role presumes access to and use in AWS.

You must have a policy such as the below Attach attached to the EC2 instances (by creating an IAM role and an Instance Profile)

			{
			  "Version": "2012-10-17",
			  "Statement": [{
			    "Effect": "Allow",
			    "Action": [
			      "iam:ListUsers",
			      "iam:GetGroup"
			    ],
			    "Resource": "*"
			  }, {
			    "Effect": "Allow",
			    "Action": [
			      "iam:GetSSHPublicKey",
			      "iam:ListSSHPublicKeys"
			    ],
			    "Resource": [
			      "arn:aws:iam::<YOUR_USERS_ACCOUNT_ID_HERE>:user/*"
			    ]
			  }, {
			      "Effect": "Allow",
			      "Action": "ec2:DescribeTags",
			      "Resource": "*"
			  }]
			}

## Using a multi account strategy with a central IAM user account

If you are using multiple AWS accounts you probably have one AWS account with all the IAM users (I will call it **users account**), and separate AWS accounts for your environments (I will call it **dev account**). Support for this is provided using the AssumeRole functionality in AWS.

### Setup users account

1. In the **users account**, create a new IAM role
2. Select Role Type **Role for Cross-Account Access** and select the option **Provide access between AWS accounts you own**
3. Put the **dev account** number in **Account ID** and leave **Require MFA** unchecked
4. Skip attaching a policy (we will do this soon)
5. Review the new role and create it
6. Select the newly created role
7. In the **Permissions** tab, expand **Inline Policies** and create a new inline policy
8. Select **Custom Policy**
9. Paste the content of the [`iam_ssh_policy.json`](./iam_ssh_policy.json) file and replace `<YOUR_USERS_ACCOUNT_ID_HERE>` with the AWS Account ID of the **users account**.

### Setup dev account

For your EC2 instances, you need a IAM role that allows the `sts:AssumeRole` action

1. In the **dev account**, create a new IAM role
2. Select ROle Type **AWS Service Roles** and select the option **Amazon EC2**
3. Skip attaching a policy (we will do this soon)
4. Review the new role and create it
5. Select the newly created role
6. In the **Permissions** tab, expand **Inline Policies** and create a new inline policy
7. Select **Custom Policy**
8. Paste the content of the [`iam_crossaccount_policy.json`](./iam_crossaccount_policy.json) file and replace `<YOUR_USERS_ACCOUNT_ID_HERE>` with the AWS Account ID of the **users account** and `<YOUR_USERS_ACCOUNT_ROLE_NAME_HERE>` with the IAM rol name that you created in the **users account**
9. Create/edit the file `/etc/aws-ec2-ssh.conf` and add this line: `ASSUMEROLE="IAM-ROLE-ARN` or run the install.sh script with the -a argument

## Limitations

* your EC2 instances need access to the AWS API either via an Internet Gateway + public IP or a Nat Gatetway / instance.
* it can take up to 10 minutes until a new IAM user can log in
* if you delete the IAM user / ssh public key and the user is already logged in, the SSH session will not be closed
* uid's and gid's across multiple servers might not line up correctly (due to when a server was booted, and what users existed at that time). Could affect NFS mounts or Amazon EFS.
* this solution will work for ~100 IAM users and ~100 EC2 instances. If your setup is much larger (e.g. 10 times more users or 10 times more EC2 instances) you may run into two issues:
  * IAM API limitations
  * Disk space issues
* **not all IAM user names are allowed in Linux user names** (e.g. if you use email addresses as IAM user names). See section [IAM user names and Linux user names](#iam-user-names-and-linux-user-names) for further details.

Role Variables
--------------

The following variables are available. They must be specified but they can be left blank (see example below) to accept the role defaults

    IAM_authorized_groups: 		# Comma separated list of IAM groups to import
    IAM_authorized_groups_tag:	# Key Tag of EC2 that contains a Comma separated list of IAM groups to import - IAM_AUTHORIZED_GROUPS_TAG will override IAM_AUTHORIZED_GROUPS, you can use only one of them 
    local_groups:				# Comma seperated list of UNIX groups to add the users in
    sudoers_groups: 			# Comma seperated list of IAM groups that should have sudo access
    sudoers_groups_tags:		# Key Tag of EC2 that contains a Comma separated list of IAM groups that should have sudo access - SUDOERS_GROUPS_TAG will override SUDOERS_GROUPS, you can use only one of them
    assumerole:					# IAM Role ARN for multi account. See below for more info

         

Dependencies
------------

Not much use outside of AWS

Example Playbook
----------------

		---
		- hosts: all
		become: yes

		roles:
				- role: aws-ecs-iam-users-tags
		vars:
			#leave empty for role defaults
			IAM_authorized_groups: bastion-users-for-dazn-platform
			IAM_authorized_groups_tag:
			local_groups:
			local_marker_group:
			sudoers_groups: bastion-users-for-dazn-platform
			sudoers_groups_tags:
			assumerole:

License
-------

MIT

Author Information
------------------

Joshua Kite June 2018