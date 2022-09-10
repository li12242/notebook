---
layout: post
title: AWS 运行 HPC 应用程序使用介绍
date: 2022-9-7
categories: tools
---

## 1. 简介

本文对 AWS 使用，特别是 Parallel Cluster 上运行 HPC 软件进行介绍。

## 2. 使用 AWS 建立并行集群

### 2.1. 准备

在使用 AWS 之前需要有一个 AWS 账户，可以直接使用默认创建的根账户建立集群，但是更好的方法是创建 IAM 账户来使用。

进入 AWS 控制台后，导航到 EC2（Elastic Compute Cloud）服务。EC2 虚拟服务器，也称为实例（Instance），是 AWS 上构建超算集群的基本模块。

### 2.2. 存储

Amazon S3（Simple Storage Service）是一种对象存储服务，可以用来存储任意数量的数据。在使用 AWS 超算集群过程中，S3 可以作为中间存储媒介，将本地数据保存在 AWS 中。

在创建 S3 时，需要设置 bucket 名称和存储区域。设置完成后，即可根据设置名称将文件上传到 AWS。上传可以使用 S3 控制台网页、AWS 命令行（CLI）或 AWS CloudShell 等方式上传。对于超过 10 GB 以上的大文件，推荐使用后两种方法。

在使用命令行模式时，通过以下命令可以检查创建的 S3 存储。

```bash
aws s3 ls
```

将文件复制到存储桶时，使用 `cp` 命令

```bash
aws s3 cp test.txt s3://bucketname/
```

### 2.3. ParallelCluster

ParallelCluster 解决方案架构由各种 AWS 服务组成，如下图所示。

#### 2.3.1. 创建密钥

可以通过如下命令在 AWS CloudShell 里创建密钥文件，下载到本地后即可根据此文件登录。

```bash
aws ec2 create-key-pair --key-name cfd_ireland --query KeyMaterial --output text > cfd_ireland.pem

chmod 600 cfd_ireland.pem
```

#### 2.3.2. ParallelCluster 安装和配置

使用 ParallelCluster 时，可以通过如下命令将其安装到本地。除此之外，也可以直接使用 AWS Cloudshell。

```bash
python3 -m pip install "aws-parallelcluster==3.0.3" --user
```

配置 ParallelCluster 时可以使用下面命令生成配置文件，并对其进行修改。

```
pcluster configure --config config
INFO: Configuration file test will be written.
Press CTRL-C to interrupt the procedure.


Allowed values for AWS Region ID:
1. ap-northeast-1
. . .
16. us-west-2
AWS Region ID [eu-west-1]: eu-west-1
Allowed values for EC2 Key Pair Name:
1. your-key
EC2 Key Pair Name [your-key]: #This is where you type the name of the key you previously generated (e.g. cfd_ireland) 
Allowed values for Scheduler:
1. slurm
2. awsbatch
Scheduler [slurm]: slurm
Allowed values for Operating System:
1. alinux2
2. centos7
3. ubuntu1804
4. ubuntu2004
Operating System [alinux2]: alinux2
Head node instance type [t2.micro]: c5n.large
Number of queues [1]: 1
Name of queue 1 [queue1]: compute
Number of compute resources for compute [1]: 1
Compute instance type for compute resource 1 in compute [t2.micro]: c5n.18xlarge
Maximum instance count [10]: 10
Automate VPC creation? (y/n) [n]: y
Allowed values for Availability Zone:
1. eu-west-1a
2. eu-west-1b
3. eu-west-1c
Availability Zone [eu-west-1a]: eu-west-1a
Allowed values for Network Configuration:
1. Head node in a public subnet and compute fleet in a private subnet
2. Head node and compute fleet in the same public subnet
Network Configuration [Head node in a public subnet and compute fleet in a private subnet]: 1
Beginning VPC creation. Please do not leave the terminal until the creation is finalized
Creating CloudFormation stack...
Do not leave the terminal until the process has finished.
Stack Name: parallelclusternetworking-pubpriv-20211116161450 (id: arn:aws:cloudformation:eu-west-1:008589934032:stack/parallelclusternetworking-pubpriv-20211116161450/680fea70-46f8-11ec-b10b-022a17eafb09)
Status: parallelclusternetworking-pubpriv-20211116161450 - CREATE_COMPLETE      
The stack has been created.
Configuration file written to config
You can edit your configuration file or simply run 'pcluster create-cluster --cluster-configuration config --cluster-name cluster-name --region eu-west-1' to create your cluster.
```

在配置结束后，AWS ParallelCluster 会将默认配置文件放入 `~/.parallelcluster` 目录内。

## 3. 附录

### 3.1. EC2 实例

AWS 有近 400 种实例类型，它们使用标准命名约定进行组织。以c5n.18xlarge实例为例，该实例名称表示：

* “C” 代表计算优化，其内存 (RAM) 与 CPU 的比率为 2-5（与具有更高 RAM 与 CPU 比率的 M 或 R 实例相反）。
* “5” 代表此特定实例的第五代（有一个基于老一代 Intel 芯片组的 C4 实例）。
* 网络优化的 “n” 标准，这意味着它支持 100 Gbit/s Elastic Fabric Adapter (EFA) 适配器。这是实现强大的扩展性以允许您的代码在数千个内核上运行所必需的。您可以在此处阅读有关 EFA的更多信息。 
* 此实例的“ 18xlarge ”是此实例的最大虚拟机选项。这意味着您是此特定实例（服务器）的唯一用户，并且可以获得它包含的最大网络、内存和处理器。

### Cloud9

在 AWS 中拥有众多服务，其中 Cloud9 是一个基于云的集成开发环境，只需使用浏览器即可完成对代码编写、运行和调试。

为了创建 Cloud9，需要在 AWS 管理面板中搜索栏找到 Cloud9，随后创建环境，对环境进行命名。
在配置界面，找到资源最少的实例以节约成本，创建完环境后即可准备就绪。
