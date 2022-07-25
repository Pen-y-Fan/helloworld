# Hello world NGINX unit

This is an example taken from NGINX unit [README](https://github.com/nginx/unit) and [documentation](https://unit.nginx.org/), working with Ubuntu wsl2

I wanted to experiment with NGINX unit as it offers the ability to specify the runtime for the project, this will be particularly useful when some projects are PHP 7.4, some PHP 8.0 and others PHP 8.1.

## Notes

These are my notes from creating this project, which will help me write a blog in the future. "How to install NGINX unit on WSL 2 (Ubuntu)", similar to my blog [How to install MySQL on WSL 2 \(Ubuntu\)](https://pen-y-fan.github.io/2021/08/08/How-to-install-MySQL-on-WSL-2-Ubuntu/)

### Installation

Source 
- README for [NGINX unit](https://github.com/nginx/unit)
- official [NGINX documentation](https://unit.nginx.org/installation/#ubuntu)

The official docs use **#** for **su** or **sudo** account and **$** for a standard user, my notes have been changed to prepend `sudo`. Any commands with sudo will prompt for the password you set when you installed the wsl, if you can't remember the password see [Set up your Linux username and password](https://docs.microsoft.com/en-us/windows/wsl/setup/environment).

```shell
sudo apt update && sudo apt upgrade
```

Enter the password you set when you installed Ubuntu WSL.

Wait for the packages to install, there will be a lot of information scrolling up the screen!

Run the configuration script (from the NGINX unit README)

```shell
curl -sL 'https://unit.nginx.org/_downloads/setup-unit.sh' | sudo -E bash
```

Now check the configuration file:

```shell
cat /etc/apt/sources.list.d/unit.list
```

```text
deb [signed-by=/usr/share/keyrings/nginx-keyring.gpg] https://packages.nginx.org/unit/ubuntu/ focal unit
deb-src [signed-by=/usr/share/keyrings/nginx-keyring.gpg] https://packages.nginx.org/unit/ubuntu/ focal unit
```

If this file hasn't been created you can manually add it using `sudo nano /etc/apt/sources.list.d/unit.list`

Next install NGINX unit and any packages, in this example I install unit-dev and unit-php, chose you required packages see the [official documentation](https://unit.nginx.org/installation/#ubuntu) for the latest

```shell
sudo apt update && sudo apt upgrade
sudo apt install unit
sudo apt install unit-dev unit-php
```

Choice of packages at the time of writing:

```text
unit-dev unit-go unit-jsc11 unit-jsc16 unit-jsc17 unit-jsc18 unit-perl unit-php unit-python2.7 unit-python3.10 unit-ruby
```

### start the unit service

The documented way to start unit in ubuntu `systemctl restart unit` **doesn't** work!, wsl's version of Ubuntu does not have **systemd**!

```text
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
```

The workaround is to manually start the service source [Docker file for NGINX unit](https://github.com/nginx/unit-examples/blob/master/Dockerfile), which has the command. This took a lot of searching to find, I hope it helps!

```shell
sudo unitd --control unix:/var/run/control.unit.sock
```

## Configure the application

If you have cloned this project, edit the **config.json** and change the path for the project:

- was: "root": "/www/helloworld/"
- now: "root": "/your/path/"

Note: this has to be from the **root**, if you have it in your profile **~/**, then the root should start **/home/your-user-name/www/helloworld**

If you want to re-create yourself:

```shell
sudo mkdir /www/helloworld
sudo cd /www/helloworld
sudo nano config.json
```

Add the config:

```json
{
    "helloworld": {
        "type": "php",
        "root": "/www/helloworld/"
    }
}
```

If you are unfamiliar with **nano**, it is much easier to use then **vi** or **vim** editors, as the commands are along the bottom of the screen:

- **^O** means to write the file, it will prompt for a file name, press enter, as the file name was set **config.json** when opened
- **^X** Exit means CTRL + X will exit the app, back to the terminal

### Configuring an application

This is a two-step process, adding a config file and listening for a connection

```shell
sudo curl -X PUT --data-binary @config.json --unix-socket  \
    /var/run/control.unit.sock http://localhost/config/applications
```

```json
{
	"success": "Reconfiguration done."
}
```

If you get `curl: (7) Couldn't connect to server`, the unit service is not running, check the above and make use NGINX unit is installed and the service is running. For more detail see Troubleshooting below.

### Add a listener

```shell
sudo curl -X PUT -d '{"127.0.0.1:8000": {"pass": "applications/helloworld"}}'  \
       --unix-socket /var/run/control.unit.sock http://localhost/config/listeners
```

```json
{
	"success": "Reconfiguration done."
}
```

## Check the config

```shell
sudo curl --unix-socket /var/run/control.unit.sock http://localhost/config/
```

```json
{
        "listeners": {
                "127.0.0.1:8000": {
                        "pass": "applications/helloworld"
                }
        },

        "applications": {
                "helloworld": {
                        "type": "php",
                        "root": "/www/helloworld/"
                }
        }
}
```

## Change the owner

To make it easier to edit the project the owner for /www can be set to your username:

```shell
ls /home -l
```

Note your username and group, in my case its michael michael, replace michael:michael with your username:

```shell
sudo chown michael:michael /www -R
```

## Create hello world

Now you can create hello world's index.php without sudo:

```shell
cd /www/helloworld
nano index.php
```

Type or paste in:

```text
<?php echo "<h1>Hello, PHP on Unit!</h1>"; ?>
```

If you are unfamiliar with **nano**, it is much easier to use then **vi** or **vim** editors, as the commands are along the bottom of the screen:

- **^O** means to write the file, it will prompt for a file name, press enter, as the file name was set **index.php** when opened
- **^X** Exit means CTRL + X will exit the app, back to the terminal

### View the website

Now the index has been created you can open your browser (in windows) to <http://localhost:8000>, which will display **Hello, PHP on Unit!**

## Next steps

Now the first project has been created and is working, I suggest to follow the [documentation](https://unit.nginx.org/) for your next project.

Enjoy and good luck!

## Troubleshooting

The main problem I came across was getting the service to start, `curl: (7) Couldn't connect to server`, searches didn't yield much help. The issue was the NGINX unit service (unitd) wasn't starting automatically. 

### Check NGINX unit is installed

First check the NGINX unit service is installed:

```shell
unitd --version
```

```text
unit version: 1.27.0
configured as ./configure --prefix=/usr --state=/var/lib/unit --control=unix:/var/run/control.unit.sock --pid=/var/run/unit.pid --log=/var/log/unit.log --tmp=/var/tmp --user=unit --group=unit --tests --openssl --modules=/usr/l
ib/unit/modules --libdir=/usr/lib/x86_64-linux-gnu --cc-opt='-g -O2 -fdebug-prefix-map=/data/builder/debuild/unit-1.27.0/pkg/deb/debuild/unit-1.27.0=. -specs=/usr/share/dpkg/no-pie-compile.specs -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' --ld-opt='-Wl,-Bsymbolic-functions -specs=/usr/share/dpkg/no-pie-link.specs -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie'
```

If you don't see any version information check unit is installed, see above for installation information.

### Manually start the service

The documented way to start unit in ubuntu `systemctl restart unit` **doesn't** work!, wsl's version of Ubuntu does not have **systemd**!

```text
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
```

The workaround is to manually start the service \(source [Docker file for NGINX unit](https://github.com/nginx/unit-examples/blob/master/Dockerfile)\), which has the command. This took a lot of searching to find, I hope it helps!

```shell
sudo unitd --control unix:/var/run/control.unit.sock
```

```text
2022/07/17 13:21:07 [info] 49#49 unit 1.27.0 started
```

You will not be able to apply to config above.

### Check the config

If your config has the wrong path you will get:

```text
{
        "error": "Failed to apply new configuration."
}
```

Check the path set in your config e.g: if a new project was created in helloworld2, but the config.json has a typo, helloworld3, then to check the directory exists copy path and from a terminal type ls then paste in the path (not type it as you my type incorrectly!)

```shell
ls /www/helloworld3
```

```text
ls: cannot access '/www/helloworld3/': No such file or directory
```

If you get this error then correct your path, navigate to the correct path and type **pwd**

```shell
cd /www
cd /helloworld2
pwd
```

```text
/www/helloworld2
```

Copy the path output and paste it in the config file.

Once the **config.json** has been updated, rerun the config: 

```shell
sudo curl -X PUT --data-binary @config.json --unix-socket  \
    /var/run/control.unit.sock http://localhost/config/applications
```

```json
{
        "success": "Reconfiguration done."
}
```

Note: the output above confirms this is a **Reconfiguration**.

```shell
sudo curl --unix-socket /var/run/control.unit.sock http://localhost/config/
```

```json
{
  "listeners": {
    "127.0.0.1:8000": {
      "pass": "applications/helloworld"
    }
  },

  "applications": {
    "helloworld": {
      "type": "php",
      "root": "/www/helloworld2/"
    }
  }
}
```

You can see the **helloworld** app is now being served from **/www/helloworld2/**, is this what I intend? Maybe or maybe not, as I coped the config, but didn't change the app name **helloworld** to **helloworld2**, so the new config has replaced the existing one!

I hope this information will help you get started, see the [documentation](https://unit.nginx.org/) for further guides.
