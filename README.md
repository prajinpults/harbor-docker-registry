# Harbor Docker Registry

Harbor is an open source docker registry to keep docker images.


## Download and Install Harbor Docker Registry



We are using harbor for keeping our docker image all commands are running in DOCKER REGISTRY

| Role                | IP           | FQDN              | OS        | RAM | CPU |
|---------------------|--------------|-------------------|-----------|-----|-----|
| DOCKER REGISTRY     | 172.24.10.17 | dockerregistry.in | Ubuntu 18 | 16G | 4   |


```shell
sudo su
```
set hostname properly if not


```shell
nano /etc/hostname
```


```shell
dockerregistry.in
```


```shell
cat >>/etc/hosts<<EOF
#added
172.24.10.23 kubernetesmaster2.in
172.24.10.24 kubernetesmaster3.in
172.24.10.25 kubernetesmaster1.in

EOF
```


Harbor is deployed as several Docker containers. We can therefore deploy it on any Linux distribution that supports Docker. The target host requires Docker, and Docker Compose to be installed.

### Hardware

The following table lists the minimum and recommended hardware configurations for deploying Harbor.

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU      | 2 CPU   | 4 CPU       |
| Mem      | 4 GB    | 8 GB        |
| Disk     | 40 GB   | 160 GB      |

### Software

The following table lists the software versions that must be installed on the target host.

| Software       | Version                       | Description                                                                                                                                   |
|----------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| Docker engine  | Version 17.06.0-ce+ or higher | For installation instructions, see [Docker Engine Install](https://git.ults.in/kla-k8s/k8s-ha-cluster#install-docker-engine-if-not-installed) |
| Docker Compose | Version 1.18.0 or higher      | For installation instructions, see [Docker Compose Install](#install-docker-compose-if-not-installed)                                         |
| Openssl        | Latest is preferred           | Used to generate certificate and keys for Harbor                                                                                              |

#### Install docker Compose if not installed

```shell
curl -SL https://github.com/docker/compose/releases/download/v2.11.2/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
```
Apply executable permissions to the standalone binary in the target path for the installation.

```shell
chmod +x docker-compose
```
Test and execute compose commands using docker-compose.

If the command docker-compose fails after installation, check our path.
We can also create a symbolic link to /usr/bin or any other directory in our path. For example:

```shell
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```


### Network ports

Harbor requires that the following ports be open on the target host.

| Port | Protocol | Description                                                                                                                                         |
|------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| 443  | HTTPS    | Harbor portal and core API accept HTTPS requests on this port. We can change this port in the configuration file.                                  |
| 4443 | HTTPS    | Connections to the Docker Content Trust service for Harbor. Only required if Notary is enabled. We can change this port in the configuration file. |
| 80   | HTTP     | Harbor portal and core API accept HTTP requests on this port. We can change this port in the configuration file.                                   |



### Download the Harbor Installer

create a directory for keeping the installation file and config.


```shell
mkdir harbor
cd harbor
```

skip this if no internet connection file is already there.

download the Harbor installers from the [official releases page.](https://github.com/goharbor/harbor/releases)
Download either the offline installer using curl.


```shell
curl -L https://github.com/goharbor/harbor/releases/download/v1.10.14/harbor-offline-installer-v1.10.14.tgz \
 > harbor-offline-installer-v1.10.14.tgz
```

download the corresponding *.asc file to verify that the package is genuine.

```shell
curl -L https://github.com/goharbor/harbor/releases/download/v1.10.14/harbor-offline-installer-v1.10.14.tgz.asc \
 > harbor-offline-installer-v1.10.14.tgz.asc
```

Obtain the public key for the *.asc file.
```shell
gpg --keyserver hkps://keyserver.ubuntu.com --receive-keys 644FF454C0B4115C
```

verify file is genuine
```shell
gpg -v --keyserver hkps://keyserver.ubuntu.com --verify harbor-offline-installer-v1.10.14.tgz.asc
```

### Installing Harbor
Use tar to extract the installer package:
```shell
tar xzvf harbor-offline-installer-v1.10.14.tgz
```


### Configure HTTPS Access to Harbor


#### Generate a Certificate Authority Certificate


To generate a CA certficate, run the following commands in /home/kla/cert directory.

1. Generate a CA certificate private key.

    ```sh
    cd /home/kla/cert
    openssl genrsa -out ca.key 4096
    ```

2. Generate the CA certificate.

   also possible to use existing certificate, *it will expire in 10 years
    ```sh
   openssl req -x509 -new -nodes -sha512 -days 3650 \
   -subj "/C=IN/ST=Kerala/L=Kerala/O=Kerala Legislative Assembly/OU=E-Niyamasabha/emailAddress=devops@kla.in/CN=dockerregistry.in" \
   -key ca.key \
   -out ca.crt
    ```
   Please keep a copy in this repository [here](./ca.crt)
   
3. copy this to kubernetes master1 for adding to kubernetes 
   
   ```shell
   ssh kla@kubernetesmaster1.in "mkdir -p /home/kla/cert/"
   scp /home/kla/cert/ca.crt kla@kubernetesmaster1.in:/home/kla/cert/ca-docker-reg$(date +%d-%m-%Y_%H:%M).crt
   ```

   
   

#### Generate a Server Certificate

The certificate usually contains a `.crt` file and a `.key` file, for example, 
`dockerregistry.in.crt` and `dockerregistry.in.key`.


1. Generate a private key.

    ```sh
    openssl genrsa -out dockerregistry.in.key 4096
    ```

2. Generate a certificate signing request (CSR).

   new csr

    ```sh
    openssl req -sha512 -new \
        -subj "/C=IN/ST=Kerala/L=Kerala/O=Kerala Legislative Assembly/OU=E-Niyamasabha/emailAddress=devops@kla.in/CN=dockerregistry.in" \
        -key dockerregistry.in.key \
        -out dockerregistry.in.csr
    ```

3. Generate an x509 v3 extension file.

   Regardless of whether we're using either an FQDN or an IP address to connect to our Harbor host, 
   we must create this file so that we can generate a certificate for our Harbor host that complies
   with the Subject Alternative Name (SAN) and x509 v3 extension requirements. 
   Replace the `DNS` entries to reflect our domain.

    ```shell
    cat > v3.ext <<-EOF
    authorityKeyIdentifier=keyid,issuer
    basicConstraints=CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    extendedKeyUsage = serverAuth
    subjectAltName = @alt_names

    [alt_names]
    DNS.1=dockerregistry.in
    DNS.2=www.dockerregistry.in
    DNS.3=dockerregistry
    EOF
    ```

4. Use the `v3.ext` file to generate a certificate for our Harbor host.

   Replace the `dockerregistry.in` in the CRS and CRT file names with the Harbor host name.

    ```sh
    openssl x509 -req -sha512 -days 3650 \
        -extfile v3.ext \
        -CA ca.crt -CAkey ca.key -CAcreateserial \
        -in dockerregistry.in.csr \
        -out dockerregistry.in.crt
    ```
   Please keep a copy in this repository [here](./dockerregistry.in.crt)
#### Provide the Certificates to Harbor and Docker

After generating the `ca.crt`, `dockerregistry.in.crt`, and `dockerregistry.in.key` files, we must provide them to Harbor and to Docker, and reconfigure Harbor to use them.

clean the /data directory if anything exists

1. Copy the server certificate and key into the certificates' folder on our Harbor host.
   ```shell
   mkdir -p /data/cert/
   cp dockerregistry.in.crt /data/cert/
   cp dockerregistry.in.key /data/cert/
   ```
   
2. Convert `dockerregistry.in.crt` to `dockerregistry.in.cert`, for use by Docker.

   The Docker daemon interprets `.crt` files as CA certificates and `.cert` files as client certificates.

    ```sh
    openssl x509 -inform PEM -in dockerregistry.in.crt -out dockerregistry.in.cert
    ```

3. Copy the server certificate, key and CA files into the Docker certificates folder on the Harbor host. 
   We must create the appropriate folders first.
   
   ```shell
    cd /home/kla/cert/
    mkdir -p /etc/docker/certs.d/dockerregistry.in/
   ```

    ```sh
    cp dockerregistry.in.cert /etc/docker/certs.d/dockerregistry.in/
    cp dockerregistry.in.key /etc/docker/certs.d/dockerregistry.in/
    cp ca.crt /etc/docker/certs.d/dockerregistry.in/
    ```

4. Restart Docker Engine.

    ```sh
    systemctl restart docker
    ```

We might also need to trust the certificate at the OS level.

```shell
cp ca.crt /usr/local/share/ca-certificates/ca-docker.cert
update-ca-certificates
```

The following example illustrates a configuration that uses custom certificates.

```
/etc/docker/certs.d/
    └── dockerregistry.in:port
       ├── dockerregistry.in.cert  <-- Server certificate signed by CA
       ├── dockerregistry.in.key   <-- Server key signed by CA
       └── ca.crt               <-- Certificate authority that signed the registry certificate
```


### Configure the Harbor YML File

assume the directory is /home/kla/harbor/harbor
```shell
cd /home/kla/harbor/harbor
```
We set system level parameters for Harbor in the `harbor.yml` file that is contained in the installer package.
These parameters take effect when we run the `install.sh` script to install or reconfigure Harbor.

After the initial deployment and after we have started Harbor, 
we perform additional configuration in the Harbor Web Portal.



#### Required Parameters

The table below lists the parameters that must be set when we deploy Harbor. By default, 
all the required parameters are uncommented in the `harbor.yml` 
file. The optional parameters are commented with `#`. We do not necessarily need to
change the values of the required parameters from the defaults that are provided, 
but these parameters must remain uncommented. At the very least, we must update the `hostname` parameter.

verify
```shell
sudo nano harbor.yml
```

and update with new changes in [harbor.yml](./harbor.yml)

create data directory 
```yaml

# Configuration file of Harbor

# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname: dockerregistry.in



# https related config
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /etc/docker/certs.d/dockerregistry.in/dockerregistry.in.cert
  private_key: /etc/docker/certs.d/dockerregistry.in/dockerregistry.in.key
  
harbor_admin_password: Kla@2022
```


#### Deploy or Reconfigure Harbor

If we already deployed Harbor with HTTP and want to reconfigure it to use HTTPS, perform the following steps.

1. Run the `prepare` script to enable HTTPS.

   Harbor uses an `nginx` instance as a reverse proxy for all services.
   We use the `prepare` script to configure `nginx` to use HTTPS.
   The `prepare` is in the Harbor installer bundle, at the same level as the `install.sh` script.

    ```sh
    ./prepare
    ```

2. If Harbor is running, stop and remove the existing instance.

   Our image data remains in the file system, so no data is lost.

    ```sh
    docker-compose down -v
    ```

3. Restart Harbor:

    ```sh
    docker-compose up -d
    ```

## Verify the HTTPS Connection

After setting up HTTPS for Harbor, we can verify the HTTPS connection by performing the following steps.

*   Install the [ca](ca.crt) certificates in local machines for accessing the dashboard
   without certificate warning
   
   
   [refer](https://thomas-leister.de/en/how-to-import-ca-root-certificate/).
   
   also add the domain to hosts file
   
   c:\Windows\System32\Drivers\etc\hosts (windows)
   /etc/hosts (ubuntu)

* Open a browser and enter https://dockerregistry.in. It should display the Harbor interface.

  Some browsers might show a warning stating that the Certificate Authority (CA) is unknown. 
  This happens when using a self-signed CA that is not from a trusted third-party CA.
  We can import the CA to the browser to remove the warning.


* Log into Harbor from the Docker client.

    ```sh
    docker login dockerregistry.in
    ```
