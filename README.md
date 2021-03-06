# wordpress_as_a_code

This repository includes playbook to make an aws based ec2 which hosts a wordpress site. 

The following resources will be provisoned with the playbook roles:

		vpc - create a new vpc, subnet, IGW, route table and security group
		ec2 - create ec2, elastic ip and add the ip to the ansible in-memory inventory
		updates - install and update nessessary components to run LAMP
		apache - configures apache2 document root, virtualhost (file is in files/apache.conf.j2), enable rewrite module , new site and disable the old site
		mysql - configures mysql users and database, removes the unused users and db
		wordpress - install wordpress and new plugins, removes the old ones, setups wp-config (fils is in files/wp-config.php.j2) and configures the folder and file ownership and rights

All the variables, which are used, are defined in folder group_vars, file vars.yml or in encrypted ansible vault file vault.

Vars variables:

		key_name: stock # name of the key, which is used for connection
		region: eu-north-1 # region, which You want to use for ec2 provisioning
		image: ami-01996625fff6b8fcc # ubuntu 18.04
		name: wordpress # name for the EC2 which is visible in the web console
		vpc_name: VPC for Wordpress #name for the new vpc
		sg_name: SG for Wordpress # Name for the Security Group to be used
		size: t3.small # ec2 size
		wordpress_subnet: 10.10.1.0/24 # wordpress subnet
		ansible_ssh_private_key_file: .ssh/publickey.pem  # key file, which ansible will use when loggin into the ec2
		ansible_user: ubuntu  # user, which ansible will use when logging into the ec2

		#HTTP Settings
		http_host: "your_domain" # for example www.kullakamakas.com
		http_conf: "your_domain.conf"
		http_port: "80" #default port

		# Indreku public key file
		pub_file: /home/ubuntu/indrek.pub # Public key which You want to upload to the newly created ec2 so that the users could connect
	
Vault enty:

		#MySQL Settings
		mysql_root_password: "mysql_root_password" # the mysql root user password
		mysql_db: "wordpress" # db used by wordpress
		mysql_user: "tavid" # regular user
		mysql_password: "password" # and its password

	
File directory:

		apache.conf.j2 # apache conf, which will replace the default one
		wp-config.php.j2 # wp config file, which will be used

What do You need to know:

Python3 is needed as the interpreter, aws credentials should be stored in
the environment variables. You can store them in the vault, but then You
need to change the scripts to include them as variables. 
All the roles,which run towards the ec2, are run with sudo rights. 
The play will not use default aws provisoned services (vpc, network, sg, etc).	
There is no inventory file, the playbook will add the created ec2 ip to the
ansible in-memory inventory.

Just a small reminder:

    ansible-playbook - playbook.yml --ask-vault-pass '-e ansible_python_interpreter=/usr/bin/python3'

**When the play runs and the ec2 is created, then You need to insert "yes" to the cli, inorder to trust the connection being made towards the new ec2.**


What was not included into the plays:
Scripts, which install the wpi cli and activate the plugins and ssl
certificate addon. These need to be execute after the wordpress 5 minute
installation and need some prework, for example adding a valid dns record.


    - name: Install WP CLI, create WP user and activate all plugins

    shell: |

    curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar

    chmod +x wp-cli.phar

    mv wp-cli.phar /usr/local/bin/wp

    sudo -u {{ wp_user }} -i -- wp plugin activate --path='/var/www/{{ http_host }}/' --all

    useradd -g www-data {{ wp_user }}

    mkdir /home/{{ wp_user }}

    chown -R {{ wp_user }}:www-data /home/{{ wp_user }}

    sudo -u {{ wp_user }} -i -- wp plugin activate --path='/var/www/{{ http_host }}/' --all



    The certbot actions have been commented out as we need a proper dns for certificate creation

    - name: Create and Install Cert Using apache Plugin

    shell: "certbot --{{ certbot_plugin }} -d {{ http_host }} -m {{ certbot_mail_address }} --agree-tos --noninteractive --redirect"

    - name: Set Letsencrypt Cronjob for Certificate Auto Renewal

    cron: name=letsencrypt_renewal special_time=monthly job="/usr/bin/certbot renew"



The repository also includes a play for cleanup: **remove-all.yml** which will remove the resources, which were created previously.