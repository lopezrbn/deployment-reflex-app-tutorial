# Tutorial on how to deploy a Reflex web app into a Ubuntu VPS

<br>



## Introduction
This tutorial explains how to deploy a [Reflex](https://reflex.dev/) web app into a Virtual Private Server (VPS) configured with Linux to which you have access through SSH terminal. In this case, the Ubuntu 23.10 distribution is used.

<br>



## Table of Contents
1. Cloning of the GitHub repository and installation of the virtual environment
2. Configuration of rxconfig.py file
3. Installation and configuration of NGINX as a reverse proxy with WebSocket
4. SSL certificate for HTTPS
5. Reflex run
6. Configuration of Reflex app as a system service

<br>



## 1. Cloning of the GitHub repository and installation of virtual environment and requirements
Firstly, choose a repository containing a Reflex web app and copy its link. For this tutorial, I will be using my own [car-price-checker](https://github.com/lopezrbn/car-price-checker) repository, a project that I developed to check the value of a car in the used car market. Just substitute any reference to this project name below with yours. 

Click the green `<> Code` button and copy the link under the `HTTPS` option.

![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/9fa113c0-075a-4ba0-870c-cb5c0b565c3c)

Navigate to the directory in which you will be creating the project and:
   ```
   git clone https://github.com/lopezrbn/car-price-checker.git
   ```

If everything is correct, you will see some messages like this:

![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/d73923fe-d8de-4933-9739-dfa3105f3b10)

Now navigate inside the project folder and create a virtual environment under the name `venv`:
   ```
   cd car-price-checker/
   ```
   ```
   python3 -m venv venv
   ```

And activate the virtual environment:
```
source venv/bin/activate
```

Finally, installation of requirements and deactivation of the virtual environment:
```
pip install -r requirements.txt
```
```
deactivate
```

<br>



## 2. Configuration of rxconfig.py file
The `rxconfig.py` file tells Reflex how to build your app. Here we will modify four parameters: `front_port`, `back_port`, `your_vps_domain_or_ip` and `your_hostname`.

- `front_port` defines the port in which the frontend will be running. I have configured it under port `3002` but Reflex uses by default port `3000`. Use any port you prefer under the range of 3000's.
- `back_port` defines the port in which the backend will be running. I have configured it under port `8002` but Reflex uses by default port `8000`. Use any port you prefer under the range of 8000's.
- Change `you_vps_domain_or_ip` for the domain in which you want to serve your app in case you have one , or in the other case, just write the IP of your VPS. Use always `http://` before the domain or the IP, or `https://` in case you are using a domain and want to configure it as secure (we will show how to do it in the following sections).
- `your_hostname` controls if your Reflex app is being run on your VPS (production mode) or in any other machine (usually in a testing environment). Change this parameter for your VPS hostname. To know your VPS hostname, just type:
```
hostname
```

So finally open the file under `nano` to edit and change the parameters according to explained above:
```
sudo nano rxconfig.py
```

When you have finished, `Ctrl` + `X` to close, then press `y` and `Enter` to save the changes.

<br>



## 3. Installation and configuration of NGINX as a reverse proxy web server with WebSocket

The next step is to install and configure a web server to handle the client petitions to serve your Reflex app. We will be using NGINX as it is one of the most adopted by the community.

I have personally found this [guide](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04) useful, so I will replicate it here.

Firstly, install NGINX:
```
sudo apt install nginx
```

Then check if the server is active and running. If it is, you should see something as shown below.

```
systemctl status nginx.service
```

![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/d3bbeb6a-bf43-42d5-9676-3da44ce04ef1)

And, if you navigate to your VPS IP, should see the NGINX welcome page.

![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/8f41ae56-5899-407f-b137-785dbfb006dc)

So once NGINX is installed, the next step is to configure how to redirect the client's petitions to the ports in which Reflex is running.

For this, go to the `sites-available` folder of NGINX and create a new configuration file under the name `vps.confg`:
```
cd /etc/nginx/sites-available
```
```
sudo nano vps.confg
```

And copy the following code:
```
server {
  listen 80;
  server_name your_vps_ip_or_domain;

  location / {
    proxy_pass http://localhost:3002;
  }

  location /_event {
    proxy_pass http://localhost:8002;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;
  }
}
```

We are telling NGNIX to create a virtual server under the block `server`, which will be listening to `port 80` (default one for http petitions) for any petition coming to `your_vps_ip_or_domain`. Of course, change `your_vps_ip_or_domain` for the IP of your VPS in case you are not using a domain, and the domain name otherwise. Use the format `http://XXX.XXX.XXX.XXX` for the IP and `http://domain_name` for the domain (or `https://domain_name`).

Then, we are opening two `location` blocks.

In the first one, `location /` will serve any petition made to the route `/` of your Reflex app. This is usually, your index page. Here we are telling NGINX to reverse proxy this petition to `http://localhost:3002`, which is where Reflex is running your frontend. Please, change the port `3002` to whatever you have configured above in the section [2. Configuration of rxconfig.py file](#2.-configuration-of-rxconfig.py-file).

In the second one, `location /_event` is the reserved route by Reflex for backend petitions. Again, we are redirecting these petitions to `http://localhost:8002`, which is where Reflex is running the backend. And of course, change the port `8002` to whatever you have configured above in the section [2. Configuration of rxconfig.py file](#2.-configuration-of-rxconfig.py-file).

Here we are also adding the directives `proxy_http_version 1.1`, `proxy_set_header Upgrade $http_upgrade`, `proxy_set_header Connection "Upgrade"` and `proxy_set_header Host $host` to tell NGINX to upgrade the connection to a WebSocket so the front and the back can communicate properly. More information about configuring NGINX as a WebSocket proxy in the official [tutorial](https://www.nginx.com/blog/websocket-nginx/).

Finally, `Ctrl` + `X` to close, and press `y` and `Enter` to save the changes. Then, we create a symbolic link to the file to the `sites-enabled` folder of NGINX:
```
ln -s vps.confg /etc/nginx/sites-enabled/vps.confg
```

After this, the last step is resetting NGINX:
```
sudo systemctl restart nginx
```

In the case there is some mistake in the configuration, you will see a message as shown below. Just open the file again and check everything is written as in the code snippet above.

![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/6284afa4-cf55-4d65-82d9-2cff4503f32a)

In case you do not find any error message, you can check the status of NGINX with:
```
systemctl status nginx.service
```
If everything is working fine, you should see something as:

![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/ade90a4a-572e-4cc8-969f-c7584a0be1e8)

<br>



## 4. SSL certificate for HTTPS

In case you are using a domain to access your web app (which needs to be redirected to your VPS IP), you can choose to upgrade your app to use `https` instead of `http`.

For this, we will be using [certbot](https://certbot.eff.org/) to automatically handle the process. Just navigate to the webpage and go to the bottom where you will find the statement `My HTTP website is running Software on System`. Use the selection buttons and choose `Nginx` and `Ubuntu 20` as in the image below:

![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/3ad69437-f6b9-461a-91ae-4fe13e0eff21)

Now you just need to follow the steps shown on the web page, that I will reproduce here.

1. SSH into the server: you already are logged in to the server if you have followed this tutorial up to here, so omit this step.
2. Install snapd: you can skip this step as snapd is already installed on Ubuntu 18 or higher distributions.
3. Remove certbot-auto just in case:
   ```
   sudo apt-get remove certbot
   ```
4. Install Certbot
   ```
   sudo snap install --classic certbot
   ```
5. Prepare the Certbot command
   ```
   sudo ln -s /snap/bin/certbot /usr/bin/certbot
   ```
6. Choose the mode _get and install your certificates_
   ```
   sudo certbot --nginx
   ```
7. Test automatic renewal
   ```
   sudo certbot renew --dry-run
   ```

If at this point, you have followed all the steps with no error, Certbot will have automatically updated your NGINX configuration file, adding the needed directives to use HTTPS on your site.

<br>



## 5. Reflex run

Now NGINX is already configured to serve your Reflex app so you only need to run Reflex for the app to be available in your VPS.

For this, first navigate to the route of your Reflex app and activate again the virtual environment:
```
cd /your/Reflex/app/path
```
```
source venv/bin/activate
```

We will tell Reflex to run the app in production mode so it creates an optimized build of it which is much faster to be served:
```
reflex run --env prod
```

If everything was fine, you should see something like this:

![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/df636648-a0c9-4df7-b3ab-1f964d865c11)

And as the message says, your app is already running at `http://localhost:3002` (remember we have configured NGINX to redirect any petition from your browser when you enter your VPS IP or domain to this port).

Now you can go to your browser and navigate to `http://your_vps_ip_or_domain` to check the app is working properly.

<br>



## 6. Configuration of Reflex app as a system service

If you have come to this point, you already have your Reflex app running hosted in your VPS. However, what happens when you close your SSH terminal?

Well, what happens is that once we close the terminal, the Reflex process is killed and your app stops working, which of course is not the desired behaviour. So we need a way of running Reflex as a background process to not be killed when we close our terminal.

The usual form of doing this is running Reflex, or any other process, in the background adding `&` at the end of the command `reflex run --env prod &`. However, although this will effectively execute Reflex in the background, it will not work either due to Ubuntu sending a killing signal to the background processes when the SSH terminal is closed.

So the best way to solve this problem is to add the Reflex process as a system service to be executed not only in the background but automatically every time the server restarts.

Hence, we go first to the Ubuntu system folder:

```
cd /etc/systemd/system
```

Then we create a new configuration file, using `<your_service_name>.service` as the name for which we will control the service later. I use the formula `reflex-bg-<name-of-the-app>.service`, so then is always easy to remember how the service was called. In the `car-checker-prices` project, the service would name `reflex-bg-car-price-checker.service`:

```
sudo nano <your_service_name>.service
```

And copy the following code inside it:

```
[Unit]
Description=<your_app_service_description>
After=network.target

[Service]
User=<your_vps_user>
Type=simple
WorkingDirectory=<your_app_path>
Environment="PATH=<your_app_path>:/usr/bin"
ExecStartPre=<your_app_path>/venv/bin/python3 -m reflex init
ExecStart=<your_app_path>/venv/bin/python3 -m reflex run --env prod
#ExecStart=echo $PATH
RemainAfterExit=yes
TimeoutSec=0
#Restart=always

[Install]
WantedBy=multi-user.target
```

Of course, now you should change the following fields:
- `<your_app_service_description>`: this will be the description of the service you will see when you call `systemctl status <your_service_name>.service`. I use the formula `<your_app_name> web app background Reflex service`. So in this project, it would be `Car price checker web app background Reflex service.`. But of course, use whatever works for you.
- `<your_vps_user>`: this is the user you log in with to enter the VPS. In my case, it is `ubuntu`, which is a common name for Ubuntu servers.
- `<your_app_path>`: this is the path of the folder in which you have cloned the app from the GitHub repository. If you have doubts about the full path, navigate to it and use `pwd` to obtain the exact path. Be careful and check the script twice as you will need to write `<your_app_path>` for four times in the file.

As always, `Ctrl` + `X` to close, and press `y` and `Enter` to save the changes.

And once the file is written, we just need to mount and initialize it.

Reset the daemons:
```
sudo systemctl daemon-reload
```

Enable the service:
```
sudo systemctl enable <your_service_name>.service
```

If it works, you should see a message saying that a symbolic link to the file has been created.

Then we start the service:
```
sudo systemctl start <your_service_name>.service
```

Finally, we check the service is running with no errors:
```
systemctl status <your_service_name>.service
```

If the service works well, you will see something like the following:

![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/bc78410a-2866-4ec1-b987-87bb1abd17cc)


And that is everything. Now you have a Reflex web app running in the background as a system service hosted in your VPS!
