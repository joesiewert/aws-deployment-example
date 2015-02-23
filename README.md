# Deploying to Amazon S3

## Objectives

Expose you to everything that happens "under the hood" when you deploy a Rails app.

**Expectations**

This is an entire field of the industry.  You are not expected to be able to reproduce
these steps from memory or be able to deeply understand all the concepts by the
end of this lesson.

**Disclaimer**

This is _not_ an exhaustive demo.  This doesn't cover a number of crucial topics
such as log rotation, scaling, security etc...

## Setup

* Open this README in your text editor of choice
* Turn off "soft wrap" if it's enabled
* Run `bin/setup`, then `rails s` then open http://localhost:3000 to make sure it works
* Turn off your local server (we won't need it anymore)
* Have two terminal tabs (and only two) open
  * The one on the left will be the "server"
  * The one on the right will be your local

## Launch an EC2 Instance

1. Visit https://610753855993.signin.aws.amazon.com/console
1. Login with `g4-ec2-demo` and the password from the daily notes.
1. Click "Launch Instance"
1. Next to "Amazon Linux AMI 2014.09.2 (HVM)" click "Select"
1. Check the box next to `t2.medium`
1. Click "Review and Launch"
1. In the "Boot from General Purpose" dialog, click "Next"
1. Click "Edit Security Groups"
1. Add rule for HTTP on port 80
1. Under Security Group Name, add your name (like `Jeff Dean`)
1. Click "Review and Launch"
1. Click "Launch"
1. Select "Create a new key pair"
1. Type your name like so `jeff-dean`
1. Click "Download Key Pair" - this should end up in your Downloads
1. Click "Launch instances"
1. Click on the name of your instance above
1. Name your instance

## Connect to your Instance

1. Click "Connect"
1. `cd` into your Downloads directory
1. Run the line like `chmod 400 jeff-dean.pem`
1. Run `ssh -i jeff-dean.pem ec2-user@54.148.101.119` to connect to your server
1. Say "yes" to the prompt
1. You are now logged into your new server!

Verify:

1. Logout
1. Log back in

## Get Ruby Installed

On the remote server, run each of these one-by-one:

```
sudo yum -y update
sudo yum -y install gcc gcc-c++ libcurl-devel openssl-devel zlib-devel
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
\curl -sSL https://get.rvm.io | bash -s stable
echo 'source ~/.profile' >> /home/ec2-user/.bash_profile
source /home/ec2-user/.rvm/scripts/rvm
rvm requirements
rvm install ruby
rvm use ruby --default
```

NOTE: `rvm install ruby` will take a while

Verify

```
which ruby
# ~/.rvm/rubies/ruby-2.2.0/bin/ruby
ruby -v
# ruby 2.2.0p0 (2014-12-25 revision 49005) [x86_64-linux]
```

Resources:

* http://curl.haxx.se/libcurl/
* https://rvm.io/
* https://www.gnupg.org/
* http://en.wikipedia.org/wiki/Yellowdog_Updater,_Modified

NOTE on copy / pasting with return character.
NOTE on VIM.

## Install and Run Apache

On the remote server:

```
sudo yum -y install httpd httpd-devel apr-devel apr-util-devel
sudo service httpd start
```

Verify:

1. Go to the AWS Console
1. Find the public DNS address and open it in a browser

Resources

* https://apr.apache.org/
* http://httpd.apache.org/

## Get Passenger Installed

On the remote server, run:

```
gem install passenger --no-ri --no-rdoc
sudo chmod o+x "/home/ec2-user"
passenger-install-apache2-module
```

NOTE: `passenger-install-apache2-module` will cause a long wait

Paste the following command all at once:

```
sudo bash -c "cat >> /etc/httpd/conf.d/passenger.conf" <<EOL
LoadModule passenger_module /home/ec2-user/.rvm/gems/ruby-2.2.0/gems/passenger-5.0.4/buildout/apache2/mod_passenger.so
<IfModule mod_passenger.c>
 PassengerRoot /home/ec2-user/.rvm/gems/ruby-2.2.0/gems/passenger-5.0.4
 PassengerDefaultRuby /home/ec2-user/.rvm/gems/ruby-2.2.0/wrappers/ruby
</IfModule>
EOL
```

Then run:

```
sudo service httpd restart
```

Verify:

```
cat /etc/httpd/conf.d/passenger.conf
passenger-memory-stats
```

Resources

* https://www.phusionpassenger.com/

## Install PostgreSQL

Run the following commands on the remote server:

```
sudo yum install -y postgresql93 postgresql93-devel postgresql93-libs postgresql93-server
sudo service postgresql93 initdb
sudo service postgresql93 start
sudo -H -u postgres bash -c 'createuser rails -s'
sudo cp /var/lib/pgsql93/data/pg_hba.conf{,.bak}
sudo vi /var/lib/pgsql93/data/pg_hba.conf
```

Change the localhost line from `peer` to `trust` near the bottom.

```
sudo service postgresql93 restart
```

Verify:

```
psql -U rails postgres
```

You should be in a `psql` prompt.  `CTL+D` to exit.

Resources

* http://www.postgresql.org/docs/9.4/static/auth-methods.html

## Setup your Environment Variables

Environment variables:

Locally run `rake secret` to generate a big long secret.

On the remote server run:

```
sudo bash -c "echo 'export SECRET_KEY_BASE=your-secret-key-here' >> /etc/bashrc"
source /etc/bashrc
```

Verify:

```
echo $SECRET_KEY_BASE
```

You should see your secret.

## Push your App

Locally, from this directory run:

```
git archive -o app.tar.gz --prefix=app/ master
scp -i ~/Downloads/jeff-dean.pem app.tar.gz ec2-user@54.148.77.194:
```

NOTE: Replace these commands with your key name and IP address.

On the server run:

```
tar zxvf app.tar.gz
cd app
bundle --without development test
RAILS_ENV=production rake db:create db:migrate db:seed
RAILS_ENV=production rake assets:precompile
```

NOTE: `bundle` will take a long time.

Verify

To see if this is working, we can turn off Apache and just run Rails directly:

```
sudo service httpd stop
rvmsudo -E bash -c 'rails s -p 80 -b 0.0.0.0 -e production'
```

`CTL+C` to stop that server.

## Register the App with Apache

Paste the following code in all at once:

```
sudo bash -c "cat >> /etc/httpd/conf.d/app.conf" <<EOL
<VirtualHost *:80>
  ServerName ec2-54-148-77-194.us-west-2.compute.amazonaws.com
  DocumentRoot /home/ec2-user/app/public
  <Directory /home/ec2-user/app/pubic>
     # This relaxes Apache security settings.
     AllowOverride all
     # MultiViews must be turned off.
     Options -MultiViews
     # Uncomment this if you're on Apache >= 2.4:
     #Require all granted
  </Directory>
</VirtualHost>
EOL
```

Verify:

```
sudo service httpd restart
```

## That's it!

Well, kind of...
