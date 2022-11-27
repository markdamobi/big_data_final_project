## Rails Installation on new server.

These are the steps I used to install my rails app on our 3 servers.


Servers                   | public ipv4 DNS
------------------------------------------------------------------------------------
single web server         | ec2-3-15-219-66.us-east-2.compute.amazonaws.com
Load balance server 1     | ec2-18-224-73-142.us-east-2.compute.amazonaws.com
Load Balancer server 2    | ec2-18-216-195-92.us-east-2.compute.amazonaws.com
load Balance url          | mpcs53014-loadbalancer-217964685.us-east-2.elb.amazonaws.com

My rails port is 3334.

## Process to deploy to a new server(assuming deploy already happened and capistrano config is already setup)
see https://www.phusionpassenger.com/library/deploy/standalone/automating_app_updates/ruby/ and https://www.phusionpassenger.com/library/walkthroughs/deploy/ruby/ownserver/standalone/oss/install_language_runtime.html


### install rvm, ruby, bundler and passenger

login as ec2-user(or any user with admin privileges.)
ssh ec2-user@server_address -i ~/.ssh/markdamobi_big_data.pem

1. sudo yum install -y curl gpg gcc gcc-c++ make
2. try one of the following and move to step 3.
- sudo gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
- sudo gpg2 --keyserver hkp://pool.sks-keyservers.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
- command curl -sSL https://rvm.io/mpapis.asc | sudo gpg2 --import - && command curl -sSL https://rvm.io/pkuczynski.asc | sudo gpg2 --import -
3. curl -sSL https://get.rvm.io | sudo bash -s stable
3.5a exit shell and come back in to activate rvm.
3.5b if sudo grep -q secure_path /etc/sudoers; then sudo sh -c "echo export rvmsudo_secure_path=1 >> /etc/profile.d/rvm_secure_path.sh" && echo Environment variable installed; fi
4. rvmsudo rvm install ruby-2.6.5
5. rvm --default use ruby-2.6.5
6. rvmsudo gem install bundler -N
7. Node is already installed. no need.
8. rvmsudo gem install passenger -N



### add server to capistrano production file.
  - see capistrano production.rb in rails app for details.


### Preparing the server.

sudo adduser markdamobi-project-rails-user
sudo su - markdamobi-project-rails-user
mkdir .ssh
chmod 700 .ssh
touch .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
vim .ssh/authorized_keys

paste public key from local machine in vim:
ssh-keygen -y -f ~/.ssh/markdamobi_big_data.pem (on local machine)

##exit to get to ec2 user.
exit
sudo mkdir -p /var/www/markdamobi-project-rails/shared
sudo chown markdamobi-project-rails-user: /var/www/markdamobi-project-rails /var/www/markdamobi-project-rails/shared

go back into deploy user - sudo su - markdamobi-project-rails-user

#### create master.key and Passengerfile.json

vim /var/www/markdamobi-project-rails/shared/Passengerfile.json

{
  "environment": "production",
  "port": 3334,
  "daemonize": true,
  "user": "markdamobi-project-rails-user"
}

#exit to use ec2-user to write to this file master.key.
exit
sudo mkdir -p /var/www/markdamobi-project-rails/shared/config
sudo vim /var/www/markdamobi-project-rails/shared/config/master.key
#get master key(from local machine)
cat /Users/mark_amobi/CloudStorage/Google\ Drive/school/Uchicago/Fall_2020/big_data/markdamobi-project-rails/config/master.key

sudo chown -R markdamobi-project-rails-user: /var/www/markdamobi-project-rails/shared/config
sudo chmod 600 /var/www/markdamobi-project-rails/shared/config/master.key
sudo chown -R markdamobi-project-rails-user: /var/www/markdamobi-project-rails/shared/config/master.key

### ensure app restarts on system reboot.
sudo vim /etc/rc.local
then paste the 3 lines below.

cd /var/www/markdamobi-project-rails/current
bundle exec passenger start
cd ~



### deploy to new server.

#### add deploy key to github.

- sudo su - markdamobi-project-rails-user
- ssh-keygen -t rsa -b 4096 -C "markdamobi@uchicago.edu" (dont type any password. just hit Enter 3 times.)
- cat .ssh/id_rsa.pub
- paste in github. (https://github.com/markdamobi/markdamobi-project-rails/settings/keys)
- finally deploy.
bundle exec cap production deploy (from local)

### An issue may come up relating to passenger. if passenger issue comes up, simply do this and try deploying again.
exit (to get into ec2-user)
rvmsudo passenger start /var/www/markdamobi-project-rails/current

app should be running on port 3334
I don't understand why the issue above shows up but it does...



### setup up details for API access.
vim /var/www/markdamobi-project-rails/shared/.env.production.local

paste
HBASE_API_HOST=ec2-3-15-219-66.us-east-2.compute.amazonaws.com
HBASE_API_PORT=3335

HBASE_API_HOST=ec2-18-224-73-142.us-east-2.compute.amazonaws.com
HBASE_API_PORT=3335

HBASE_API_HOST=ec2-18-216-195-92.us-east-2.compute.amazonaws.com
HBASE_API_PORT=3335


### Install googler tool.(for my News Engine)
```
sudo curl -o /usr/local/bin/googler https://raw.githubusercontent.com/jarun/googler/v4.3.1/googler && sudo chmod +x /usr/local/bin/googler
```

And also install python on all the servers with:

```
sudo yum install python37

```