# Connecting, Loading Data and Auto Scaling

This lab will walk you through the process of connecting to the DB cluster you have just created, and using the cluster for the first time. At the end you will test out how Aurora read replica auto scaling works in practice using a load generator script.

This lab contains the following tasks:

1. Connecting to your workstation EC2 instance
2. Connecting to the DB cluster
3. Loading an initial data set from S3
4. Running a read-only workload

This lab requires the following lab modules to be completed first:

* [Prerequisites](/reinvent/prerequisites/)

## 1. Connecting to your workstation EC2 instance

To interact with the Aurora database cluster, you will use an Amazon EC2 Linux instance that acts like a workstation for the purposes of the labs. All necessary software packages and scripts have been installed and configured on this EC2 instance for you. To ensure a unified experience, you will be interacting with this workstation using <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html" target="_blank">AWS Systems Manager Session Manager</a>. With Session Manager you can interact with your workstation directly from the management console.

Open the <a href="https://eu-west-1.console.aws.amazon.com/systems-manager/session-manager?region=eu-west-1" target="_blank">Systems Manager: Session Manager service console</a>. Click **Configure Preferences**.

!!! warning "Region Check"
    Ensure you are still working in the correct region, especially if you are following the links above to open the service console at the right screen.

!!! note
    The introduction screen with the **Configure Preferences** and **Start Session** buttons only appears when you start using Session Manager for the first time in a new account. Once you have started using this service the console will display the session listing view instead, and the preferences page is accessible by clicking on the **Preferences** tab. From there click the **Edit** button if you wish to change settings.

<span class="image">![Session Manager](../../modules/connect/1-session-manager.png?raw=true)</span>

Check the box next to **Enable Run As support for Linux instances**, and enter `ubuntu` in the text field. This will instruct Session Manager to connect to the workstation using the `ubuntu` operating system user account. Click **Save**.

<span class="image">![Session Preferences](../../modules/connect/1-session-prefs.png?raw=true)</span>

Next, navigate to the **Sessions** tab, and click the **Start session** button.

<span class="image">![Start Session](../../modules/connect/1-start-session.png?raw=true)</span>

Please select the EC2 instance to establish a session with. The workstation is named `labstack-bastion-host`, select it and click **Start session**.

<span class="image">![Connect Instance](../../modules/connect/1-connect-session.png?raw=true)</span>

If you see a black command like terminal screen and a prompt, you are now connected to the EC2 based workstation. With Session Manager it is not necessary to allow SSH access to the EC2 instance from a network level, reducing the attack surface of that EC2 instance.

Execute the following commands to ensure a familiar experience.

```
bash
cd ~
```


## 2. Connecting to the DB cluster

Connect to the Aurora database just like you would to any other MySQL-based database, using a compatible client tool. In this lab you will be using the `mysql` command line tool to connect. The command is as follows:

```
mysql -h [clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab
```

You can find the value for the cluster endpoint parameter in the stack outputs from the end of the [Prerequisites](/reinvent/prerequisites/) section. We have set the DB cluster's database credentials automatically for you, and have also created the schema named `mylab` as well. The credentials were saved to an <a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html" target="_blank">AWS SecretsManager</a> secret.

!!! note
    While we have created the database credentials (MySQL username and password) automatically and set them in the `DBUSER` and `DBPASS` environment variables, you can view and retrieve the credentials stored in the secret using the following command:

    ```
    aws secretsmanager get-secret-value --secret-id [secretArn] | jq -r '.SecretString'
    ```


Once connected to the database, use the code below to create a stored procedure we'll use later in the labs, to generate load on the DB cluster. Run the following SQL queries:

```
DELIMITER $$
DROP PROCEDURE IF EXISTS minute_rollup$$
CREATE PROCEDURE minute_rollup(input_number INT)
BEGIN
 DECLARE counter int;
 DECLARE out_number float;
 set counter=0;
 WHILE counter <= input_number DO
 SET out_number=SQRT(rand());
 SET counter = counter + 1;
END WHILE;
END$$
DELIMITER ;
```


## 3. Loading an initial data set from S3

Once connected to the DB cluster, run the following SQL queries to create an initial table:

```
DROP TABLE IF EXISTS `sbtest1`;
CREATE TABLE `sbtest1` (
 `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
 `k` int(10) unsigned NOT NULL DEFAULT '0',
 `c` char(120) NOT NULL DEFAULT '',
 `pad` char(60) NOT NULL DEFAULT '',
PRIMARY KEY (`id`),
KEY `k_1` (`k`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

Next, load an initial data set by importing data from an Amazon S3 bucket:

```
LOAD DATA FROM S3 MANIFEST
's3-eu-west-1://aurorareinvent2018/output.manifest'
REPLACE
INTO TABLE sbtest1
CHARACTER SET 'latin1'
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\r\n';
```

Data loading may take several minutes, you will receive a successful query message once it completes. When completed, exit the MySQL command line:

```
quit;
```


## 4. Running a read-only workload

Once the data load completes successfully, you can run a read-only workload to generate load on the cluster. Download and review our [simple load testing script](/scripts/loadtest.py) for reference. You will also observe the effects on the DB cluster topology. For this step you will use the **Reader Endpoint** of the cluster. The value of the reader endpoint can be found in your CloudFormation stack outputs.

Run the load generation script from the Session Manager workstation command line:

```
python3 loadtest.py -e [readerEndpoint] -u $DBUSER -p "$DBPASS" -d mylab
```

Now, open the <a href="https://eu-west-1.console.aws.amazon.com/rds/home?region=eu-west-1#databases:" target="_blank">Amazon RDS service console</a> in a different browser tab.

!!! warning "Region Check"
    Ensure you are still working in the correct region, especially if you are following the links above to open the service console at the right screen.

!!! note
    Please ignore any console notification titled **Update your Amazon RDS SSL/TLS certificates before March 5, 2020**, you may see in the RDS service console. The DB clusters provisioned for this workshop will not persist until that date. 

Take note that the reader node is currently receiving load. It may take a minute or more for the metrics to fully reflect the incoming load.

<span class="image">![Reader Load](../../modules/connect/4-read-load.png?raw=true)</span>

After several minutes return to the list of instances and notice that a new reader is being provisioned to your cluster.

<span class="image">![Application Auto Scaling Creating Reader](../../modules/connect/4-aas-create-reader.png?raw=true)</span>

Once the new replica becomes available, note that the load distributes and stabilizes (it may take a few minutes to stabilize).

<span class="image">![Application Auto Scaling Creating Reader](../../modules/connect/4-read-load-balanced.png?raw=true)</span>

You can now toggle back to the Session Manager command line, and type `CTRL+C` to quit the load generator. After a while the additional reader will be removed automatically.