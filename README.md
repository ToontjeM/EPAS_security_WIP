# EDB Postgres Advanced Server security enhancements (WIP)

## Intro
This demo is deployed using Vagrant and will deploy one EDB Postgres Advanced Server node.

## Demo prep
### Pre-requisites
To deploy this demo the following needs to be installed in the PC from which you are going to deploy the demo:

- VirtualBox (https://www.virtualbox.org/)
- Vagrant (https://www.vagrantup.com/)
- Vagrant Hosts plug-in (`vagrant plugin install vagrant-hosts`)
- Vagrant Reload plug-in (`vagrant plugin install vagrant-reload`)
- A file called `.edbtoken` with your EDB repository 2.0 token. This token can be found in your EDB account profile here: https://www.enterprisedb.com/accounts/profile

The environment is deloyed in a VirtualBox private network. Adjust the IP addresses to your needs in `vars.yml`.

### Provisioning VM's.
Provision the host using `vagrant up`. This will create the bare virtual machine and will take appx. 5 minutes to complete. 

After provisioning, the hosts will have the current directory mounted in their filesystem under `/vagrant`

### Userid and Passwords
- enterprisedb / enterprisedb (Owner of the instance)

## Demo flow
### Use case 1: Password Profile Management
1. `11_show_profiles.sh` shows which profiles are available on the server.
2. `12_create_new_admin_profile.sh` creates a new, more restricted, password profile for a second admin user `admin2`
3. `13_change_password.sh` shows a failed password re-use attempt because of the more restrictive password policy in place.
```
ERROR:  password cannot be reused
DETAIL:  The password_reuse_time constraint failed.
```
4. `14_brute_force_admin2.sh` attempts to connect to the database using incorrect credentials. At the 4th attempt, the `admin2` account will be locked.
```
psql: error: connection to server at "192.168.0.211", port 5444 failed: FATAL:  role "admin2" is locked
```
5. `15_unlock_account.sh` unlocks the account.
```
psql -h 192.168.0.211 -p 5444 -U admin2 edb

Password for user admin2: 
psql (16.3 (Homebrew), server 16.3.0)
WARNING: psql major version 16, server major version 16.
         Some psql features might not work.
Type "help" for help.

edb=> 
```

### Use case 2: Redacting data
1. `21_show_customers.sh` shows the content of the `customers` table.
2. `22_create_users.sh` created two users, `hr` and `dba`.
3. `23_create_retention_policies.sh` will create two posicies:
    1. User `hr` will be able to see the full credit card number, but will not be able to see the users password.
    2. User `dba` will be able to see the passwords, but will not be able to see the credit card details.
    
    First the redaction functions will be created, then the policies will be defined using those functions.
4. `24_connect_as_hr_and_dba.sh` will show the data when connected as `hr` or as `dba`.
```
--- Connect as HR (password hr) and select data ---

Password for user hr: 
-[ RECORD 1 ]--------+------------------------
customerid           | 1
firstname            | Justin
lastname             | Elliott
address1             | 373 Wendy Island
address2             | Suite 539
city                 | Gravesville
state                | 
zip                  | 74741
country              | Pitcairn Islands
region               | 3
email                | claytonking@example.net
phone                | 
creditcardtype       | 3
creditcard           | 4549209759447656
creditcardexpiration | 08/29
username             | shelly12
password             | 0
age                  | 49
income               | 133186
gender               | M

--- Connect as DBA (password dba) and select data ---

Password for user dba: 
-[ RECORD 1 ]--------+------------------------
customerid           | 1
firstname            | Justin
lastname             | Elliott
address1             | 373 Wendy Island
address2             | Suite 539
city                 | Gravesville
state                | 
zip                  | 74741
country              | Pitcairn Islands
region               | 3
email                | claytonking@example.net
phone                | 
creditcardtype       | 3
creditcard           | xxxxxxxxxxx47656
creditcardexpiration | 08/29
username             | shelly12
password             | 3@l2LuHBG(
age                  | 49
income               | 133186
gender               | M
```

### Use case 3: SQL*Wrap

### Use case 4: Transparent Data Encryption (TDE)
Need to incorporate https://github.com/ToontjeM/TDEdemo

### Use case 5: Audit Log
Log is in /var/lib/edb/as16/data/edb_audit

### Use case 6: SQL/Protect
1. `71_create_user_to_monitor.sh` creates the `webuser` which runs the web application and enabled monitoring of this user.
2. `72_switch_monitoring_on` shows currently learned relations and switches on monitoring.
3. On your local workstation, open `http:<ip of epas server>:5000` in a browser.
4. Do a search for `Bean` and press `Search (Unsafe)`.
5. Do a search for `Bean' OR '1'='1` and you will get all records. This implies a data breach.
6. 

## Demo cleanup
To clean up the demo environment you just have to run `vagrant destroy`. This will remove the virtual machines and everything in it.

## TODO / To fix
