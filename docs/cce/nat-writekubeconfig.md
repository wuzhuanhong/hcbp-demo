# 部署CCE集群并配置NAT网关实现公网访问

## 概述

华为云云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。本最佳实践将介绍如何使用Terraform自动化部署CCE集群，同时配置NAT网关实现节点内网访问公网，以及如何使用Terraform自动生成并保存kubeconfig文件。

### 应用场景

- 需要在私有网络中部署CCE集群但同时需要访问公网资源
- 需要在安全的环境中集中管理多个CCE集群的访问凭证
- 需要自动化部署和配置完整的容器基础设施
- 需要在部署流水线中自动获取和使用kubeconfig配置文件
- 需要实现集群访问与网络访问的统一管理
- 在混合云或多云环境中需要统一配置容器集群访问

### 方案优势

- 网络隔离安全：CCE节点通过NAT网关访问公网，无需暴露公网IP
- 自动化配置：使用Terraform自动生成并保存kubeconfig文件，无需手动下载和配置
- 资源复用：NAT网关可供多个节点和集群共用，优化公网IP资源
- 安全控制：通过统一NAT网关管理出口流量，实现精细化访问控制

### 涉及服务

- 云容器引擎（CCE）：提供容器集群管理服务
- 虚拟私有云（VPC）：提供隔离的网络环境
- NAT网关（NAT Gateway）：提供网络地址转换服务
- 弹性公网IP（EIP）：提供公网访问能力
- 统一身份认证服务（IAM）：提供身份认证和权限管理

## 资源/数据源设计

本最佳实践涉及以下主要资源和数据源：

### 数据源

1. **可用区（data.huaweicloud_availability_zones）**
   - 用途：获取可用的可用区信息，用于创建CCE节点

2. **VPC网络（data.huaweicloud_vpc）**
   - 用途：获取已存在的VPC网络信息，用于部署CCE集群和NAT网关

3. **VPC子网（data.huaweicloud_vpc_subnet）**
   - 用途：获取已存在的子网信息，用于部署CCE集群和NAT网关

### 资源

1. **弹性公网IP（huaweicloud_vpc_eip）**
   - 用途：为CCE集群和NAT网关提供公网访问能力

2. **NAT网关（huaweicloud_nat_gateway）**
   - 用途：为CCE集群节点提供公网访问能力

3. **SNAT规则（huaweicloud_nat_snat_rule）**
   - 用途：配置NAT网关的源网络地址转换规则

4. **CCE集群（huaweicloud_cce_cluster）**
   - 用途：部署和管理Kubernetes集群

5. **CCE节点（huaweicloud_cce_node）**
   - 用途：创建CCE集群的工作节点

6. **本地文件（local_file）**
   - 用途：保存CCE集群的kubeconfig文件到本地

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.myaz
    └── huaweicloud_cce_node.cce-node1/cce-node2/cce-node3

data.huaweicloud_vpc.myvpc
    ├── huaweicloud_cce_cluster.cluster
    └── huaweicloud_nat_gateway.nat_1

data.huaweicloud_vpc_subnet.mysubnet
    ├── huaweicloud_cce_cluster.cluster
    ├── huaweicloud_nat_gateway.nat_1
    └── huaweicloud_nat_snat_rule.snat_1

huaweicloud_vpc_eip.nat
    └── huaweicloud_nat_snat_rule.snat_1
    
huaweicloud_vpc_eip.cce
    └── huaweicloud_cce_cluster.cluster

huaweicloud_nat_gateway.nat_1
    └── huaweicloud_nat_snat_rule.snat_1

huaweicloud_cce_cluster.cluster
    ├── huaweicloud_cce_node.cce-node1
    ├── huaweicloud_cce_node.cce-node2
    ├── huaweicloud_cce_node.cce-node3
    └── local_file.kubeconfig
```

## 详细配置

### 数据源配置

#### 1. 可用区（data.huaweicloud_availability_zones）

获取指定region（默认继承当前provider块中所指定的region）下所有可用区信息，用于创建CCE节点。

```hcl
data "huaweicloud_availability_zones" "myaz" {}
```

**参数说明**：
- 无需参数配置，默认获取当前区域下所有可用区信息

#### 2. VPC网络（data.huaweicloud_vpc）

获取指定region（默认继承当前provider块中所指定的region）下已存在的VPC网络信息。

```hcl
variable "vpc_id" {
  description = "已存在的VPC ID"
  type        = string
}

data "huaweicloud_vpc" "myvpc" {
  id = var.vpc_id
}
```

**参数说明**：
- **id**：VPC ID，用于查询指定的VPC

#### 3. VPC子网（data.huaweicloud_vpc_subnet）

获取指定region（默认继承当前provider块中所指定的region）下已存在的子网信息。

```hcl
variable "subnet_id" {
  description = "已存在的子网ID"
  type        = string
}

data "huaweicloud_vpc_subnet" "mysubnet" {
  id = var.subnet_id
}
```

**参数说明**：
- **id**：子网ID，用于查询指定的子网

### 资源配置

#### 1. 弹性公网IP（huaweicloud_vpc_eip）

在指定region（默认继承当前provider块中所指定的region）下创建弹性公网IP，分别为CCE集群和NAT网关提供公网访问能力。

```hcl
resource "huaweicloud_vpc_eip" "cce" {
  publicip {
    type = "5_bgp"
  }
  bandwidth {
    name        = "cce-apiserver"
    size        = 20
    share_type  = "PER"
    charge_mode = "traffic"
  }
}

resource "huaweicloud_vpc_eip" "nat" {
  publicip {
    type = "5_bgp"
  }
  bandwidth {
    name        = "NAT"
    size        = 100
    share_type  = "PER"
    charge_mode = "traffic"
  }
}
```

**参数说明**：
- **publicip**：弹性公网IP配置
  - **type**：弹性公网IP类型，5_bgp表示动态BGP
- **bandwidth**：带宽配置
  - **name**：带宽名称
  - **size**：带宽大小（Mbit/s）
  - **share_type**：带宽共享类型，PER表示独享带宽
  - **charge_mode**：计费模式，traffic表示按流量计费

#### 2. NAT网关（huaweicloud_nat_gateway）

在指定region（默认继承当前provider块中所指定的region）下创建NAT网关，为CCE集群节点提供公网访问能力。

```hcl
variable "nat_name" {
  description = "NAT网关名称"
  type        = string
}

resource "huaweicloud_nat_gateway" "nat_1" {
  name      = var.nat_name
  spec      = "1"
  vpc_id    = data.huaweicloud_vpc.myvpc.id
  subnet_id = data.huaweicloud_vpc_subnet.mysubnet.id
}
```

**参数说明**：
- **name**：NAT网关名称
- **spec**：NAT网关规格，1表示小型（最多支持10,000个SNAT连接）
- **vpc_id**：VPC ID，指定NAT网关所属的VPC
- **subnet_id**：子网ID，指定NAT网关所属的子网

#### 3. SNAT规则（huaweicloud_nat_snat_rule）

在指定region（默认继承当前provider块中所指定的region）下创建SNAT规则，配置NAT网关的源网络地址转换规则。

```hcl
resource "huaweicloud_nat_snat_rule" "snat_1" {
  nat_gateway_id = huaweicloud_nat_gateway.nat_1.id
  floating_ip_id = huaweicloud_vpc_eip.nat.id
  subnet_id      = data.huaweicloud_vpc_subnet.mysubnet.id
}
```

**参数说明**：
- **nat_gateway_id**：NAT网关ID
- **floating_ip_id**：弹性公网IP ID
- **subnet_id**：子网ID，指定需要进行SNAT的子网

#### 4. CCE集群（huaweicloud_cce_cluster）

在指定region（默认继承当前provider块中所指定的region）下创建CCE集群，提供容器编排和管理能力。

```hcl
variable "cce_name" {
  description = "CCE集群名称"
  type        = string
}

resource "huaweicloud_cce_cluster" "cluster" {
  name                   = var.cce_name
  cluster_type           = "VirtualMachine"
  cluster_version        = "v1.19"
  flavor_id              = "cce.s1.small"
  vpc_id                 = data.huaweicloud_vpc.myvpc.id
  subnet_id              = data.huaweicloud_vpc_subnet.mysubnet.id
  container_network_type = "overlay_l2"
  authentication_mode    = "rbac"
  eip                    = huaweicloud_vpc_eip.cce.address
  delete_all             = "true"
}
```

**参数说明**：
- **name**：集群名称
- **cluster_type**：集群类型，VirtualMachine表示虚拟机类型
- **cluster_version**：集群版本，v1.19表示Kubernetes 1.19版本
- **flavor_id**：集群规格
- **vpc_id**：VPC ID，指定集群所属的VPC
- **subnet_id**：子网ID，指定集群所属的子网
- **container_network_type**：容器网络类型，overlay_l2表示叠加网络模式
- **authentication_mode**：认证模式，rbac表示基于角色的访问控制
- **eip**：弹性公网IP地址，用于访问集群控制面
- **delete_all**：删除集群时是否删除所有资源

#### 5. CCE节点（huaweicloud_cce_node）

在指定region（默认继承当前provider块中所指定的region）下创建CCE节点，作为集群的工作节点。

```hcl
variable "key_pair_name" {
  description = "节点使用的密钥对名称"
  type        = string
}

resource "huaweicloud_cce_node" "cce-node1" {
  cluster_id        = huaweicloud_cce_cluster.cluster.id
  name              = "node1"
  flavor_id         = "s6.large.2"
  availability_zone = data.huaweicloud_availability_zones.myaz.names[0]
  key_pair          = var.key_pair_name

  root_volume {
    size       = 80
    volumetype = "SAS"
  }
  data_volumes {
    size       = 100
    volumetype = "SAS"
  }
}

resource "huaweicloud_cce_node" "cce-node2" {
  cluster_id        = huaweicloud_cce_cluster.cluster.id
  name              = "node2"
  flavor_id         = "s6.large.2"
  availability_zone = data.huaweicloud_availability_zones.myaz.names[0]
  key_pair          = var.key_pair_name

  root_volume {
    size       = 80
    volumetype = "SAS"
  }
  data_volumes {
    size       = 100
    volumetype = "SAS"
  }
}

resource "huaweicloud_cce_node" "cce-node3" {
  cluster_id        = huaweicloud_cce_cluster.cluster.id
  name              = "node3"
  flavor_id         = "s6.large.2"
  availability_zone = data.huaweicloud_availability_zones.myaz.names[0]
  key_pair          = var.key_pair_name

  root_volume {
    size       = 80
    volumetype = "SAS"
  }
  data_volumes {
    size       = 100
    volumetype = "SAS"
  }
}
```

**参数说明**：
- **cluster_id**：CCE集群ID
- **name**：节点名称
- **flavor_id**：节点规格
- **availability_zone**：可用区
- **key_pair**：节点登录使用的密钥对名称
- **root_volume**：系统盘配置
  - **size**：系统盘大小（GB）
  - **volumetype**：系统盘类型，SAS表示高IO磁盘
- **data_volumes**：数据盘配置
  - **size**：数据盘大小（GB）
  - **volumetype**：数据盘类型，SAS表示高IO磁盘

#### 6. 保存kubeconfig文件（local_file）

当CCE集群创建成功后，自动将kubeconfig文件保存到本地，无需手动下载。

```hcl
resource "local_file" "kubeconfig" {
  content  = huaweicloud_cce_cluster.cluster.kube_config_raw
  filename = "$Local kubeconfig file path"
}
```

**参数说明**：
- **content**：文件内容，使用CCE集群的kube_config_raw属性
- **filename**：文件名称，指定保存kubeconfig文件的路径

## 部署流程

1. 创建弹性公网IP
2. 创建NAT网关
3. 配置SNAT规则
4. 创建CCE集群
5. 部署CCE节点
6. 保存kubeconfig文件
7. 使用kubectl连接并管理集群

## 操作步骤

1. **准备工作**
   - 安装Terraform
   - 配置华为云认证信息
   - 创建工作目录

2. **创建Terraform配置文件**
   ```bash
   touch main.tf
   touch variables.tf
   ```

3. **初始化和部署**
   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

4. **验证部署**
   - 检查kubeconfig文件是否生成
   - 使用kubectl测试集群连接
   ```bash
   kubectl config use-context external
   kubectl get nodes
   ```
   - 检查NAT网关是否正常工作
   - 验证节点是否可以访问公网

## 注意事项

1. **网络规划**
   - 确保VPC网段和容器网段不重叠
   - 合理规划子网地址空间，预留扩展空间
   - NAT网关的EIP应该有足够的带宽支持多节点访问

2. **安全配置**
   - 节点密钥对应妥善保管，确保私钥安全
   - kubeconfig文件包含敏感信息，应妥善保管
   - 考虑使用IAM权限控制对NAT网关的访问

3. **高可用设计**
   - 对于生产环境，考虑在多可用区部署节点
   - 考虑使用节点池实现自动扩缩容
   - 设置合理的NAT网关规格以支持业务流量

4. **成本优化**
   - 选择合适的NAT网关规格
   - 使用按流量计费的方式优化EIP成本
   - 合理配置节点规格以适应业务负载

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 自动化部署的CCE集群和NAT网关环境
2. 安全可控的公网访问方案
3. 自动生成和管理的kubeconfig文件
4. 可重复使用的Terraform配置
5. 符合最佳安全实践的网络架构

## 参考信息

- [CCE产品文档](https://support.huaweicloud.com/intl/zh-cn/cce/)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE最佳实践](https://github.com/huaweicloud/terraform-provider-huaweicloud/blob/master/examples/cce/nat-writekubeconfig) 
