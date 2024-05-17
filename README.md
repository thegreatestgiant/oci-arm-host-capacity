YouTube video instruction [https://youtu.be/uzAqgjElc64](https://youtu.be/uzAqgjElc64) 
is a bit outdated regarding [Configuration](#configuration) but still can be useful for the rest.

- [Generating API key](#generating-api-key)
- [Installation](#installation)
- [Configuration](#configuration)
  - [Create/copy .env file](#createcopy-env-file)
  - [General](#general)
  - [Private key](#private-key)
  - [Instance parameters](#instance-parameters)
    - [Mandatory](#mandatory)
      - [OCI_SUBNET_ID and OCI_IMAGE_ID](#oci_subnet_id-and-oci_image_id)
- [Running the script](#running-the-script)
- [Periodic job setup (cron)](#periodic-job-setup-cron)
  - [Linux / WSL](#linux--wsl)

## Generating API key

After logging in to [OCI Console](http://cloud.oracle.com/), click profile icon and then "My Profile"

Go to Resources -> API keys, click "Add API Key" button

Make sure "Generate API Key Pair" radio button is selected, click "Download Private Key" and then "Add".

Copy the contents from textarea and save it to file with a name "config". I put it together with *.pem file in newly created directory /home/ubuntu/.oci

## Installation

Clone and move it back
```bash
git clone https://github.com/thegreatestgiant/oci-arm-host-capacity.git api &&
cd api &&
rm -rf oci-arm-host-capacity images README.md && 
chmod 777 oci.log
```
composer stuff
```bash
sudo apt update && sudo apt install -y php libapache2-mod-php php8.1-curl php8.1-dom cron
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer
composer install
```

## Configuration

### Create/copy .env file

Copy `.env.example` as `.env`
```bash
cp .env.example .env
```
You must modify `.env` file below. **Don't push/share it as it possibly contains sensitive information.** 

All parameters except `OCI_AVAILABILITY_DOMAIN` are mandatory to be set. Please read the comments in `.env` file as well.

### General

Region, user, tenancy, fingerprint should be taken from textarea during API key generation step.
Adjust these values in `.env` file accordingly:
- `OCI_REGION`
- `OCI_USER_ID`
- `OCI_TENANCY_ID`
- `OCI_KEY_FINGERPRINT`

### Private key

`OCI_PRIVATE_KEY_FILENAME` is an absolute path (including directories) or direct public accessible URL to your *.pem private key file.

### Instance parameters

#### Mandatory

##### OCI_SUBNET_ID and OCI_IMAGE_ID

You must start instance creation process from the OCI Console in the browser (Menu -> Compute -> Instances -> Create Instance)

Change image and shape. 
For Always free AMD x64 - make sure that "Always Free Eligible" availabilityDomain label is there:

![Changing image and shape](images/create-compute-instance.png)

ARMs can be created anywhere within your home region.

Adjust Networking section, set "Do not assign a public IPv4 address" checkbox. If you don't have existing VNIC/subnet, please create VM.Standard.E2.1.Micro instance before doing everything.

![Networking](images/networking.png)

"Add SSH keys" section does not matter for us right now. Before clicking "Create"…

![Add SSH Keys](images/add-ssh-keys.png)

…open browser's dev tools -> network tab. Click "Create" and wait a bit most probably you'll get "Out of capacity" error. Now find /instances API call (red one)…

![Dev Tools](images/dev-tools.png)

…and right click on it -> copy as curl. Paste the clipboard contents in any text editor and review the data-binary parameter. 
Find `subnetId`, `imageId` and set `OCI_SUBNET_ID`, `OCI_IMAGE_ID`, respectively.

Note `availabilityDomain` for yourself, then read the corresponding comment in `.env` file regarding `OCI_AVAILABILITY_DOMAIN`.


## Running the script

```bash
php ./index.php
```

I bet that the output (error) will be similar to the one in a browser a few minutes ago
```json
{
    "code": "InternalError",
    "message": "Out of host capacity."
}
```
or if you already have instances:
```json
{
    "code": "LimitExceeded",
    "message": "The following service limits were exceeded: standard-a1-memory-count, standard-a1-core-count. Request a service limit increase from the service limits page in the console. "
}
```

## Periodic job setup (cron)

### Linux / WSL

You can now setup periodic job to run the command

Get full path to PHP binary
```bash
which php
```
Usually that's `/usr/bin/php`

Setup itself:
```bash
EDITOR=nano crontab -e
```
Add new line to execute the script every minute and append log the output:
```bash
* * * * * /usr/bin/php /home/sean/api/index.php >> /home/sean/api/oci.log
```
**NB!** Use absolute paths wherever possible
