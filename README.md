# Jupyter lab notes

Deploying Jupyter lab, sample docs, ...

## Deploying in AWS Cheatsheet

Deploying Jupyter Lab in AWS is similar to deploying locally, with these extra steps:
- Deploy a free-tier AWS EC2 Ubuntu instance
- Serve via HTTPS
- Set a password
- Skip browser launch
- Create a self-signed SSL cert

I will not write details unless the step seems unusual

### Free-tier EC2 instance

Deploy an EC2 instance, I am using a free-tier machine, you might need something larger depending on your data and computations. These notes are Ubuntu-specific, but you should be able to figure out the differences if you need to use a different operating system.
- Create an EC2 Ubuntu instance using a free-tier machine type
- One of the early steps will be labeled **Key pair (login)**. I always use a **key pair**. In the `ssh` step below, the key pair is used to authenticate to the system. I also store that key pair in my password manager.
- Create a security group inbound rule. I don't like using default ports, so I switched to `8942`. Here is my inbound rule:
  - Protocol: TCP
  - Port Range: 8942
  - Source: 0.0.0.0/0
  - Description: TCP port 8942 (non-standard, used for Jupyter Lab)
- Accept the defaults for the rest of the steps

### Connect to the instance

I use ssh from my workstation, using the PEM cert generated during instance creation:

```bash
ssh -i ~/.ssh/JupyterEC2.pem ubuntu@ec2-3-139-59-249.us-east-2.compute.amazonaws.com
```

### SSL cert

Generate a cert for the Jupyter Lab server, and set the ownership
```bash
cd
sudo openssl req -x509 -nodes -days 365 \
-newkey rsa:2048 \
-keyout ssl_cert.pem \
-out ssl_cert.pem

sudo chown $USER:$USER ssl_cert.pem
```
### Set up the environment



### Quarto

You may not need Quarto, I am using it to generate HTML from my Notebook (`.ipynb`) files. If you don't plan to use Quarto, then skip this.

```bash
mkdir quarto
cd quarto

# Check the Quarto site for the version
wget https://github.com/quarto-dev/quarto-cli/releases/download/v1.5.57/quarto-1.5.57-linux-amd64.deb

sudo apt install ./quarto-1.5.57-linux-amd64.deb
rm ./quarto-1.5.57-linux-amd64.deb
```

### Make sure that Python 3 is available

```bash
python3 --version
```

### Install Jupyter Lab

Use a Python virtual environment (`venv`). There is a link at the bottom of this page about the merits of virtual environments.

```bash
mkdir JupyterLab
cd JupyterLab
python3 -m venv .venv
source .venv/bin/activate
```

Install Node.js and Node Version Manager

```bash
sudo apt install nodejs python3.12-venv
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install 21
nvm use 21
node -v > .nvmrc
```
> Note:
>
> If you use this instance for anything else, then you might want to have the Node.js version automatically set based on the current working directory. See the [`nvm-sh`](https://github.com/nvm-sh/nvm?tab=readme-ov-file#bash) GitHub repository for details.

Finally, install Jupyter Lab:

```bash
python3 -m pip install jupyter jupyterlab matplotlib plotly
```

#### Create a Jupyter Lab configuration

```bash
mkdir -p /home/ubuntu/.jupyter/

cat << EOF > /home/ubuntu/.jupyter/jupyter_lab_config.py
c = get_config()  #noqa
# listen on all interfaces
c.ServerApp.ip = '0.0.0.0'
# use non-standard port
c.ServerApp.port = 8942
# don't try to open a browser
c.ServerApp.open_browser = False
# SSL
c.ServerApp.certfile = u'/home/ubuntu/ssl_cert.pem'
# hash of password
# The password hash does NOT go in here, running `jupyter lab password` will generate a separate JSON file
EOF
```

#### Generate a password for Jupyter Lab

```bash
jupyter lab password
```

#### Start Jupyter Lab

Grab a sample notebook to test with

```bash
wget https://quarto.org/docs/get-started/hello/_hello.ipynb
mv _hello.ipynb hello.ipynb
```

```bash
cd /home/ubuntu/quarto/quarto-example # Replace with your path
source .venv/bin/activate
python3 -m jupyter lab hello.ipynb
```

When you start Jupyter Lab you will see logging in the terminal. One thing that you are looking for is the endpoint where the server is running. Look for:

- `https`
- Some kind of hostname, in this example it is `ip-172-31-38-116`
- The port number that you used in the security group inbound rule (`8942` in my case)

```bash
[I 2024-09-20 14:25:47.955 ServerApp] Jupyter Server 2.14.2 is running at:
[I 2024-09-20 14:25:47.955 ServerApp] https://ip-172-31-38-116:8942/lab
[I 2024-09-20 14:25:47.955 ServerApp]     https://127.0.0.1:8942/lab
[I 2024-09-20 14:25:47.955 ServerApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
```
### Start Jupyter Lab at boot

Running Jupyter Lab from an ssh session is fine as long as the ssh session runs. Running Jupyter Lab from a service removes the dependency on an active ssh session, and starts the service at server boot.

#### Create a service file

```bash
sudo cat << EOF > /etc/systemd/system/jupyterlab.service
[Unit]
Description=Jupyter Lab

[Service]
Type=simple
PIDFile=/run/jupyter.pid
ExecStart=/bin/bash -i -c "source .venv/bin/activate;python3 -m jupyter lab"
User=ubuntu
WorkingDirectory=/home/ubuntu/quarto/quarto-example
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

#### Enable and start the service

```bash
systemctl enable jupyterlab.service
systemctl daemon-reload
systemctl status jupyterlab.service
systemctl start jupyterlab.service
systemctl status jupyterlab.service
```

#### Browse

[Jupyter Lab](https://ec2-3-139-59-249.us-east-2.compute.amazonaws.com:8942/lab)
