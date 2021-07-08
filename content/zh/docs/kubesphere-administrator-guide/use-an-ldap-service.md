---
title: "使用 LDAP 服务"
keywords: "LDAP, identity provider, external, authentication"
description: "如何使用 LDAP 服务。"

linkTitle: "使用 LDAP 服务"
weight: 12000
---

本文介绍如何使用 LDAP 服务作为外部身份提供程序，从而通过 LDAP 服务来认证用户。

## 准备工作

* 您需要部署一个 Kubernetes 集群并在此集群上安装 KubeSphere。有关详细信息，请参见[在 Linux 上安装 KubeSphere](https://kubesphere.io/zh/docs/installing-on-linux/) 和[在 Kubernetes 上安装 KubeSphere](https://kubesphere.io/zh/docs/installing-on-kubernetes/)。
* 您需要获取 LDAP 服务的管理员识别名 (Distinguished Name) 和管理员密码。

### 操作步骤

1. 以 `admin` 身份登录 KubeSphere 控制台，将鼠标移至右下角的 <img src="/images/docs/access-control-and-account-management/external-authentication/use-an-ldap-service/toolbox.png" width="25px"> 上，点击 **Kubectl**，执行以下命令来编辑 `kubesphere-config` ConfigMap：

   ```bash
   kubectl -n kubesphere-system edit cm kubesphere-config
   ```

   示例：

   ```yaml
   apiVersion: v1
   data:
     kubesphere.yaml: |
       authentication:
         authenticateRateLimiterMaxTries: 10
         authenticateRateLimiterDuration: 10m0s
         loginHistoryRetentionPeriod: 168h
         maximumClockSkew: 10s
         multipleLogin: true
         jwtSecret: "********"
         oauthOptions:
           accessTokenMaxAge: 1h
           accessTokenInactivityTimeout: 30m
           identityProviders:
           - name: LDAP
             type: LDAPIdentityProvider
             mappingMethod: auto
             provider:
               host: 192.168.0.2:389
               managerDN: uid=root,cn=users,dc=nas
               managerPassword: ********
               userSearchBase: cn=users,dc=nas
               loginAttribute: uid
               mailAttribute: mail
   ```

2. 配置 `data:kubesphere.yaml:authentication` 部分中除 `oauthOptions:identityProviders` 之外的其他字段。有关详细信息，请参见[配置外部认证](../set-up-external-authentication/)。

3. 配置 `oauthOptions:identityProviders` 部分中的字段。

   * `name`：用户定义的 LDAP 服务名称。
   * `type`：要将 LDAP 服务用作身份提供程序，您必须将该值设为 `LDAPIdentityProvider`。
   * `mappingMethod`：帐户映射方法。该值可设为 `auto` 或 `lookup`。
     *  如果该值为 `auto`（默认），您需要指定一个新的用户名。KubeSphere 会根据用户名自动创建一个帐户，并将此帐户映射到 LDAP 用户。
     *  如果该值为 `lookup`，您需要执行第 4 步中的操作，手动将现有 KubeSphere 帐户映射到 LDAP 用户。
   * `provider`:
     * `host`：LDAP 服务的地址和端口号。
     * `managerDN`：用于绑定 LDAP 目录的识别名。
     * `managerPassword`：对应 `managerDN` 的密码。
     * `userSearchBase`：用户搜索基准。将该值设为所有 LDAP 用户都能查找到的目录层级下的识别名。
     * `loginAttribute`：该属性用于标识 LDAP 用户。
     * `mailAttribute`：该属性用于标识 LDAP 用户电子邮件地址。
   
4. 如果 `mappingMethod` 设为 `lookup`，请执行以下命令并添加标签以将 KubeSphere 帐户映射到 LDAP 用户。如果 `mappingMethod` 设为 `auto`，可跳过此步骤。

   ```bash
   kubectl edit user <KubeSphere username>
   ```

   ```yaml
   labels:
     iam.kubesphere.io/identify-provider: <LDAP service name>
     iam.kubesphere.io/origin-uid: <LDAP username>
   ```

5. 配置好这些字段后，执行以下命令重启 ks-apiserver。

   ```bash
   kubectl -n kubesphere-system rollout restart deploy/ks-apiserver
   ```
   
   {{< notice note >}}
   
   重启 ks-apiserver 期间，KubeSphere Web 控制台将不可用。请等待重启完成。
   
   {{</ notice >}}
   
6. 前往 KubeSphere 登录页面，输入 LDAP 用户的用户名和密码来登录。

   {{< notice note >}}

   LDAP 用户的用户名即 `loginAttribute` 指定的属性值。

   {{</ notice >}}
