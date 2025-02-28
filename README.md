# EDB Postgres Advanced Server security enhancements

## Intro
This demo is deployed using Vagrant and will deploy one EDB Postgres Advanced Server node.

## Demo prep
### Pre-requisites
To deploy this demo the following needs to be installed in the PC from which you are going to deploy the demo:

- VirtualBox (https://www.virtualbox.org/)
- Vagrant (https://www.vagrantup.com/)
- Vagrant Hosts plug-in (`vagrant plugin install vagrant-hosts`)
- Vagrant Reload plug-in (`vagrant plugin install vagrant-reload`)
- A file called `.edb_subscription_token` with your EDB repository 2.0 token in your `$HOME/token` directory. This token can be found in your EDB account profile here: https://www.enterprisedb.com/accounts/profile


### Provisioning VM's.
Provision the host using `00-provision.sh`. This will create the bare virtual machine and will take appx. 5 minutes to complete. 

After provisioning, the hosts will have the following directories mounted in their filesystem:

| Host directory | Directory inside VM | Purpose |
| --- | --- | --- |
| `.` (demo dir) | `/vagrant` | Access to the scripts needed to run the demo | 
| `./scripts` | `/vagrant_scripts` | Scripts used for provisioning and setup of the demo |
| `./config` | `/vagrant_config` | Configuration settings for the server and applications |
| `$HOME/tokens` | `/tokens` | Directory hosting your subscription token |

### Userid and Passwords
- enterprisedb / enterprisedb (Owner of the instance)
- webuser / webuser (Owner of the SQL/Protect demo database)

## Use cases
### Password Profile Management
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

### Data redaction
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

### SQL*Wrap
- Show `list_customers.sql`
- In another pane, connect to the database using `psql edb`
- Import the stored procedure using `\\i list_customers.sql`.
- Display the stored procedure using `SELECT prosrc FROM pg_proc WHERE proname = 'list_customers';`
- Encrypt the stored procedure using `sqlwrap -i list_customers.sql`. A `list_customers.plb` is created.
- Show the list_customers.plb file using `less list_customers.plb`
  ```
  [enterprisedb@epas EPAS_security]$ cat list_customers.plb 
  $__EDBwrapped__$$PROTOCOL2$
  UTF8
  d+TDv2VChMI2fRLBe1M/Fz7IqNyA4WTqHdEuLD3LW47mGjlqSGeF3JTUokymbpH2Jyo9nzoo6UMU
  OxXsLL7cWRLT3jGB5CVuEdrc35/rq3PNu7cqX/xXn4X+1NM+2qwX9kI4P7cAF1/JUGnPAbFTsNoe
  UOFostTmiUZN/wosweJA388yP+4Jb8Af9SfnmOyBFKJaQBX4XoKydUsofOkxJD2wx1n4IhkkZhl1
  Zc+47rTEUZem4PIAQBVXv+np8omV9EuFSC6CI6UF6h/0Cdiqv6gaPorp9MSZ+YNwbZ7OKa7Jhm+c
  rsZ2LYUEBR9Uw5QOUhbrOZeGo9VIm048Z3F+/gUpTxB22ksl7BSP0Xi9NqAKySc3yDrz/4G4Q4qC
  C3aDLzaOx/lgOfm2R+YbVE1uZu65en6izsX6JNrdZi0m3ySxDAY3rZRdHgm12S5gPZWXt2W0ldjO
  0K5jDnWokeyIpA==
  $__EDBwrapped__$
  [enterprisedb@epas EPAS_security]$ 
  ```
- Import the wrapped stored procedure using `\\i list_customers.plb`
- Show the warpped function again using `SELECT prosrc FROM pg_proc WHERE proname = 'list_customers';`
- Run `exec list_customers;` to show that the stored procedure is working correctly.

### Transparent Data Encryption (TDE)
- Become user `enterprisedb` using `sudo su - enterprisedb` and `cd` into `EPAS_security`.
- Create a standard cluster using script `31_create_cluster_no_tde.sh` This database will run on port 6444.
- Create a TDE-enabled cluster using script `32_create_cluster_with_tde.sh`. This database will run on port 6445.
- Open four terminal panes.

| Top Left | Top Right | Bottom Left | Bottom Right |
| --- | --- | --- | --- | 
| `./scripts/ssh.sh` |  `./scripts/ssh.sh` |  `./scripts/ssh.sh` |  `./scripts/ssh.sh` | 
| `sudo su - enterprisedb` | `sudo su - enterprisedb` | `sudo su - enterprisedb` |`sudo su - enterprisedb` |
| `cd /vagrant` |  `cd /vagrant` | `cd /vagrant` | `cd /vagrant` |
| | | `cat 31_create_cluster_no_tde.sh` | `cat 32_create_cluster_with_tde.sh` |
| | | `31_create_cluster_no_tde.sh` | `32_create_cluster_with_tde.sh` |
| `psql -p 6444 edb` | `psql -p 6445 edb` | | |
| | | `less $PGDATA/../datanotde/postgresql.conf` |`less $PGDATA/../datanotde/postgresql.conf` |
| | | `/Data\ Encrypt`| `/Data\ Encrypt` |
| `select data_encryption_version from pg_control_init(); | `select data_encryption_version from pg_control_init(); | | |
| | | | `cat $PGDATA/../datawithtde/pg_encryption/key.bin` |
| `\i 33-create_user.sql` | `\i 33-create_user.sql` | | |
| `select pg_relation_filepath('users');` | `select pg_relation_filepath('users');` |
| | | `hexdump -C $PGDATA/../datanotde/<paste the result>` | `hexdump -C $PGDATA/../datawithtde/<paste the result>` |
| | | `pg_dump -p 6444 -U enterprisedb edb` | `pg_dump -p 6445 -U enterprisedb edb` |

### Enhanced Auditing
Log is in /var/lib/edb/as17/data/edb_audit
```
2025-02-25 14:53:49.006 CET,"enterprisedb","edb",34910,"192.168.0.130:53014",67bdcb6c.885e,2,"idle",2025-02-25 14:53:48 CET,5/3,0,AUDIT,00000,"statement: select * from customers limit 1;",,,,,,,,,"psql","client backend",,0,"SELECT","","select"
2025-02-25 14:53:49.077 CET,"enterprisedb","edb",34911,"192.168.0.130:53015",67bdcb6d.885f,1,"authentication",2025-02-25 14:53:49 CET,6/1,0,AUDIT,00000,"connection authorized: user=enterprisedb database=edb",,,,,,,,,"","client backend",,0,"","","connect"
2025-02-25 14:53:49.084 CET,"enterprisedb","edb",34911,"192.168.0.130:53015",67bdcb6d.885f,2,"idle",2025-02-25 14:53:49 CET,6/3,0,AUDIT,00000,"statement: select count(*) from customers;",,,,,,,,,"psql","client backend",,0,"SELECT","","select"
2025-02-25 14:54:08.041 CET,"enterprisedb","edb",34915,"192.168.0.130:53044",67bdcb80.8863,1,"authentication",2025-02-25 14:54:08 CET,7/1,0,AUDIT,00000,"connection authorized: user=enterprisedb database=edb",,,,,,,,,"","client backend",,0,"","","connect"
2025-02-25 14:54:08.047 CET,"enterprisedb","edb",34915,"192.168.0.130:53044",67bdcb80.8863,2,"idle",2025-02-25 14:54:08 CET,7/3,0,AUDIT,00000,"statement: CREATE USER hr WITH PASSWORD 'hr';",,,,,,,,,"psql","client backend",,0,"CREATE ROLE","","create"
2025-02-25 14:54:08.137 CET,"enterprisedb","edb",34916,"192.168.0.130:53045",67bdcb80.8864,1,"authentication",2025-02-25 14:54:08 CET,8/1,0,AUDIT,00000,"connection authorized: user=enterprisedb database=edb",,,,,,,,,"","client backend",,0,"","","connect"
2025-02-25 14:54:08.143 CET,"enterprisedb","edb",34916,"192.168.0.130:53045",67bdcb80.8864,2,"idle",2025-02-25 14:54:08 CET,8/3,0,AUDIT,00000,"statement: CREATE USER dba WITH PASSWORD 'dba';",,,,,,,,,"psql","client backend",,0,"CREATE ROLE","","create"
2025-02-25 14:54:08.231 CET,"enterprisedb","edb",34917,"192.168.0.130:53046",67bdcb80.8865,1,"authentication",2025-02-25 14:54:08 CET,9/1,0,AUDIT,00000,"connection authorized: user=enterprisedb database=edb",,,,,,,,,"","client backend",,0,"","","connect"```
```

### SQL/Protect
1. `71_create_user_to_monitor.sh` creates the `webuser` which runs the web application and enabled monitoring of this user.
2. `72_switch_monitoring_on` shows currently learned relations and switches on monitoring.
3. On your local workstation, open `http:<ip of epas server>:5000` in a browser.
4. Do a search for `Bean` and press `Search (Unsafe)`.
5. Do a search for `Bean' OR '1'='1` and you will get all records. This implies a data breach.

## Demo cleanup
To clean up the demo environment you just have to run `./99-deprovision.sh`. This will remove the virtual machines and everything in it.

## TODO / To fix
