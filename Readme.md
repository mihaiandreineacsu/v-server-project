# V-Server Setup

This is a documentation of setting up ssh login using SSH-Key Pairs, configure Nginx web server to access an alternative Web Page and configure Git on V-Server.

The setup was performed on host machine Windows 11 using Powershell Terminal and remote machine Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-105-generic x86_64).

Along reading this Documentation, you will come across many examples that contain wrapped expressions like for example `<some_expressions>`, that means the content wrapped, in this case `<some_expressions>` should be replaced with your own content.

---

## Table of Content

1. [Technologies](#technologies)
1. [SSH Login](#ssh-login)
1. [Nginx Alternative Web Page](#nginx-alternative-web-page)
1. [Git Configuration](#git-configuration)

---

## Technologies

- Host machine - Windows 11 using Powershell Terminal
- Remote machine Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-105-generic x86_64).
- SSH
- Nginx
- Git and Github

---

## SSH Login

### To configure a ssh login with ssh-key pairs follow this steps

```powershell
# On host machine generate a ssh-key pairs.
ssh-keygen -t ed25519

# Provide a name for the ssh-key files:
Generating public/private ed25519 key pair.
Enter file in which to save the key (~/.ssh/id_ed25519):  ~/.ssh/<ssh_key_name>

# Provide no input for the passphrase
Enter passphrase (empty for no passphrase):
Enter same passphrase again:

# After that the output of generating ssh-key pairs should look like this.
Your identification has been saved in <ssh_key_name>
Your public key has been saved in <ssh_key_name>.pub
The key fingerprint is:
SHA256:<some_hashed_generated_value> <hostmachine_username@hostmachine_name>
The key''s randomart image is:
+--[ED25519 256]--+
|             o.oo|
|            + +  |
|           o X . |
|          . B + o|
|        S  + = = |
|         .*.o.O. |
|         o+*.o.Bo|
|         .++o =+*|
|         E+o..+Bo|
+----[SHA256]-----+

# Add the public ssh-key to your v-server using the Powershell command:
Get-Content $env:USERPROFILE\.ssh\<ssh_key_name>.pub | ssh <user>@<remoteserver> "cat >> .ssh/authorized_keys"

# Test your ssh login using ssh-key:
ssh -i ~/.ssh/<ssh_key> <user>@<remoteserver>
```

### Deactivate ssh password login

> [!Note]
> One of the basic SSH hardening step is to disable password based SSH login. You know that you can use ssh with the root or other account’s password to login remotely into a Linux server. But this poses a security risk because a huge numbers of bots are always trying to login to your system with random passwords. This is called brute force attack.

---

> [!WARNING]
> First make sure you configured and testet ssh login using ssh-key works, then follow this steps:

```bash
# Connect to your v-server using your provided username and password:
ssh <user>@<remoteserver>

# Open the sshd configuration file to write to:
sudo nano /etc/ssh/sshd_config

# Search for "PasswordAuthentication"
# To disable tunneled clear text passwords, change to no here!
PasswordAuthentication no
# Save the changes and close the editor

# Restart ssh service:
sudo systemctl restart ssh

# To logout from your v-server just run in terminal:
logout

# Try reconnect via password:
ssh -o PubkeyAuthentication=no <username>@<remoteserver>

# For successful deactivated PasswordAuthentication the output should look something like this:
<user>@<remoteserver>: Permission denied (publickey).

```

### Setup a SSH-Config for more identities on your host machine, for an easer ssh-login

```powershell
# Navigate to ssh folder on your host machine
cd ~/.ssh

# There should be a file named "config". If there is none, create one.
# Append it this content:
Host <remoteserver>
    User <user>
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/<ssh_key_name>
# Save the changes.

# With this configuration you can run this command to login to your v-server:
ssh <remoteserver>
```

> [!NOTE]
> "Bad owner or permissions" error may happen on Windows when you run `ssh <remoteserver>` . To Fix it follow this steps:

```powershell
# Navigate to the .ssh directory:
cd <path_to>\.ssh

# Check the current permissions of your <ssh_config_file>:
Get-Acl .\<ssh_config_file> | Format-List

# Remove the problematic permissions:
$acl = Get-Acl .\<ssh_config_file>
    $acl.Access | Where-Object { $_.IdentityReference -match "Authenticated Users" } | ForEach-Object {
        $acl.RemoveAccessRule($_)
    }
    Set-Acl .\<ssh_config_file> $acl

# Ensure only the user has permissions:
icacls .\<shh_config_file> /inheritance:r
icacls .\<shh_config_file> /grant:r "$($env:USERNAME):(F)"
```

---

## Nginx Alternative Web Page

### Log into your v-server, and first update the packages by running this command

```bash
# A prompt may pop asking you to select some services that need to be restarted after the update.
# Just selected some and moved on.
sudo apt update
```

### After the update install nginx by running following commands

```bash
# The "-y" means to answer automatically "yes" on all input prompts that come along installation process.
sudo apt install nginx -y

#  After installation finishes you can run this command to check if nginx is up and running. :
sudo systemctl status nginx

# If nginx is up and running, you should now have a web server that anyone can access via web browser just by given your external IP address in there browser.
# For now nginx will show a default page.
# This default page can be found in "/var/www/html/" and by default is usually named "index.nginx-debian.html".
# And the default nginx configuration file that serves this "html" via browser connection is by default in "/etc/nginx/sites-enabled/default".
# This file also contains interesting information about nginx that you may wanna check it out.
```

>[!TIP]
> You can generally use ```sudo systemctl <on_service_operation> <service_name>``` command to check the status with ```status``` , stop and start with ```restart```, stop with ```stop``` or start with ```start``` a service on your v-server.

### Add your own web page

```bash
# Created a directory were your web page file that you wanna server via web server should be. Example:
sudo mkdir /var/www/alternatives
# With ```mkdir``` you can create directories in your server.

# Then create the file itself. Example:
sudo touch /var/www/alternatives/alternate-index.html
# With ```touch``` you can create files in your server.

# Open the file to write it content. Example:
sudo nano /var/www/alternatives/alternate-index.html
```

```html
<!-- Then give it this content: -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Alternative</title>
</head>
<body>
    <h1>Hello, Nginx!</h1>
    <p>I just configured our Nginx web server on Ubuntu Server!</p>
</body>
</html>
```

### Add your own nginx configuration

```bash
# Create your own nginx configuration file. Example:
sudo touch /etc/ngninx/sites-enabled/alternatives
```

```nginx
# Then give it this content:
server {
    # This directive tells Nginx to listen on port <port> for IPv4 connections.
    listen <port>;

    # This directive tells Nginx to listen on port <port> for IPv6 connections.
    listen [::]:<port>;

    # This is your created folder, all files inside this folder will be served via nginx
    root /var/www/alternatives;

    # This is your created file, a default file to be server via nginx if no files was requested
    index alternate-index.html;

    # Basically gives a Not Found Response to requester if no such requested file has been found
    location / {
        try_files $uri $uri/ =404;
    }
}
```

 ```bash
 # Checked your nginx configuration for errors by running:
 nginx -t

 # Last, restarte nginx service.
 sudo systemctl restart nginx
 ```

Now if you access your `<remoteserver>` via the browser, you will not see your own page, but the default page of nginx. To see your page you have to access your `<remoteserver>:<port>` via the browser. Port is what you gave it in your nginx configuration.

---

## Git configuration

### Install and configure git on your v-server

```bash
sudo apt update
sudo apt install git

# After you installed git you need to configure it. That means only setting the git's global configuration user:
git config --global user.name "<your_name>"
git config --global user.email "<your_email>"
```

### Generate SSH-Key Pairs for Github

Basically almost the same as you did in [SSH Login](#ssh-login), but following Github Documentation.

```bash
# Generate SSH-Key Pairs, but this time on your v-server:
ssh-keygen -t ed25519 -C "<your_email>"
# >[!Note]: here give your Github Account email for <your_email>

# Starte the ssh-agent in the background:
eval "$(ssh-agent -s)"
    Agent pid 20664

#  Add your SSH private key to the ssh-agent:
ssh-add ~/.ssh/<ssh_key_file_name>
    Identity added: /path_to/.ssh/<ssh_key_file_name> (your_email)

# Copy the SSH public key to your clipboard.
cat ~/.ssh/<ssh_key_file_name>.pub
```

### Paste the SSH Public Key into your Github Account settings

- In the upper-right corner of any page on GitHub, click your profile photo, then click Settings.
- In the "Access" section of the sidebar, click  SSH and GPG keys.
Click New SSH key or Add SSH key.
- In the "Title" field, add a descriptive label for the new key. For example, if you're using a personal laptop, you might call this key "Personal laptop".
- Select the type of key, either authentication or signing. For more information about commit signing, see "About commit signature verification."
- In the "Key" field, paste your public key.
Click Add SSH key.
