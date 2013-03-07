### Overview
Debian Package for JetBrains TeamCity Server and Agent.

This package will install `mysql` as database on localhost and
`nginx` as HTTPS frontend.
Recommends `postfix` as mailer.

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

### Step 2. Creating SSL keys for Your TeamCity Server
In root of Git repo:
```
mkdir ssl-certs
cd ssl-certs
openssl req -new -out request.csr
```
Enter password you wish (anyone you like, but remember it), answer on questions
`openssl` asks from you, depends on your Organization rules about issuing
certificates.
```
cat request.csr
```
Perform submit this request for certificate to issue certification system on
your Organization (ask admin for this). In result you must have signed
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
You also can save `ssl-certs` directory in your Git repo, but this is not very
secure:
```
git add ssl-certs
git commit -m "Add artifacts of SSL keys creation"
git push origin master
```

### Step 3. Creating SSH keys for `teamcity` user
TeamCity Server and Agent runned under user `teamcity`. If you want to use SSH
authorization to access into Version Control you need to generate SSH keys
(private and public).

```
ssh-keygen -t rsa -N "" -C teamcity -f ssh-keys/id_rsa
```
It prompted for overwriting existing keys, you should agree.
Don't forget to commit these new keys:
```
git add ssh-keys/id_rsa*
git commit -m "Update SSH keys"
git push origin master
```

### Step 4. Build package
In Git repo root and on Ubuntu/Debian host:
Update version (do not modify first version part, it used for downloading
TeamCity from JetBrains site)
```
dch -i
```
Build
```
debuild
```
Upload to you local Debian distributives repo:
```
debrelease
```
#### Version of TeamCity and Version of Debian Package
Version of TeamCity is first part of debian package version from `debian/changelog`
first line. For example:
```
teamcity (7.1.4.0) lucid; urgency=low
```
Version of TeamCity – `7.1.4`
Version of Debian Package – `7.1.4.0`

On package build TeamCity version is directly put into `postinst` script and it
will uprgade TeamCity automatically on package install. Script will download TeamCity
for specified version from JetBrains Downloads site, and install it.

### Setup TeamCity Server
Put into ```/etc/apt/sources.list``` (if not exists):
```
deb http://dist.acme.com/packages stable/all/
```
Then:
```
sudo apt-get update && sudo apt-get install teamcity-server
```
Package will create mysql database on localhost, connect TeamCity with it.

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
Put into ```/etc/apt/sources.list``` (if not exists):
```
deb http://dist.acme.com/packages stable/all/
```
Then:
```
sudo apt-get update && sudo apt-get install teamcity-agent
```
This install for you script `/etc/init.d/teamcity-agent`, run this one to
install agent on host:
```
sudo /etc/init.d/teamcity-agent install <agent.name.without.spaces> <server.url>
```
This will download Agent from TeamCity server and install on this host. After
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
Where ```7.1.4```, is version published on JetBrains Site. This script will
download tar.gz, unpack it into `/usr/local/teamcity-7.1.4` and link to
`/usr/local/teamcity`. Then TeamCity will be restarted.

Version upgrading on first install will performed automatically.

### Upgrade TeamCity Server. Good practice.
Update package via:
```
dch -i
```
Change TeamCity version to new one.
```
git commit -a -m "Upgrade to TeamCity 7.1.4"
git push origin master
debuild
debrelease
```
Then just install new package on target hosts.

### Database

This package is used `mysql` as one of most popular on Ubuntu.

Database `teamcity` created automatically on `localhost` for user `teamcity`, with
password `teamcity` on first install. For security issues you can modify this
package as you wish, and as your Organization is requires.

On first Run TeamCity will not use MySQL database, after first run and performing
some initial setup if you see warning of 'External database', please just restart
TeamCity server:
```
sudo /etc/init.d/teamcity restart
```
And then:
```
tail -f /usr/local/teamcity/logs/teamcity-server.log
```
When TeamCity prompt for token, copy&paste it from this log and continue.
