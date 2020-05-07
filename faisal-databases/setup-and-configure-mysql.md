# Instructions to set up MySQL on AKS, Client for MySQL, configurations needed to run updated code

## In the cncfapp-on-azure repo, navigate to ~cncfapp-on-azure/Databases/mySQL. 
- Run k create -f 02-azure-file-pvc.yaml 
- Run helm install fta-mysql stable/mysql -f values.yaml

## Below is what you should expect to see: 
```
NAME: fta-mysql
LAST DEPLOYED: Mon Apr 13 15:38:18 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
fta-mysql.default.svc.cluster.local
```
- The FQDN to access the MySQL cluster is **fta-mysql.default.svc.cluster.local**
 
## To connect to your database:
``` 
1. Run an Ubuntu pod that you can use as a client:
 
    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il
 
2. Install the mysql client:
 
    $ apt-get update && apt-get install mysql-client -y
(remember to do this from inside the ubuntu container)
 
3. Connect using the mysql cli, then provide your password (copy the password from Notepad, and then just single click to paste, password will not show, fyi, then the ):
    $ mysql -h fta-mysql -p
```
- You will be promted for a password. Copy and paste the password that is in the values.yaml file.
- Please Note: To paste the password, single right click at the prompt, then hit enter. The password will not be displayed.
- Successful logon will log in and show the **mysql>** prompt.
 
## Dont worry about any other instructions, just follow the above part.

## MySQL Info:
```
contosoexpdb
fta-mysql.default.svc.cluster.local
User: ftacncf
Password: In the values.yaml file
Connection String: Server=myServerAddress;Port=1234;Database=contosoexpdb;Uid=ftacncf;Pwd=FT@CNCF0n@zur3;
```

## Then run the following. Will upate the HELM Chart so code creates the DB and the Table with correct permissions
```
mysql> CREATE USER IF NOT EXISTS 'ftacncf'@'%' IDENTIFIED BY 'FT@CNCF0n@zur3';
```
## Then, Grant permissions to the user on the database:
```
mysql> GRANT ALL PRIVILEGES ON *.* TO 'ftacncf'@'%';
```

### The chnages needed to the code are checked in and so are updated NuGet Packages 