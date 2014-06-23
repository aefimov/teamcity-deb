### Overview
Generator of custom Debian Package for JetBrains TeamCity Server and Agent.

Details of TeamCity Server configuration:
- is accessible via HTTPS with signed certificates (package installs `nginx` as HTTPS frontend)
- uses TeamCity Data Directory located at `/var/lib/teamcity/BuildServer`

The package installs TeamCity server start and stop scripts into 
`/etc/init.d` and also provides easy means to upgrade to newer TeamCity versions.

`postfix` package is recommended to be used as a mail server to send out TeamCity email notifications.

### Step 1. Clone repo to your own copy
```
mkdir teamcity-deb
cd teamcity-deb
git init
git remote add upstream git://github.com/aefimov/teamcity-deb.git
git fetch upstream
git merge upstream/master
```
Then add your Organization repository as `origin`:
```
git remote add origin git@git.acme.com:/namespace/teamcity-deb.git
```
And push changes into it:
```
git push -u origin master
```

### Step 2. Create SSL keys for your TeamCity Server
In the root of Git repo:
```
mkdir ssl-certs
cd ssl-certs
openssl req -new -out request.csr
```
Enter password you wish (any you like, but remember it) and answer the questions
`openssl` asks. These generally depend on your Organization rules about issuing
certificates.
```
cat request.csr
```
Submit this certificate request to issue certification system of
your Organization (ask admin for this). In the result you should have signed
`certnew.cer` file or something like this.
Then convert private key into nginx key:
```
openssl rsa -in privkey.pem -out ../etc/nginx/ssl/teamcity.key
```
Enter password you remembered on previous step.
Then download Certificates Chain for you Organization into file `allcas.pem`,
and perform merge for CA certs with your new issued certificate:
```
cat certnew.cer allcas.pem > ../etc/nginx/ssl/teamcity.crt
cd ..
```
Don't forget to commit these new key and cert:
```
git add etc/nginx/ssl/teamcity.key
git add etc/nginx/ssl/teamcity.crt
git commit -m "Update SSL certs"
git push origin master
```
You also can save `ssl-certs` directory in your Git repo, but note that this is
not secure:
```
git add ssl-certs
git commit -m "Add artifacts of SSL keys creation"
git push origin master
```

### Step 3. Create SSH keys for `teamcity` user
TeamCity Server and Agent run under user `teamcity`. If you want to use SSH
authorization to access Version Control servers you need to generate SSH keys
(private and public) for the user.

```
ssh-keygen -t rsa -N "" -C teamcity -f ssh-keys/id_rsa
```
When it prompts for overwriting existing keys, you should agree.
Don't forget to commit these new keys:
```
git add ssh-keys/id_rsa*
git commit -m "Update SSH keys"
git push origin master
```

### Step 4. Build package
In Git repo root and on Ubuntu/Debian host:
Update version (do not modify the first part of the version, it isused for downloading
TeamCity distribution from JetBrains site):
```
dch -i
```
Build:
```
debuild
```
Upload to you local Debian distributives repo:
```
debrelease
```
#### Version of TeamCity and Version of Debian Package
Version of TeamCity is the first part of debian package version from the first line of
`debian/changelog`. For example:
```
teamcity (7.1.4.0) lucid; urgency=low
```
Version of TeamCity – `7.1.4`
Version of Debian Package – `7.1.4.0`

During the package build TeamCity version is embedded into `postinst` script.
On package install, the script will automatically download the specified 
TeamCity version from JetBrains Downloads site, and install it.

### Setup TeamCity Server
Add the folowing line into ```/etc/apt/sources.list``` if it is not yet present there:
```
deb http://dist.acme.com/packages stable/all/
```
Then:
```
sudo apt-get update && sudo apt-get install teamcity-server
```
If you like to use mysql as localhost database:
```
sudo apt-get update && sudo apt-get install teamcity-server-localhost-mysql
```
Package will create mysql database on localhost and configure TeamCity to use it.

#### TeamCity Server Startup and Shutdown
Start:
```
sudo /etc/init.d/teamcity start
```
Stop:
```
sudo /etc/init.d/teamcity stop
```
Restart:
```
sudo /etc/init.d/teamcity restart
```

### Setup TeamCity Agent
Add the folowing line into ```/etc/apt/sources.list``` if it is not yet present there:
```
deb http://dist.acme.com/packages stable/all/
```
Then:
```
sudo apt-get update && sudo apt-get install teamcity-agent
```
This installs the script `/etc/init.d/teamcity-agent`.
Run it to install agent on the host:
```
sudo /etc/init.d/teamcity-agent install <agent.name.without.spaces> <server.url>
```
This will download Agent from TeamCity server and install it on the current host. After
this you can run it:
```
sudo /etc/init.d/teamcity-agent start <agent.name.without.spaces>
```

#### TeamCity Agents Startup and Shutdown
Get list of installed agents:
```
sudo /etc/init.d/teamcity-agent
```

Start:
```
sudo /etc/init.d/teamcity-agent start <agent-name>
```
Stop (gently):
```
sudo /etc/init.d/teamcity-agent stop <agent-name>
```
Stop (killing):
```
sudo /etc/init.d/teamcity-agent kill <agent-name>
```
Restart:
```
sudo /etc/init.d/teamcity-agent restart <agent-name>
```

### Upgrade TeamCity Server. Bad practice
```
sudo /etc/init.d/teamcity upgrade 7.1.4
```
Where ```7.1.4```, is the version published on JetBrains Site. This script will
download tar.gz, unpack it into `/usr/local/teamcity-7.1.4` and link to
`/usr/local/teamcity`. Then TeamCity will be restarted.

On the first install the upgrade will be performed automatically.

### Upgrade TeamCity Server. Good practice.
Update package via:
```
dch -i
```
Change TeamCity version to the new one.
```
git commit -a -m "Upgrade to TeamCity 7.1.4"
git push origin master
debuild
debrelease
```
Then just install the new package on target hosts.

### Database

You need to use `teamcity-server-localhost-mysql` package.
This package uses `mysql` as one of the most popular on Ubuntu.

Database `teamcity` is created automatically on `localhost` for user `teamcity`, with
password `teamcity` on first install. For security reasons it is advised to modify 
these settings in the package or otherwise to comply with your Organization rules.

On the very first start after initial TeamCity installation, it might not use 
MySQL database. If you see a warning in TeamCity UI about 'External database',
just restart TeamCity server:
```
sudo /etc/init.d/teamcity restart
```
And then:
```
tail -f /usr/local/teamcity/logs/teamcity-server.log
```
When TeamCity prompts for an authentication token, copy&paste it from this log and continue.
