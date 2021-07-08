---
title: "配置外部认证"
keywords: "LDAP, external, third-party, authentication"
description: "如何在 KubeSphere 上配置外部认证。"

linkTitle: "配置外部认证"
weight: 11000
---

本文介绍如何在 KubeSphere 中使用外部身份提供程序 (Identity Provider)，例如 LDAP 服务或活动目录 (Active Directory) 服务。

KubeSphere 提供内置的 OAuth 服务器，用户可以获取 OAuth 访问令牌用于对 KubeSphere API 进行身份认证。KubeSphere 管理员可以编辑 ConfigMap (`kubesphere-config`) 来配置 OAuth 并指定身份提供程序。

## 准备工作

您需要部署一个 Kubernetes 集群并在此集群上安装 KubeSphere。有关详细信息，请参见[在 Linux 上安装 KubeSphere](https://kubesphere.io/zh/docs/installing-on-linux/) 和[在 Kubernetes 上安装 KubeSphere](https://kubesphere.io/zh/docs/installing-on-kubernetes/)。


## 操作步骤

1. 以 `admin` 身份登录 KubeSphere 控制台，将鼠标移至右下角的 <img src="/images/docs/access-control-and-account-management/external-authentication/set-up-external-authentication/toolbox.png" width="25px"> 上，点击 **Kubectl**，执行以下命令来编辑 `kubesphere-config` ConfigMap：

   ```bash
   kubectl -n kubesphere-system edit cm kubesphere-config
   ```

2. 配置位于 `data:kubesphere.yaml:authentication` 部分的字段。

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
           - name: ldap
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

   字段说明如下：

   * `authenticateRateLimiterMaxTries`：`authenticateRateLimiterDuration` 指定的期间内可允许的最大连续登录失败次数。

   * `authenticateRateLimiterDuration`：`authenticateRateLimiterMaxTries` 适用的时间段。

   * `loginHistoryRetentionPeriod`：登录记录留存期限。过期的登录记录会被自动删除。

   * `maximumClockSkew`：时间敏感性操作（例如令牌过期验证）的最大时钟偏移。默认值为 `10s`。

   * `multipleLogin`：是否允许多个用户从不同位置同时登录。默认值为 `true`。

   * `jwtSecret`：签发用户令牌的密钥。在多集群环境中，所有集群必须使用[相同的密钥](https://kubesphere.io/zh/docs/multicluster-management/enable-multicluster/direct-connection/#prepare-a-member-cluster)。

   * `oauthOptions`：OAuth 设置。
     * `accessTokenMaxAge`：访问令牌有效期。对于多集群环境中的 Member 集群，默认值为 `0h`，即访问令牌永不过期。对于其他集群，默认值为 `2h`。
     * `accessTokenInactivityTimeout`：访问令牌空闲超时期限。该字段指定自访问令牌空闲至其失效的间隔时间。访问令牌超时后，用户需要获取新的访问令牌以重新获得访问权限。
     * `identityProviders`：身份提供程序。
       * `name`：身份提供程序的名称。
       * `type`：身份提供程序的类型。
       * `mappingMethod`：帐户映射方式。该值可以是 `auto` 或者 `lookup`。
         *  如果该值为 `auto`（默认），您需要指定一个新的用户名。KubeSphere 会根据用户名自动创建一个帐户，并将此帐户映射到第三方帐户。
         * 如果该值为 `lookup`，您需要执行第 3 步中的操作，手动将现有 KubeSphere 帐户映射到第三方帐户。
       * `provider`：身份提供程序的信息。此部分的字段根据身份提供程序的类型而异。

3. 如果 `mappingMethod` 设为 `lookup`，请执行以下命令并添加标签以将 KubeSphere 帐户映射到第三方帐户。如果 `mappingMethod` 设为 `auto`，可跳过此步骤。

   ```bash
   kubectl edit user <KubeSphere username>
   ```
   
   ```yaml
   labels:
     iam.kubesphere.io/identify-provider: <Identity provider name>
     iam.kubesphere.io/origin-uid: <Third-party username>
   ```
   
4. 配置好这些字段后，执行以下命令重启 ks-apiserver。

   ```bash
   kubectl -n kubesphere-system rollout restart deploy/ks-apiserver
   ```

