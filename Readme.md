# V-Server Setup

This is a documentation of setting up ssh login using SSH-Key Pairs, configure Nginx web server to access an alternative Web Page and configure Git on V-Server.

The setup was performed on host machine Windows 11 using Powershell Terminal and remote machine Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-105-generic x86_64).

Along reading this Documentation, you will come across many examples that contain wrapped expressions like for example `<some_expressions>`, that means the content wrapped, in this case `<same_expressions>` should be replaced with your own content.

---

## Table of Content

1. [SSH Login](#ssh-login)
1. [Nginx Alternative Web Page](#nginx-alternative-web-page)
1. [Git Configuration](#git-configuration)

---

## SSH Login

1. On host machine i generated a ssh-key pairs. On prompt for input i give no Passphrase and named the key files:

    ```powershell
    ssh-keygen -t ed25519
    ```

1. I added the public ssh-key to my v-server using the Powershell command:

    ```powershell
    Get-Content $env:USERPROFILE\.ssh\<ssh_key_name>.pub | ssh <user>@<remoteserver> "cat >> .ssh/authorized_keys"
    ```

1. I deactivated the Password-Login to my v-server following this steps:

    1. Connect to v-server via Password.
    1. Run ```sudo nano /etc/ssh/sshd_config``` to open the file to write to.
    1. Set `PasswordAuthentication` to `no`, save and closed the file.
    1. Restart `ssh` service using ```sudo systemctl restart ssh```.
    1. Reconnect via ssh-key with `ssh -i ~/.ssh/<ssh_key> <user>@<remoteserver>`
    1. Try reconnect via password with `ssh -o PubkeyAuthentication=no <username>@<remoteserver>`

1. I setup a SSH-Config for more identities on my host machine.

    ```ssh.config
    Host <remoteserver>
    User <user>
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/<ssh_key_name>
    ```

    With this configuration now i just have to run ```ssh <remoteserver>``` and i log into my v-server.

    You may get Bad owner or permissions when you run. To Fix it follow this steps on Windows like i did:

    1. Navigate to the .ssh directory:

        Usually in your Windows User's folder:

        ```powershell
        cd <path_to>\.ssh
        ```

    1. Check the current permissions of your <ssh_config_file>:

        ```powershell
        Get-Acl .\<ssh_config_file> | Format-List
        ```

    1. Remove the problematic permissions:

        ```powershell
        $acl = Get-Acl .\<ssh_config_file>
        $acl.Access | Where-Object { $_.IdentityReference -match "Authenticated Users" } | ForEach-Object {
            $acl.RemoveAccessRule($_)
        }
        Set-Acl .\<ssh_config_file> $acl
        ```

    1. Ensure only the user has permissions:

        ```powershell
        icacls .\<shh_config_file> /inheritance:r
        icacls .\<shh_config_file> /grant:r "$($env:USERNAME):(F)"
        ```

---

## Nginx Alternative Web Page

1. I logged into my v-server, and first i updated the packages by running ```sudo apt update```

    In this case a prompt may pop asking you to select some services that need to be restarted after the update. I just selected some and moved on.

1. After the update i installed nginx by running ```sudo apt install nginx -y```.

    The ```-y``` means to answer automatically "yes" on all input prompts that come along installation process.

    After installation finishes you can run ```sudo systemctl status nginx``` to check if nginx is up and running.

    Note here, you can generally use ```sudo systemctl <on_service_operation> <service_name>``` command to check the status with ```status``` , stop and start with ```restart```, stop with ```stop``` or start with ```start``` a service on your v-server.

    If nginx is up and running, you should now have a web server that anyone can access via web browser just by given your external IP address in there browser. For now nginx will show them a default page.

    This default page can be found in ```/var/www/html/```, by ist default name usually called ```index.nginx-debian.html```. And the default nginx configuration file that serves this ```html``` via browser connection is by default in ```/etc/nginx/sites-enabled/default```. This file also contains interesting information about nginx that you may wanna check it out. 🙂

1. Now I added my own nginx configuration and web page.

    1. I created a directory were my web page file that I wanna server via web server should be. I called my ```sudo mkdir /var/www/alternatives```.
        With ```mkdir``` you can create directories in your server. 😉
    1. Then I created the file itself. I called it ```sudo touch /var/www/alternatives/alternate-index.html```
        With ```touch``` you can create files in your server. 😃
    1. Then I gave it this content:

        ```alternative-index.html
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

    1. Now the most important step is to create my own nginx configuration file. I called it ```sudo touch /etc/ngninx/sites-enabled/alternatives```.
    1. Then I gave it this content:

        ```nginx.conf
        server {
            # this directive tells Nginx to listen on port <port> for IPv4 connections.
            listen <port>;
            # This directive tells Nginx to listen on port <port> for IPv6 connections.
            listen [::]:<port>;

            # that is my created folder, all files inside this folder will be served via nginx
            root /var/www/alternatives;
            # that is my created file, a default file to be server via nginx if no files was requested
            index alternate-index.html;

            # basically gives a Not Found Response to requester if no such requested file has been found
            location / {
                try_files $uri $uri/ =404;
            }
        }
        ```

        So ease you can set up a web server. 😊
    1. Last, I restarted my nginx service. ```sudo systemctl restart nginx```. And checked my nginx configuration for errors by running ```nginx -t``` bevor restarting.

Now if I access my `<remoteserver>` via the browser, I will not see my own page, but the default page of nginx. To see my page I have to access my `<remoteserver>:<port>` via the browser. Port is what I gave it in my nginx configuration.

---

## Git configuration

1. Install git on my v-server by running following commands:

    ```bash
    sudo apt update
    sudo apt install git
    ```

    But why always update? 🧐

    I did some asking around and this is what i get:
    ```apt update updates the package lists. If you don't do it before an installation, apt might complain that it cannot find the package in the repository, because it computed the URL based on an old version of the list (which listed an older version of the package).```

    So is not mandatory, but there is no reason why not to do it. 🫡
1. After I installed git I needed to configure it. By that I mean only setting the git's global configuration user:

    For that I run this commands:

    ```bash
    git config --global user.name "<your_name>"
    git config --global user.email "<your_email>"
    ```

1. Now I Generate SSH-Key Pairs for Github:

    Basically almost the same as I did in [SSH Login](#ssh-login), but following Github Documentation.

    1. I generated SSH-Key Pairs, but this time on my v-server:

        ```bash
        ssh-keygen -t ed25519 -C "<your_email>"
        ```

        Note: here I gave my Github Account email for <your_email>

        ---

        -t ed25519:

        The -t option specifies the type of key to create. In this case, ed25519 indicates that the key type is Ed25519, a modern and secure public-key signature system. Ed25519 is known for its performance and security benefits.

        ---

        -C "your_email":

        The -C option provides a comment that will be included in the public key for identification purposes. It's often used to add an email address or other identifying information to the key, which can help in managing multiple keys. In this example, the comment is "your_email".

        ---

    1. I started the ssh-agent in the background

        ```bash
        eval "$(ssh-agent -s)"
        Agent pid 20664
        ```

    1. I added my SSH private key to the ssh-agent.

        ```bash
        ssh-add ~/.ssh/<ssh_key_file_name>
        Identity added: /path_to/.ssh/<ssh_key_file_name> (your_email)
        ```

    1. I copied the SSH public key to my clipboard.

        ```bash
        cat ~/.ssh/<ssh_key_file_name>.pub
        ```

    1. Now I paste the SSH Public Key into my Github Account settings.

        In the upper-right corner of any page on GitHub, click your profile photo, then click Settings.

        In the "Access" section of the sidebar, click  SSH and GPG keys.

        Click New SSH key or Add SSH key.

        In the "Title" field, add a descriptive label for the new key. For example, if you're using a personal laptop, you might call this key "Personal laptop".

        Select the type of key, either authentication or signing. For more information about commit signing, see "About commit signature verification."

        In the "Key" field, paste your public key.

        Click Add SSH key.

---
