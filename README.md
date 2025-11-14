
-----

# Deploy a 5-node WarehousePG Cluster (with MinIO and an NFS share pre-installed) on AWS with Terraform

This repository provides a set of Terraform configurations and helper scripts to automate the deployment of a 5-node <a href="https://warehouse-pg.io/" target="_blank" rel="noopener noreferrer">WarehousePG 7</a> cluster on AWS.

The cluster will also have MinIO Server pre-installed on segment host 1, and a cluster-wide NFS share.

The deployment is managed by a user-friendly wrapper script (`deploy.sh`) that prompts for necessary configuration details, making the setup process straightforward.

-----

## 🏛️ Architecture

The Terraform scripts will provision the following AWS resources, creating a logically separated and secure environment for the WarehousePG cluster:

  * **VPC:** A dedicated Virtual Private Cloud with a `10.0.0.0/16` CIDR block to isolate the cluster network.
  * **Subnets:**
      * A **public subnet** (`10.0.1.0/24`) for the Coordinator node.
      * A **private subnet** (`10.0.2.0/24`) for the 4 Segment nodes.
  * **Internet & NAT Gateways:**
      * An Internet Gateway to allow outbound internet access from the public subnet.
      * A NAT Gateway placed in the public subnet, enabling instances in the private subnet (the segment nodes) to access the internet for software downloads without being publicly exposed.
  * **EC2 Instances:** Five EC2 instances based on a Rocky Linux 8.9 AMI (`ami-020c6cfb9f8b61b53`).
      * **Server 1 (Coordinator):** In the public subnet with a public Elastic IP.
      * **Server 2 (Segment host 1):** In the private subnet.
      * **Server 3 (Segment host 2):** In the private subnet.
      * **Server 4 (Segment host 3):** In the private subnet.
      * **Server 5 (Segment host 4):** In the private subnet.
  * **Security Group:** A single security group that:
      * Allows inbound SSH access from a designated jumpbox ip address.
      * Allows all internal traffic within the VPC for seamless communication between cluster nodes.
  * **Elastic IPs:** An Elastic IP is assigned to the Coordinator nodes to provide it with a static public IP addresse.

### Architecture Diagram


```mermaid
graph TD
    subgraph "Internet"
        jumpbox["fa:fa-laptop<br>Jumpbox / Bastion Host"]
    end

    subgraph "AWS Cloud"
        subgraph "VPC (10.0.0.0/16)"
            subgraph "Public Subnet (10.0.1.0/24)"
                Coordinator["fa:fa-server<br><b>Server 1 (Coordinator)</b>"]
                NAT["fa:fa-route<br>NAT Gateway"]
            end
            subgraph "Private Subnet (10.0.2.0/24)"
                %% --- Invisible padding nodes for the top and bottom ---
                top_pad[" "]
                Segment1["fa:fa-server<br><b>Server 2 (Segment 1)</b>"]
                Segment2["fa:fa-server<br><b>Server 3 (Segment 2)</b>"]
                Segment3["fa:fa-server<br><b>Server 4 (Segment 3)</b>"]
                Segment4["fa:fa-server<br><b>Server 5 (Segment 4)</b>"]
                bottom_pad[" "]
            end
        end
        InternetGateway["fa:fa-globe<br>Internet Gateway"]
        EIP1["fa:fa-location-dot<br>Elastic IP 1"]
    end

    %% --- Style the padding nodes to be invisible ---
    style top_pad fill:transparent,stroke:transparent
    style bottom_pad fill:transparent,stroke:transparent

    %% --- Define visible connections (13 total) ---
    jumpbox -- "SSH" --> EIP1
    EIP1 --> Coordinator
    NAT --> InternetGateway
    Coordinator <--"Interconnect"--> Segment1
    Coordinator <--"Interconnect"--> Segment2
    Coordinator <--"Interconnect"--> Segment3
    Coordinator <--"Interconnect"--> Segment4
    Segment1 <--"Interconnect"--> Segment2
    Segment3 <--"Interconnect"--> Segment4
    Segment1 <--"Interconnect"--> Segment3
    Segment2 <--"Interconnect"--> Segment4
    Segment3 <--"Interconnect"--> Segment2
    Segment1 <--"Interconnect"--> Segment4
    Segment1 --"Outbound NAT"--> NAT
    Segment2 --"Outbound NAT"--> NAT
    Segment3 --"Outbound NAT"--> NAT
    Segment4 --"Outbound NAT"--> NAT

    %% --- Add the invisible link to force height ---
    top_pad ~~~ bottom_pad

    %% --- Style the VERY LAST link (the 14th one, index 13) to be invisible ---
    linkStyle 13 stroke-width:0px
```

-----

## ✅ Prerequisites

Before you begin, ensure you have the following installed and configured:

1.  **AWS Account:** An active AWS account with permissions to create the resources listed above.
2.  **AWS CLI:** The AWS Command Line Interface installed and configured with your credentials. The `deploy.sh` script specifically uses a named profile.
3.  **Terraform:** Terraform CLI (version 1.0 or later) installed.
4.  **SSH Key Pair:** A public/private SSH key pair that you will use when connecting to the cluster. If you don't have one, you can generate it with `ssh-keygen -t rsa`.
5.  **EDB Repo Token:** You need a valid EDB repository token to download WarehousePG. This is passed as a sensitive variable.
5.  **Public IP Address of your Jumpbox host:** You will need to designate a single public ip address that you will `ssh` to the cluster from.

-----

## 🚀 Deployment Steps

The deployment process is broken down into two main phases:

1.  **Infrastructure Provisioning** using the `deploy.sh` script and Terraform.
2.  **Cluster Initialization** using the `setup_whpg.sh` script on the coordinator node.

### Phase 1: Infrastructure Provisioning

1.  **Clone the Repository**

    ```bash
    git clone https://github.com/bluethumpasaurus/terraform-whpg7-5node-cluster-aws-nfs-nostandby
    cd terraform-whpg7-5node-cluster-aws-nfs-nostandby
    ```

2.  **Run the Deployment Script**

    Make the `deploy.sh` script executable and run it. This script will guide you through the configuration process.

    ```bash
    chmod +x deploy.sh
    ./deploy.sh
    ```
    
    **Note** that before executing the `deploy.sh` script, you will need to run through the interactive wizard for the AWS Command Line Interface (CLI) that sets up your terminal to authenticate using AWS IAM Identity Center:

    ```bash
    aws configure sso --profile <your AWS IAM Profile Name>
    ```


3.  **Provide Configuration Details**
\
    The script will prompt you for the following information. You can press Enter to accept the default values in brackets.

      * `Enter your AWS IAM Profile Name`: The named AWS profile to use for authentication.
      * `Enter the AWS Region`: The AWS region for deployment.
      * `Enter the EC2 Instance Type`: The instance size for all four nodes.
      * `Enter a unique name for your cluster`: A prefix for all created resources.
      * `Enter the ip address to be used for SSH`: This will limit `ssh` access to ONLY this ip address.
      * `Enter the path to the SSH PUBLIC KEY`: The path to your `.pub` file.
      * `Enter the path to the SSH PRIVATE KEY`: The path to your private key file.
      * `Enter your EDB Repo Token`: Your secret token from EDB.

4.  **Review and Apply Terraform Plan**

    The script will initialize Terraform (`terraform init`) and show you an execution plan (`terraform plan`). Review the plan and, when prompted, type `yes` to create the resources in AWS with `terraform apply`.

    Typing `no` to the prompt will exit the process, and will give instructions on how to manually perform the  `terraform apply` later.


6.  **📋 Gather Terraform Outputs**

    Once the deployment is complete, Terraform will display a list of outputs similar to those shown below. 

    Copy the `ssh_command_for_whpg_coordinator` outputs, as you will need this to `ssh` to the Coordinator host from your jumpbox.

    ```
    Outputs:

    coordinator_private_ip = "10.0.1.100"
    segment_server_1_private_ip = "10.0.2.200"
    segment_server_2_private_ip = "10.0.2.201"
    segment_server_3_private_ip = "10.0.2.202"
    segment_server_4_private_ip = "10.0.2.203"
    ssh_command_for_whpg_coordinator = "ssh -i ~/.ssh/id_rsa rocky@<coordinator_public_ip>"
    ```

### Phase 2: Cluster Initialization

1.  **SSH into the Coordinator Node**

    On the jumpbox, use the `ssh_command_for_whpg_coordinator` command from the Terraform output to connect to the primary coordinator server. 

    You will log in as the `rocky` user.

    ```bash
    # Use the command from your output
    ssh -i ~/.ssh/id_rsa rocky@<coordinator_public_ip>
    ```

2.  **Switch to the `gpadmin` User**

    The repo's `configure-instance.sh.tpl` script created a `gpadmin` user. Switch to this user to perform the WarehousePG cluster initialisation tasks. 


    ```bash
    sudo su - gpadmin
    ```


3.  **Run the Cluster Initialization Script**

    There will be a WarehousePG cluster initialisation script located at `/home/gpadmin/setup_whpg.sh`. 

    `source` the script to run it - this will set up passwordless SSH, create data directories, initialize the WarehousePG database system, and run a `psql` command at completion to verify the created cluster.


    ```bash
    source setup_whpg.sh
    ```

    The script will take a few minutes to complete.

4.  **Verification**

    The script will automatically run a `psql` query at completion to verify that all segments are configured correctly - each segment host will have 3 primary segments & 3 mirror segments.

-----

## 💻 Connecting to the Database

Once the setup is complete, you can connect to your new WarehousePG database.

1.  Ensure you are logged into the **coordinator node** as the **`gpadmin`** user.

2.  The `/home/gpadmin/setup_whpg.sh` script will have configured the `.bashrc` file with the necessary environment variables. Simply run `psql`:

    ```bash
    psql
    ```

You are now connected to the `aws_db` database and can begin creating tables and loading data.


-----

## 💿 A cluster-wide NFS share is pre-created on the Coordinator

The Coordinator's NFS share is mounted at `/mnt/backups` for all hosts. This allows for testing of database backups to NFS with the WarehousePG `gpbackup` utility.

The following command will perform a backup of the pre-created `aws_db` database to the NFS share - the `gpbackup` and `gprestore` utilities are pre-installed on the cluster.


```
gpbackup -dbname aws_db --single-backup-dir --backup-dir /mnt/backups
```

The following command shows the NFS share configuration for the cluster:

```
[gpadmin@ip-10-0-1-100 ~]$ gpssh -f /home/gpadmin/all_hosts "df -h | grep /mnt/backups"
[ip-10-0-2-200.eu-west-2.compute.internal] 10.0.1.100:/srv/nfs/backups  9.0G  2.2G  6.8G  24% /mnt/backups
[ip-10-0-2-203.eu-west-2.compute.internal] 10.0.1.100:/srv/nfs/backups  9.0G  2.2G  6.8G  24% /mnt/backups
[ip-10-0-2-201.eu-west-2.compute.internal] 10.0.1.100:/srv/nfs/backups  9.0G  2.2G  6.8G  24% /mnt/backups
[ip-10-0-2-202.eu-west-2.compute.internal] 10.0.1.100:/srv/nfs/backups  9.0G  2.2G  6.8G  24% /mnt/backups
[ip-10-0-1-100.eu-west-2.compute.internal] 10.0.1.100:/srv/nfs/backups  9.0G  2.2G  6.8G  24% /mnt/backups
[gpadmin@ip-10-0-1-100 ~]$
```


-----

## 💿 Setting up MinIO 

MinIO Server is pre-installed on segment host 1, with a MinIO drive setup at `/data/minio-storage/` . The `mc` client is pre-installed on the Coordinator node.



1.  Ensure you are logged into the **coordinator node** as the **`gpadmin`** user.

2.  Set up an `mc` alias to connect to the MinIO server on **segment host 1**.

    ```bash
    mc alias set minio_seg1 http://10.0.2.200:9000/ minioadmin minioadmin
    ```

3.  With `mc` confirm the connection to the MinIO server on the **segment host 1**.

    ```bash
    mc admin info minio_seg1
    ```

----

## 🧹 Clean Up

To avoid ongoing AWS charges, you can destroy all the resources created by this project. From your local machine (where you ran `./deploy.sh`), execute:

```bash
terraform destroy
```

Type `yes` when prompted to confirm the deletion.