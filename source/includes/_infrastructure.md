# Infrastructure

## Getting Started

1. Install the necessary Python libraries by running `pip install -r requirements.txt`.

2. Download the vaultpass using `aws s3 cp s3://jako-infrastructure/credentials/vaultpass .vaultpass --profile jako`



## How To

### Deploy to Staging or Production

```sh
# deploy to the staging environment
bash bin/deploy.sh ubiq staging jakoenterprise/ubiq-magento 7de0d6692209457c5e0fd8897add2f9266346c0a
# --> staging d-96I9EBUEL

# deploy to the KicksUSA production and admin environments at the same time
bash bin/deploy.sh kicksusa production jakoenterprise/kicksusa-magento 7de0d6692209457c5e0fd8897add2f9266346c0a
# --> production d-96I9EBUEL
# --> admin d-96I9EBUEL

# check the deploy status
bash bin/status.sh d-96I9EBUEL d-96I9EBUEW # and so on
```

The output (if there's no error) will be a deployment ID, such as `d-96I9EBUEL`. You can use this identifier to check the status of the deployment.

**NOTE: there is no "rollback" mechanism right now. If a deployment is bad, re-deploy the last known working commit.**

### SSH Into An Environment

```sh
bash bin/ssh.sh ubiq staging
bash bin/ssh.sh ubiq production
bash bin/ssh.sh kicksusa staging
bash bin/ssh.sh kicksusa production
```

We should avoid this if possible and a) use ad-hoc commands instead, or b) build the process as a Magento feature, but sometimes you just _need_ to SSH into an environment. We use what's called a [bastion host](https://en.wikipedia.org/wiki/Bastion_host) as a doorway to each private network. To simplify accessing an environment, just use the provided shell script.

**NOTE: you're SSH key needs to be whitelisted and built into the AWS AMI for this to work, otherwise someone with a whitelisted IP will need to add your public key to the `www-data` user's `~/.ssh/authorized_keys` file.**

### Run Ad-Hoc Commands

```sh
# restart the nginx service in ubiq production
ansible -b -i manage/packer/hosts/ec2.py \
  --vault-password-file .vaultpass \
  -m service -a "name=nginx state=restarted" \
  "tag_environment_production:&tag_application_ubiq"

# list a directory on kicksusa staging
ansible -b -i manage/packer/hosts/ec2.py \
  --vault-password-file .vaultpass \
  -m shell -a "ls /srv/www/magento" \
  "tag_environment_staging:&tag_application_kicksusa"

# enable maintenance mode on ubiq staging
ansible -b -i manage/packer/hosts/ec2.py \
  --vault-password-file .vaultpass \
  -m shell -a "touch /srv/www/magento/maintenance.flag" \
  "tag_environment_staging:&tag_application_ubiq"

# disable maintenance mode on ubiq staging
ansible -b -i manage/packer/hosts/ec2.py \
  --vault-password-file .vaultpass \
  -m shell -a "rm /srv/www/magento/maintenance.flag" \
  "tag_environment_staging:&tag_application_ubiq"
```

We continue to use Ansible for managing our infrastructure. If you need to run an ad-hoc (one-off) command you can use Ansible to do this. We use a dynamic inventory (`ec2.py`), so Ansible will download the latest list of servers before executing the command (the number of servers can change due to autoscaling).

The tags available to you are:

- `tag_application_kicksusa`

- `tag_application_ubiq`

- `tag_environment_production`

- `tag_environment_staging`

- `tag_role_kicksusa_admin`

- `tag_role_kicksusa_web`

- `tag_role_ubiq_admin`

- `tag_role_ubiq_web`

The syntax `tag:&tag` is an _intersection_ of two tags. It means "find all the servers that have both tags". Here are some examples of how to target specific sets of servers -- imagine **three UBIQ nodes** in **production**, **two web** and **one admin**:

- `tag_environment_production:&tag_role_ubiq_web` targets the two web nodes

- `tag_environment_production:&tag_role_ubiq_admin` targets the one admin node

- `tag_environment_production:&tag_application_ubiq` targets all three

With that info you should be able to target the servers you need to perform a given task. You can reference the [Ansible docs](http://docs.ansible.com/ansible/modules_by_category.html) for all of the available modules.

### Image Importer Process

```sh
# 1. SSH into a node (obviously switch this up for UBIQ)
bash bin/ssh.sh kicksusa production

# 2. switch to the app directory
cd /srv/www/magento

# 3. run the CLI image importer to insert records to the DB (skip the S3 sync)
php shell/local/jako.php rms:imageImporter --skip-sync

# 4. connect to MySQL (the host for UBIQ is db.ubiqlife.com)
mysql -h db.kicksusa.com -u your.user -p

# 5. run the image import stored procedures to associate records in Magento

# kicksusa
CALL kicks_rms_import.sp_image_name_load();
CALL kicks_rms_import.sp_image_refresh();

# ubiq
CALL ubiq_rms_import.sp_ee_image_name_load();
CALL ubiq_rms_import.sp_ee_image_refresh();
```

This is a multi-stage process that ensures the latest images are associated to their products in the Magento database. **You'll need SSH access and MySQL access to make this happen.**

### Inventory Load

```sh
# 1. SSH into a node (obviously switch this up for UBIQ)
bash bin/ssh.sh kicksusa production

# 2. connect to MySQL (the host for UBIQ is db.ubiqlife.com)
mysql -h db.kicksusa.com -u your.user -p

# 3. run the inventory import stored procedures

# kicksusa
CALL kicks_rms_import.sp_image_name_load();
CALL kicks_rms_import.sp_rmsinventory_raw_load();
CALL kicks_rms_import.sp_inventory_load();
CALL kicks_rms_import.sp_url_key_load();
CALL kicks_rms_import.sp_inventory_load_cleanup();

# ubiq
CALL ubiq_rms_import.sp_ee_image_name_load();
CALL ubiq_rms_import.sp_ee_inventory_load();
CALL ubiq_rms_import.sp_ee_map_load();
CALL ubiq_rms_import.sp_ee_url_key_load();

# 4. verify inventory import status by viewing the logs, looking
# for "log_short_description"s like "INVENTORY LOAD ENDED"

# kicksusa
SELECT * FROM kicks_globals.TBL_LOG ORDER BY log_id DESC LIMIT 100;

# ubiq
SELECT * FROM ubiq_globals.TBL_LOG ORDER BY log_id DESC LIMIT 100;
```

This is the process that ensures inventory updates and new products are moved into the Magento DB from the warehouse management system (RMS) (see [diagram](https://docs.google.com/drawings/d/1RATPCf7aXYLFEkBYrMuZVwgQXGSmOsSlFLZNL4JplqA/edit)). **You'll need SSH access and MySQL access to make this happen.**

### Build an AMI or Vagrant Box

The Packer process will build a new AWS AMI **and** a new Vagrant box for development (which is stored in S3). Please review the [`build.json`](https://github.com/jakoenterprise/infrastructure/blob/18b85c6eee6973945ef35ea04f6e1bbc5c1e7010/manage/packer/build.json) file for more details.

1. Update the `box_version` in `manage/packer/build.json`

2. `bash bin/build.sh`
