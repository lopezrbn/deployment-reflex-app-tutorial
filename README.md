# deployment-reflex-app-tutorial
Tutorial on how to deploy a Reflex app into a Ubuntu VPS


## Introduction
This tutorial explains how to deploy a [Reflex](https://reflex.dev/) web app into a Virtual Private Server (VPS) configured with Linux to which you have access through SSH. In this case, the Ubuntu 23.10 distribution is used.


## Table of Contents
1. Cloning of the GitHub repository and installation of the virtual environment
2. Configuration of rxconfig.py file
3. Installation and configuration of NGINX as a reverse proxy with Websockets
4. SSL certificate for HTTPS
5. Configuration of Reflex app as a system service


## 1. Cloning of the GitHub repository and installation of virtual environment and requirements
Firstly, choose a repository containing a Reflex web app and copy its link. For this tutorial, I will be using my own [car-price-checker](https://github.com/lopezrbn/car-price-checker) repository, a project that I developed the check the value of a car in the used car market. Just substitute any reference to this project name below with yours. 

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
> Linux/MAC:   
```
source venv/bin/activate
```
> Windows:
```
.\venv\Scripts/activate
```

Finally, installation of requirements and deactivation of the virtual environment:
```
pip install -r requirements.txt
```
```
deactivate
```



## 2. Configuration of rxconfig.py file
The `rxconfig.py` file tells Reflex how to build your app. Here we will modify four parameters: `front_port`, `back_port`, `your_vps_domain_or_ip` and `your_hostname`.

- `front_port` defines the port in which the frontend will be running. I have configured it under port `3002` but Reflex uses by default port `3000`. Use any port you prefer under the range of 3000's.
- `back_port` defines the port in which the backend will be running. I have configured it under port `8002` but Reflex uses by default port `8000`. Use any port you prefer under the range of 8000's.
- Change `you_vps_domain_or_ip` for the domain in which you want to serve your app in case you have one, or in the other case, just write the IP of your VPS. Use always `http://` before the domain or the IP, or `https://` in case you are using a domain and want to configure it as secure (we will show how to do it in the following sections).
- `your_hostname` controls if your Reflex app is being run on your VPS (production mode) or in any other machine (usually in a testing environment). Change this parameter for your VPS hostname. To know your VPS hostname, just type:
```
hostname
```

So finally open the file under `nano` to edit and change the parameters according to explained above:
```
sudo nano rxconfig.py
```

When you have finished, `Ctrl` + `X` to close, then press `y` and `Enter` to save the changes.



## 3. Installation and configuration of NGINX as a reverse proxy web server with Websockets

The next step is to install and configure a web server to handle the client petitions to serve your Reflex app. We will be using NGINX as it is one of the most adopted by the community.

I have personally found this [guide](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04) useful, so I will replicate it here.

Firstly, install NGINX:
```
sudo apt install nginx
```

Then check if the server is active and running. If it is, you should see something as shown below.
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


## 4. SSL certificate for HTTPS

Under construction.


## 5. Reflex run

At this point, NGINX is already configured to serve your Reflex app so you only need to run Reflex for the app to be available in your VPS.

For this, first navigate to the route of your Reflex app and activate again the virtual environment:
```
cd /your/Reflex/app/path
```
> Linux/MAC:   
```
source venv/bin/activate
```
> Windows:
```
.\venv\Scripts/activate
```

We will tell Reflex to run the app in production mode so it creates an optimized build of it which is much faster to be served:
```
reflex run --env prod
```

If everything was fine, you should see something like this:
![image](https://github.com/lopezrbn/deployment-reflex-app-tutorial/assets/113603061/df636648-a0c9-4df7-b3ab-1f964d865c11)

And as the message says, your app is already running at `http://localhost:3002`.

Now you can go to your browser and navigate to `http://your_vps_ip_or_domain` to check the app is working properly.


## 6. Configuration of Reflex app as a system service

Under construction.
