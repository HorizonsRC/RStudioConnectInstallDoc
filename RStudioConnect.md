# RStudio Connect: Build Guide for Horizons

|Property|Value|
|:--|:--|
|Target OS|Ubuntu 18.04.xx|
|Privileges|sudo|
|Reference| [RStudio Connect: Admin Guide](https://docs.rstudio.com/connect/admin/index.html)|
|Compiled by|Sean Hodges|
|Last updated|2019-11-13|
## Checklist

- [ ] Install R
- [ ] Install package dependencies for likely R packages 
- [ ] Download & install RStudio Connect
- [ ] Edit RStudio Connect config file
- [ ] Test deployment
- [ ] Adding the HilltopServer package
- [ ] Adding reverse proxy



## 1. Install R
Administrators of the RStudioConnect box should install the versions of R that they wish to support from source so that user content is run in an environment as close as possible to the development environment. This allows maintenance of multiple versions of R simultaneously and mitigates the risk associated with updating the version of R.

Add GPG Key: logged into your Ubuntu 18.04 server as a sudo non-root user, add the relevant GPG key.

```console
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E298A3A825C0D65DFD57CBB651716619E084DAB9
```

Add the R repository
```console
sudo add-apt-repository 'deb https://cloud.r-project.org/bin/linux/ubuntu bionic-cran35/'
```
If you’re not using 18.04, find the relevant repository from the R Project Ubuntu list, named for each release.

Update Package Lists
``` console
sudo apt update
sudo apt upgrade
```

Ensure that the `src` URI's in `/etc/apt/sources.list` are uncommmented, as various libraries and packages will need to be built from source. Without this, the `sudo apt-get build-dep` will error.

To build R from source, first, acquire the build dependencies for R.

In Ubuntu, you can install build dependencies with

```console
sudo apt-get build-dep r-base
```

Second, you should download and unpack the [source tarball](https://cran.r-project.org/src/base/R-3/R-3.6.0.tar.gz) for the version of R that you want to install from CRAN (downloading and compiling a few of the tarballs from previous versions would be beneficial (3.1.3 ,3.2.5 ,3.3.3, 3.4.4, 3.5.2), along with setting a schedule to keep R versions up to date) 

To install `R-3.6.0`:

```console
# Download and extract source code
cd /tmp
wget https://cran.r-project.org/src/base/R-3/R-3.6.0.tar.gz
tar -xzvf R-3.6.0.tar.gz
cd R-3.6.0
```

The final step is to configure, make, and install R. The recommended install location, for use by RStudio Connect, is at `/opt/R/3.6.0`.

```console
# Build R from source
sudo ./configure \
    --prefix=/opt/R/$(cat VERSION) \
    --enable-memory-profiling \
    --enable-R-shlib \
    --with-blas \
    --with-lapack
	
sudo make
sudo make install
```

**It is very important that the R installation folder is not moved once it is installed from source. Libraries are statically linked, so moving the folder will break the installation of R.**

To test that the installation went smoothly, execute:

```console
/opt/R/3.6.0/bin/R --version
```
Your output should look something like:

```
R version 3.6.0 (2019-04-26) -- "Planting of a Tree"
Copyright (C) 2019 The R Foundation for Statistical Computing
Platform: x86_64-pc-linux-gnu (64-bit)

R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under the terms of the
GNU General Public License versions 2 or 3.
For more information about these matters see
http://www.gnu.org/licenses/.

```

-----


## 2. Install package dependencies for likely R packages 

The following system dependencies are required by many common R packages and nearly all deployments will need to provide these. These package names may vary slightly between different versions of Ubuntu.

* `build-essential`
* `libcurl4-gnutls-dev`
* `openjdk-7-*` # may require also executing `R CMD javareconf`
* `libxml2-dev`
* `libssl-dev`
* `texlive-full` # very large dependency, but needed to render PDF documents from R Markdown
* `libblas-dev`
* `libpack-dev`
* `libudunits2-dev`
* `libgdal-dev`

Typically, the following commands at the console will do the trick:

```console
sudo apt-get install build-essential libcurl4-gnutls-dev libxml2-dev libssl-dev texlive-full libblas-dev libudunits2-dev libgdal-dev
sudo apt-get libpack-dev
# sudo apt install openjdk-11-jdk ## This was already installed on production environment
sudo R CMD javareconf
```

-----

## 3. Install RStudio Connect

You will use `gdebi` to install Connect and its dependencies. It is installed via the `gdebi-core` package.

```console
sudo apt-get install gdebi-core
```
### 3.1 Software Installation 

RStudio only provides (as at June 2019) a pre-built binary for the 64-bit architecture.

These steps will install RStudio Connect:

```console
cd /tmp
wget https://s3.amazonaws.com/rstudio-connect/rstudio-connect_1.7.4.2-16_amd64.deb
```

Once the `.deb` file is available locally, run the following command to install RStudio Connect.

``` console
sudo gdebi rstudio-connect_1.7.4.2-16_amd64.deb
```

This will install Connect into `/opt/rstudio-connect/`, and create a new rstudio-connect user.

### 3.2 Supplemental packages

There are additional system dependencies that may be required for some R packages depending on the types of R packages your users are leveraging. You could consider providing these packages for your users now, or wait until they are requested.

```console
libgmp10-dev
libgsl0-dev
libnetcdf6
libnetcdf-dev
netcdf-bin
libdigest-hmac-perl
libgmp-dev
libgmp3-dev
libgl1-mesa-dev
libglu1-mesa-dev
libglpk-dev
tdsodbc
freetds-bin
freetds-common
freetds-dev
odbc-postgresql
libtiff-dev
libsndfile1
libsndfile1-dev
libtiff-dev
tk8.5
tk8.5-dev
tcl8.5
tcl8.5-dev
libgsl0-dev
libv8-dev
```


## 4. Edit RStudio Connect config file

RStudio Connect is controlled by the `/etc/rstudio-connect/rstudio-connect.gcfg` configuration file. You will edit this file to make server-wide configuration changes to the system.

Make the following initial settings to this file to set up email, AD, and suppress warnings about insecure connections (site is running over HTTP on Horizons internal network - best practice would be to set up HTTPS).

### 4.1 Email

Set SenderEmail to the service desk email.

``` console
[Server]
SenderEmail = servicedesk@horizons.govt.nz
```

-----

Restart RStudio Connect after altering the rstudio-connect.gcfg configuration file.


### 4.2 Active Directory
Adding LDAP authentication is done through setting `Provider  = ldap` under the `[Authentication]` section of the config file.

``` console
[Authentication]
Provider  = ldap
```

LDAP settings were obtained from IT, and stored in 1Password - search for zRStudioLDAP. Copy these settings to any new instance of RStudio Connect.

-----

Restart RStudio Connect after altering the rstudio-connect.gcfg configuration file.

`sudo systemctl restart rstudio-connect`

### 4.3 Warning about insecure HTTP connection
Running RStudio Connect over HTTP, as opposed to HTTPS, results in a red status bar appearing on login. To prevent this message from appearing, add the `NoWarning` setting to the `[HTTP]` part of the config file:

``` console
[HTTP]
NoWarning = true
```

-----

Restart RStudio Connect after altering the rstudio-connect.gcfg configuration file.

## 5. Activate License

```console
sudo /opt/rstudio-connect/bin/license-manager activate <license code>
sudo systemctl restart rstudio-connect
```

## 6. Production Configuration Settings

### 6.1 Reverse Proxy

The default port for RStudio Connect is :3939. This is an issue on the internal wireless network, as only traffic on ports 80, 8080 and 443 are allowed through. The impact of this is that wireless devices cannot connect to RStudio Connect directly, and must use a remote desktop client instead.

To resolve this, a reverse proxy (Nginx, Apache) can be configured on the Ubuntu box to redirect from port :3939 to port :80. My preference is to use nginx.


### 6.2 Setting up Nginx

These notes are taken directly from the RStudion Connect documentation.

On Ubuntu, a version of Nginx that supports reverse-proxying can be installed using the following command:

``` console
sudo apt-get install nginx
```

The configuration file below lets Nginx act as a reverse proxy to RStudio Connect. The configuration does no path rewriting; all requests beneath `http://rsconnect.horizons.govt.nz` are routed to Connect. 

The incoming request URI is communicated to Connect using the X-RSC-Request header.

This configuration assumes that RStudio Connect running on the same server as the Nginx proxy and listening on port 3939. 

To edit the config file:
``` console
sudo nano /etc/nginx/nginx.conf
```

``` console
http {
  map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
  }
  server {
    listen 80;

    client_max_body_size 0; # Disables checking of client request body size

    location / {
      proxy_set_header X-RSC-Request $scheme://$host:$server_port$request_uri;
      proxy_pass http://localhost:3939;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
      proxy_http_version 1.1;
      proxy_buffering off; # Required for XHR-streaming
    }
  }
}

```

Restart Nginx after any configuration change.

`sudo systemctl restart nginx`

or

``` console
sudo systemctl stop nginx
sudo systemctl start nginx
```

## 7. Setting up HilltopServer package
HilltopServer is a proprietery software package that handles requests for timeseries data. Accessing timeseries data through HilltopServer is a necessary capability for Horizons.

RStudioConnect expects to compile packages from source, but this is not possible for the HilltopServer package. The steps required to install this package are as follows

### 7.1 Copy the Package

1. `scp` the hilltopserver package to a user folder on the RStudio-Connect box. There are a number of methods for achieving this. The simplest is to use WinSCP.

2. Set up user libary folders for R packages. Reviewing StackOverflow and other support sites, this documentation suggests using the following location to set this up: `/usr/local/share` (the documentation assumes this location is being used). Within this location, create an `rpkgs` directory, and beneath that, directories for each major and minor versions of R.

``` console
# Example
cd /usr/local/share
sudo mkdir rpkgs
cd rpkgs
sudo mkdir 3.4
sudo mkdir 3.5
sudo mkdir 3.6
```

2. For each R version directory that has been created, copy the HilltopServer package across from the user folder

``` console
# These commands are executed from /usr/local/share/rpkgs
sudo cp -r ~/HilltopServer 3.4/
sudo cp -r ~/HilltopServer 3.5/
sudo cp -r ~/HilltopServer 3.6/
```

3. Now, for each R environment, edit the `Renviron` file: 

``` console
# Editing for R version 3.4.x
cd /opt/R/3.4.2/lib/R/etc
sudo nano Renviron
```

Now, edit the `R_LIBS_USER` variable in the `Renviron` file, ensuring that any other entries for this variable are hashed out.

`R_LIBS_USER=${R_LIBS_USERS-'/usr/local/share/rpkgs/3.4'}`

Repeat for each version of R installed, using the appropriate major.minor version numbers.

### 7.2 Edit the config file

As a final step, to make RStudio-Connect aware of the manually installed pacage, add the following section to the config file.

``` console
[Packages]
External = HilltopServer
```

This setting will force RStudio-Connect to skip compilation when it strikes this package in a project that is being uploaded. Many `External = ` items can be added on new lines in this section.


Restart the RStudio-Connect server

`sudo systemctl restart rstudio-connect`

