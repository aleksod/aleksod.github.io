---
layout: post
title: Manage Your Infrastructure Using Chef (A Google Cloud Platform Example)
comments: true
---

<img align="center" src="https://i.imgur.com/Cev4BQR.png">

# The Problem

Imagine needing some infrastructure to support your Machine Learning (and other) projects. Perhaps, a node or two of your infrastructure performs web scraping, while another trains a complex model, and yet another hosts that model deployed. Those nodes will probably require some potentially non-trivial setup going beyond default configurations defined by out-of-box OS installations. You could do all that work manually, logging into those remote machines and performing all necessary actions. However, what if you had not one or two machines to configure, but 5 or 10 or even more? Now, this problem can even be further amplified if your setup periodically changes and needs to be updated on all of the machines. What solution can we deploy to help us with such task? 

# Solution

Luckily, amazing solutions are already available. As the title of this blog post suggests, today I will be talking about Chef, a configuration management tool written in Ruby and Erlang. According to [Wikipedia](https://en.wikipedia.org/wiki/Chef_%28software%29),
> It uses a pure-Ruby, domain-specific language (DSL) for writing system configuration "recipes". Chef is used to streamline the task of configuring and maintaining a company's servers, and can integrate with cloud-based platforms such as Internap, Amazon EC2, Google Cloud Platform, Oracle Cloud, OpenStack, SoftLayer, Microsoft Azure and Rackspace to automatically provision and configure new machines. Chef contains solutions for both small and large scale systems, with features and pricing for the respective ranges.

# Implementation

As the quote above says, Chef can be used with a myriad of setups, but today I will be using Google Cloud Platform (GCP) as a quick and easy way to spin up a tiny mock-up infrastructure that we are going to manage with Chef.

## GCP Setup

We will need several VM instances running:

1. A Chef server instance.
    > The Chef server acts as a hub for configuration data. It stores cookbooks, the policies that are applied to the systems in your infrastructure and metadata that describes each system. The knife command lets you communicate with the Chef server from your workstation. For example, you use it to upload your cookbooks. [1]
2. Several Chef nodes that we are trying to manage.
    > A node is any machine—physical, virtual, cloud, network device, etc.—that is under management by Chef. [2]
3. A Chef workstation.
    > One (or more) workstations are configured to allow users to author, test, and maintain cookbooks. Cookbooks are uploaded to the Chef server from the workstation. Some cookbooks are custom to the organization and others are based on community cookbooks available from the Chef Supermarket. [3]

In "Create an instance" dialogue, create instances with appropriate names, regions, and zones. Although, not a strict requirement, for best performance make sure all of them are in the same region and zone. Create nodes with appropriate machine type, depending on what your nodes need to do. The example I am going to go through here is very simple and does not require much computing power, so the nodes and the workstation will be of "micro" machine type (1 shared vCPU, 0.6 GB memory, f1-micro). In contrast, the server will need to be at least 1 vCPU machine type (3.75 GB memory, n1-standard-1), as I experienced frequent stalls with anything weaker than that. 

My example will also work exclusively with Ubuntu 16.04, but that is not a strict requirement, as Chef works on any operating system. However, Chef server only runs on Red Hat Enterprise Linux, SUSE Linux Enterprise Server, and on Ubuntu. Refer to the picture below for the setup I used for this demonstration.

![GCP Setup](https://i.imgur.com/uxDqJ4v.png "A GCP Setup for Chef Demo")

### Chef Server Setup

SSH into your Chef server, switch to `root` with `sudo su` command, and then download, install, and set up Chef server software by following steps 1 through 6 inclusively in [the official documentation](https://docs.chef.io/install_server.html#standalone):

> 1. Download the package from https://downloads.chef.io/chef-server/. *(I downloaded the package directly to the VM instance by using `wget` command.)*
> 2. Upload the package to the machine that will run the Chef server, and then record its location on the file system. The rest of these steps assume this location is in the /tmp directory.
>
> 1. As a root user, install the Chef server package on the server, using the name of the package provided by Chef. For Red Hat Enterprise Linux and CentOS:
>
>     ```shell
>     $ sudo rpm -Uvh /tmp/chef-server-core-<version>.rpm
>     ```
> 
>     For Ubuntu:
> 
>     ```shell
>     $ sudo dpkg -i /tmp/chef-server-core-<version>.deb
>     ```
> 
>     After a few minutes, the Chef server will be installed.
> 
> 4. Run the following to start all of the services:
> 
>     ```shell
>     $ chef-server-ctl reconfigure
>     ```
> 
>     Because the Chef server is composed of many different services that work together to create a functioning system, this step may take a few minutes to complete.
> 
> 5. Run the following command to create an administrator:
> 
>     ```shell
>     $ chef-server-ctl user-create USER_NAME FIRST_NAME LAST_NAME EMAIL 'PASSWORD' --filename FILE_NAME
>     ```
>     An RSA private key is generated automatically. This is the user’s private key and should be saved to a safe location. The --filename option will save the RSA private key to the specified absolute path.
> 
>     For example:
> 
>     ```shell
>     $ chef-server-ctl user-create stevedanno Steve Danno steved@chef.io 'abc123' --filename /path/to/alex.pem
>     ```
> 
> 6. Run the following command to create an organization:
> 
>     ```shell
>     $ chef-server-ctl org-create short_name 'full_organization_name' --association_user user_name --filename ORGANIZATION-validator.pem
>     ```
> 
>     The name must begin with a lower-case letter or digit, may only contain lower-case letters, digits, hyphens, and underscores, and must be between 1 and 255 characters. For example: 4thcoffee.
> 
>     The full name must begin with a non-white space character and must be between 1 and 1023 characters. For example: 'Fourth Coffee, Inc.'.
> 
>     The --association_user option will associate the user_name with the admins security group on the Chef server.
> 
>     An RSA private key is generated automatically. This is the chef-validator key and should be saved to a safe location. The --filename option will save the RSA private key to the specified absolute path.
> 
>     For example:
> 
>     ```shell
>     $ chef-server-ctl org-create 4thcoffee 'Fourth Coffee, Inc.' --association_user stevedanno --filename /path/to/myorg-validator.pem
>     ```

NOTE: Remember filepaths used in steps 5 and 6.

### Chef Workstation Setup

1. SSH into your Chef workstation.
1. Switch to `root` with `sudo su` command.
    ```shell
    $ sudo su
    ```

1. [Download the software](https://downloads.chef.io/chef-workstation/) to your instance. Use `wget` command if you are on Ubuntu like so:
    ```shell
    $ wget https://packages.chef.io/files/stable/chef-workstation/CURRENT_CHEF_VERSION/ubuntu/16.04/chef-workstation_CURRENT_CHEF_VERSION-1_amd64.deb
    ```

1. Install the software. If you are on Ubuntu, this is done like so:
    ```shell
    $ dpkg -i chef-workstation_CURRENT_CHEF_VERSION-1_amd64.deb
    ```

1. Follow instructions from [the official documentation](https://docs.chef.io/chefdk_setup.html) to set up Chef on your workstation:

    > ### Configure Ruby Environment
    > Open a command window and enter the following:
    >
    > ```shell
    > $ which ruby
    > ```
    > which will return something like `/usr/bin/ruby`.
    >
    > To use the Chef development kit version of Ruby as the default Ruby, edit the $PATH and GEM environment variables to include paths to the Chef development kit. For example, on a machine that runs Bash, run:
    >
    > ```shell
    > $ echo 'eval "$(chef shell-init bash)"' >> ~/.bash_profile
    > ```
    > where bash and ~/.bash_profile represents the name of the shell.
    >
    > If zsh is your preferred shell then run the following:
    >
    > ```shell
    > $ echo 'eval "$(chef shell-init zsh)"' >> ~/.zshrc
    > ```
    > Run which ruby again. It should return `/opt/chefdk/embedded/bin/ruby`.  
    >
    > ### Add Ruby to $PATH
    > The Chef Client includes a stable version of Ruby as part of its installer. The path to this version of Ruby must be added to the `$PATH` environment variable and saved in the configuration file for the command shell (Bash, csh, and so on) that is used on the machine running ~~ChefDK~~ *Chef Workstation*. In a command window, type the following:
    >
    > ```shell
    > $ echo 'export PATH="/opt/chefdk/embedded/bin:$PATH"' >> ~/.configuration_file && source ~/.configuration_file
    > ```
    >
    > where configuration_file is the name of the configuration file for the specific command shell. For example, if Bash were the command shell and the configuration file were named bash_profile, the command would look something like the following:
    >
    > ```shell
    > $ echo 'export PATH="/opt/chefdk/embedded/bin:$PATH"' >> ~/.bash_profile && source ~/.bash_profile
    > ```
    >
    > ### Create the Chef repository
    > Use the chef generate repo to create the Chef repository. For example, to create a repository called chef-repo:
    >
    > ```shell
    > $ chef generate repo chef-repo
    > ```
    >
    > #### Create .chef Directory
    > The .chef directory is used to store three files:
    >
    > * `config.rb`
    > * `ORGANIZATION-validator.pem`
    > * `USER.pem`
    >
    Side note: I did not fine `config.rb` in the folder, so the instructions may be obsolete.
    > Where `ORGANIZATION` and `USER` represent strings that are unique to each organization. These files must be present in the .chef directory in order for ~~ChefDK~~ *Chef Workstation* to be able to connect to a Chef server.
    >
    > To create the `.chef` directory:
    >
    > In a command window, enter the following:
    >
    > ```shell
    > $ mkdir -p ~/chef-repo/.chef
    > ```
    >
    > Note that you’ll need to replace chef-repo with the name of the repository you created previously.
    >
    > After the .chef directory has been created, the following folder structure will be present on the local machine:
    >
    > ```
    > chef-repo/
    >    .chef/        << the hidden directory
    >    certificates/
    >    config/
    >    cookbooks/
    >    data_bags
    >    environments/
    >    roles/
    > ```
    >
    > Add .chef to the .gitignore file to prevent uploading the contents of the .chef folder to GitHub. For example:
    >
    > ```shell
    > $ echo '.chef' >> ~/chef-repo/.gitignore
    > ```
    > ### Configure the Chef Repository
    > #### Move `.pem` Files
    > Download the `ORGANIZATION-validator.pem` and `USER.pem` files from the Chef Server and move them to the `.chef` directory.

    In order to do that, we must first generate an SSH key on the workstation and tell our server to expect that key. First, on your workstation, run the following command:
    ```shell
    $ ssh-keygen
    ```
    Do not enter enything if the prompt asks you and simply press `Enter` on your keyboard. Among the messages you'll see should be the path to your public key, something along the lines of
    ```shell
    Your public key has been saved in ~/.ssh/id_rsa.pub.
    ```
    Use that path to look up the key with `cat` command:
    ```shell
    $ cat ~/.ssh/id_rsa.pub
    ```
    Copy the displayed key. Then, on your <u>server</u>, paste the key into `~/.ssh/authorized_keys` file. Save the file.

    On your <u>workstation</u>, copy `.pem.` files from your server (see step 6 from the server setup above):
    ```shell
    $ scp root@chefserver:/path/to/pem/keys/*.pem ~/chef-repo/.chef
    The authenticity of host 'chefserver (10.168.0.5)' can't be established.
    ECDSA key fingerprint is SHA256:IIIcJWrUhxqzMtKQ4XQ8tS7PdHQbBvNdwWfEIXRXWh4.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added 'chefserver,10.168.0.5' (ECDSA) to the list of known hosts.
    alex.pem                                                        100% 1674     1.6KB/s   00:00    
    myorg-validator.pem                                             100% 1674     1.6KB/s   00:00    
    ```
    Verify that the files are in the .chef folder:
    ```shell
    $ ls chef-repo/.chef
    alex.pem  myorg-validator.pem
    ```

    > #### Create the config.rb File
    > Navigate to the ~/chef-repo/.chef directory and create the config.rb using the knife configure tool. 

    First, you need to know the chef server URL, go to your <u>server</u> and look up its address:

    ```shell
    $ cat /etc/hosts
    127.0.0.1 localhost

    # The following lines are desirable for IPv6 capable hosts
    ::1 ip6-localhost ip6-loopback
    fe00::0 ip6-localnet
    ff00::0 ip6-mcastprefix
    ff02::1 ip6-allnodes
    ff02::2 ip6-allrouters
    ff02::3 ip6-allhosts
    169.254.169.254 metadata.google.internal metadata
    10.168.0.5 chefserver.us-west2-a.c.chef-demo-221022.internal chefserver  # Added by Google
    169.254.169.254 metadata.google.internal  # Added by Google
    ```

    I am not sure about other platforms, but GCP will have its `.internal` address tagged with "Added by Google". Copy it and use it when running `knife configure` command. Notice the little discrepancy between the default chef server value and the actual server address you need to put in:

    ```shell
    $ knife configure
    WARNING: No knife configuration file found. See https://docs.chef.io/config_rb_knife.html for details.
    Please enter the chef server URL: [https://chefwork.us-west2-a.c.chef-demo-221022.internal/organizations/myorg] https://chefserver.us-west2-a.c.chef-demo-221022.internal/organizations/myorg
    Please enter an existing username or clientname for the API: [aleksod] alex
    *****

    You must place your client key in:
    ~/.chef/alex.pem
    Before running commands with Knife

    *****
    Knife configuration file written to ~/.chef/credentials
    ```
    Per instructions above, I decided to copy all the `.pem` files to `~/.chef/` folder from `~/chef-repo/.chef/` folder:
    ```shell
    $ ls
    alex.pem  myorg-validator.pem
    $ cp *.pem ~/.chef/
    ```
    Moving on with [the official instructions](https://docs.chef.io/chefdk_setup.html#get-ssl-certificates):

    > ### Get SSL Certificates
    > Chef server 12 enables SSL verification by default for all requests made to the server, such as those made by knife and the chef-client. The certificate that is generated during the installation of the Chef server is self-signed, which means there isn’t a signing certificate authority (CA) to verify. In addition, this certificate must be downloaded to any machine from which knife and/or the chef-client will make requests to the Chef server.
    >
    > Use the knife ssl fetch subcommand to pull the SSL certificate down from the Chef server:
    ```shell
    $ knife ssl fetch
    WARNING: Certificates from chefserver.us-west2-a.c.chef-demo-221022.internal will be fetched and placed in your trusted_cert
    directory (~/.chef/trusted_certs).
    
    Knife has no means to verify these are the correct certificates. You should
    verify the authenticity of these certificates after downloading.
    
    Adding certificate for chefserver_us-west2-a_c_chef-demo-221022_internal in ~/.chef/trusted_certs/chefserver_us-west2-a_c_chef-demo-221022_internal.crt
    ```
    > ### Verify Install
    > The ChefDK is installed correctly when it is able to use knife to communicate with the Chef server.
    > 
    > To verify that ChefDK can connect to the Chef server:
    > 
    > In a command window, navigate to the Chef repository:
    > 
    > ```shell
    > $ cd ~/chef-repo
    > ```
    > In a command window, enter the following:
    > 
    > ```shell
    > $ knife client list
    > ```
    > 
    > to return a list of clients (registered nodes and ~~ChefDK~~ *Chef Workstation* installations) that > have access to the Chef server. For example:
    > 
    > ```shell
    > chefdk_machine
    > registered_node
    > ```

    At this point, though, since we have not registered any nodes yet, all we see is the following:
    ```shell
    $ knife client list
    myorg-validator
    ```

### Configuring Nodes
In our demo GCP environment, I opted to use an SSH handshake in order to communicate with nodes. Therefore, a key needs to be generated on the <u>workstation</u> and appended to `~/.ssh/authorized_keys` file on each of the nodes, similarly how we have done above. After that is done, we can proceed with [standard instructions](https://docs.chef.io/install_bootstrap.html):
>
> #### Bootstrap a Node
>
> A node is any physical, virtual, or cloud machine that is configured to be maintained by a > chef-client. In order to bootstrap a node, you will first need a working installation of the Chef > software package. A bootstrap is a process that installs the chef-client on a target system so that > it can run as a chef-client and communicate with a Chef server. There are two ways to do this:
>
> * Use the knife bootstrap subcommand to bootstrap a node using the Chef installer
> * Use an unattended install to bootstrap a node from itself, without using SSH or WinRM
> ##### knife bootstrap
> The knife bootstrap command is a common way to install the chef-client on a node. The default for > this approach assumes that a node can access the Chef website so that it may download the > chef-client package from that location.
>
> The Chef installer will detect the version of the operating system, and then install the > appropriate version of the chef-client using a single command to install the chef-client and all of > its dependencies, including an embedded version of Ruby, RubyGems, OpenSSL, key-value stores, > parsers, libraries, and command line utilities.
>
> The Chef installer puts everything into a unique directory (/opt/chef/) so that the chef-client > will not interfere with other applications that may be running on the target machine. Once > installed, the chef-client requires a few more configuration steps before it can perform its first > chef-client run on a node.
>
> ###### Run the bootstrap command
>
> The knife bootstrap subcommand is used to run a bootstrap operation that installs the chef-client > on the target node. The following steps describe how to bootstrap a node using knife.
>
> Identify the FQDN or IP address of the target node. The knife bootstrap command requires the FQDN > or the IP address for the node in order to complete the bootstrap operation.
>
> Once the workstation machine is configured, it can be used to install the chef-client on one (or > more) nodes across the organization using a knife bootstrap operation. The knife bootstrap command > is used to SSH into the target machine, and then do what is needed to allow the chef-client to run > on the node. It will install the chef-client executable (if necessary), generate keys, and register > the node with the Chef server. The bootstrap operation requires the IP address or FQDN of the > target system, the SSH credentials (username, password or identity file) for an account that has > root access to the node, and (if the operating system is not Ubuntu, which is the default > distribution used by knife bootstrap) the operating system running on the target system.
>
> In a command window, enter the following:
>
> ```shell
> $ knife bootstrap 123.45.6.789 -x username -P password --sudo
> ```
> where 123.45.6.789 is the IP address or the FQDN for the node. Use the --distro option to specify a > non-default distribution. For more information about the options available to the knife bootstrap > command for Ubuntu- and Linux-based platforms, see knife bootstrap. For Microsoft Windows, the > knife windows plugin is required, see knife windows.

NOTE: GCP VM instances are aware of each other by name, and `root` user does not have any password by default, so the command above specifically for this example can be simplified to

```shell
$ knife bootstrap chefnode1
...
$ knife bootstrap chefnode2
...
$ knife bootstrap chefnode3
```

> And then while the bootstrap operation is running, the command window will show something like the following:
>
> ```shell
> Bootstrapping Chef on 123.45.6.789
> 123.45.6.789 knife sudo password:
> Enter your password:
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:05 -0700] INFO: *** Chef 10.12.0 ***
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:07 -0700] INFO: Client key /etc/chef/client.pem is not present > - registering
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:15 -0700] INFO: Setting the run_list to [] from JSON
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:15 -0700] INFO: Run List is []
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:15 -0700] INFO: Run List expands to []
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:15 -0700] INFO: Starting Chef Run for name_of_node
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:15 -0700] INFO: Running start handlers
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:15 -0700] INFO: Start handlers complete.
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:17 -0700] INFO: Loading cookbooks []
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:17 -0700] WARN: Node name_of_node has an empty run list.
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:19 -0700] INFO: Chef Run complete in 3.986283452 seconds
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:19 -0700] INFO: Running report handlers
> 123.45.6.789
> 123.45.6.789 [Fri, 07 Sep 2012 11:05:19 -0700] INFO: Report handlers complete
> 123.45.6.789
> ```
> After the bootstrap operation has finished, verify that the node is recognized by the Chef server. > To show only the node that was just bootstrapped, run the following command:

```shell
$ knife client show chefnode1.us-west2-a.c.chef-demo-221022.internal
```

> where name_of_node is the name of the node that was just bootstrapped. The Chef server will return something similar to:

```shell
admin:     false
chef_type: client
name:      chefnode1.us-west2-a.c.chef-demo-221022.internal
validator: false
```

> and to show the full list of nodes (and workstations) that are registered with the Chef server, run the following command:
>
> ```shell
> $ knife client list
> ```
> The Chef server will return something similar to:

```shell
chefnode1.us-west2-a.c.chef-demo-221022.internal
chefnode2.us-west2-a.c.chef-demo-221022.internal
chefnode3.us-west2-a.c.chef-demo-221022.internal
myorg-validator
```

Our nodes are configured and we can finally move on to managing our tiny mock-up infrastructure using Chef!

# Managing Infrastructure with Chef

The infrastructure is managed via our workstation. We are going to write Chef recipes on the workstation, upload them to the server, and then the server will apply thos erecipes to our nodes.

## Creating recipes

We should already have `~/chef-repo` on the workstation due to all the setup we've done so far. Using `tree` tool, we can see that the structure of the folder is as follows:

```shell
$ tree
.
├── chefignore
├── cookbooks
│   ├── example
│   │   ├── attributes
│   │   │   └── default.rb
│   │   ├── metadata.rb
│   │   ├── README.md
│   │   └── recipes
│   │       └── default.rb
│   └── README.md
├── data_bags
│   ├── example
│   │   └── example_item.json
│   └── README.md
├── environments
│   ├── example.json
│   └── README.md
├── LICENSE
├── README.md
└── roles
    ├── example.json
    └── README.md

8 directories, 14 files
```

We are going to follow steps in [Quick Start](https://docs.chef.io/quick_start.html). To create our first recipe within out first cookbook, navigate to `~/chef-repo/cookbooks` and execute the command

```shell
$ chef generate cookbook first_cookbook
```

After the cookbook is generated, edit the file `~/chef-repo/cookbooks/first_cookbook/recipes/default.rb` to contain:

```ruby
file "#{ENV['HOME']}/test.txt" do
  content 'This file was created by Chef!'
end
```

As the Ruby code above suggests, this tells Chef to create or alter the file `~/text.txt` so that it contains the mentioned above text. We can make Chef do this locally as shown in the Quick Start guide, but we are more interested in propagating changes to all of our nodes. So let's do just that.

Applying cookbooks and recipes to multiple nodes is pretty simple. In our case, this is done out of `~/chef-repo/cookbooks` folder with one command:

```shell
$ chef-run chefnode[1:3] first_cookbook::default
```

The output of the command should be similar to this:

```shell
[✔] Packaging cookbook... done!
[✔] Generating local policyfile... exporting... done!
[✔] Applying first_cookbook::default from /root/chef-repo/cookbooks/first_cookbook to targets.
├── [✔] [chefnode1] Successfully converged first_cookbook::default.
├── [✔] [chefnode2] Successfully converged first_cookbook::default.
└── [✔] [chefnode3] Successfully converged first_cookbook::default.
```

If you get authentication errors, run following commands on your workstation:

```shell
$ eval `ssh-agent -s`
...
$ ssh-add
```

Also, make sure that your workstation key in `~/.ssh/id_rsa.pub` is the same as the one in `~/.ssh/authorized_keys` on all of your nodes. Restart your nodes after adding the workstation key to them.

To make sure the changes you specified in the recipe have been applied to the nodes, log into them and look for `/root/test.txt` file with `This file was created by Chef!` text. The file should be on every node where the changes have been successfully applied.

Of course, this was a very simple example, but you can do much more with Chef than creating text files on nodes. A more comprehensive Ruby tutorial in Chef context can be found [here](https://docs.chef.io/ruby.html).

# Conclusion

With Chef, you essentially can turn your infrastructure into code. This code has many of the same attributes as your regular programming code, i.e. it is versionable, testable, and repeatable. You can easily apply updates to your infrastructure with various nodes in one command line.

[1]: https://docs.chef.io/platform_overview.html#uploading-your-code-to-chef-server "Chef server description"
[2]: https://docs.chef.io/nodes.html "About Nodes"
[3]: https://docs.chef.io/chef_overview.html "An Overview of Chef"