# Deploy an HTTPS web application on Kubernetes with Citrix ingress controller and HashiCorp Vault using cert-manager

For ingress resources deployed with the Citrix ingress controller, you can automate TLS certificate provisioning, revocation, and renewal using cert-manager and HashiCorp Vault. This topic provides a sample workflow that uses HashiCorp Vault as a self-signed certificate authority for certificate signing requests from cert-manager.

Specifically, the workflow uses the Vault PKI Secrets Engine to create a certificate authority (CA). This tutorial assumes that you have a Vault server installed and reachable from the Kubernetes cluster. The PKI secrets engine of Vault is suitable for internal applications. For external facing applications that require public trust, see [automating TLS certificates using Let’s Encrypt CA](./acme.md).

The workflow uses a Vault secret engine and authentication methods. For the full list of Vault features, see the following Vault documentation:

-  [Vault Secrets Engines](https://www.vaultproject.io/docs/secrets/index.html)

-  [Vault Authentication Methods](https://www.vaultproject.io/docs/auth/index.html)

This topic provides you information on how to deploy an HTTPS web application on a Kubernetes cluster, using:

-  Citrix ingress controller
-  JetStack's [cert-manager](https://cert-manager.io/docs/) to provision TLS certificates from [HashiCorp Vault](https://www.vaultproject.io/)
-  [HashiCorp Vault](https://www.vaultproject.io/)

## Prerequisites

Ensure that you have:

-  The Vault server is installed, unsealed, and is reachable from the Kubernetes cluster. For information on installing the Vault server, see the Vault installation documentation.

-  Enabled RBAC on your Kubernetes cluster.

-  Deployed Citrix ADC MPX, VPX, or CPX in Tier 1 or Tier 2 deployment model.

    In the Tier 1 deployment model, Citrix ADC MPX or VPX is used as an Application Delivery Controller (ADC). The Citrix ingress controller running in the Kubernetes cluster configures the virtual services for the services running on the Kubernetes cluster. Citrix ADC runs the virtual service on the publicly routable IP address and offloads SSL for client traffic with the help of the Let's Encrypt generated certificate.

    In the Tier 2 deployment, a TCP service is configured on the Citrix ADC (VPX/MPX) running outside the Kubernetes cluster to forward the traffic to Citrix ADC CPX instances running in the Kubernetes cluster. Citrix ADC CPX ends the SSL session and load-balances the traffic to actual service pods.

-  Deployed Citrix ingress controller. See [Deployment Topologies](../deployment-topologies.md) for various deployment scenarios.

-  Administrator permissions for all the deployment steps. If you encounter failures due to permissions, make sure that you have the administrator permission.

**Note:** The following procedure shows steps to configure Vault as a certificate authority with Citrix ADC CPX used as the ingress device. When a Citrix ADC VPX or MPX is used as the ingress device, the steps are the same except the steps to verify the ingress configuration in the Citrix ADC.


## Deploy cert-manager using the manifest file

Perform the following steps to deploy cert-manager using the supplied YAML manifest file.

1.  Install cert-manager. For information on installing cert-manager, see the [cert-manager documentation](https://cert-manager.io/docs/installation/kubernetes/).

        kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/vx.x.x/cert-manager.yaml

    You can also install cert-manager with Helm. For more information, see the [cert-manager documentation](https://cert-manager.io/docs/installation/kubernetes/#installing-with-helm).

1.  Verify that cert-manager is up and running using the following command.

        % kubectl -n cert-manager get all
        NAME                                       READY   STATUS    RESTARTS   AGE
        pod/cert-manager-77fd74fb64-d68v7          1/1     Running   0          4m41s
        pod/cert-manager-webhook-67bf86d45-k77jj   1/1     Running   0          4m41s

        NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
        service/cert-manager-webhook   ClusterIP   10.108.161.154   <none>        443/TCP   13d

        NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/cert-manager           1/1     1            1           13d
        deployment.apps/cert-manager-webhook   1/1     1            1           13d

        NAME                                             DESIRED   CURRENT   READY   AGE
        replicaset.apps/cert-manager-77fd74fb64          1         1         1       13d
        replicaset.apps/cert-manager-webhook-67bf86d45   1         1         1       13d

        NAME                                                COMPLETIONS   DURATION   AGE
        job.batch/cert-manager-webhook-ca-sync              1/1           22s        13d
        job.batch/cert-manager-webhook-ca-sync-1549756800   1/1           21s        10d
        job.batch/cert-manager-webhook-ca-sync-1550361600   1/1           19s        3d8h

        NAME                                         SCHEDULE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
        cronjob.batch/cert-manager-webhook-ca-sync   @weekly    False     0        3d8h            13d

## Deploy a sample web application

Perform the following steps to deploy a sample web application.

!!! note "Note"
    [Kuard](https://github.com/kubernetes-up-and-running/kuard), a Kubernetes demo application is used for reference in this topic.

1.  Create a deployment YAML file (`kuard-deployment.yaml`) for Kuard with the following configuration.
```yml

        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: kuard
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: kuard
          template:
            metadata:
              labels:
                app: kuard
            spec:
              containers:
              - image: gcr.io/kuar-demo/kuard-amd64:1
                imagePullPolicy: Always
                name: kuard
                ports:
                - containerPort: 8080
```

1.  Deploy the Kuard deployment file (`kuard-deployment.yaml`) to your cluster, using the following commands.

        % kubectl create -f kuard-deployment.yaml
        deployment.extensions/kuard created
        % kubectl get pod -l app=kuard
        NAME                     READY   STATUS    RESTARTS   AGE
        kuard-6fc4d89bfb-djljt   1/1     Running   0          24s

1.  Create a service for the deployment. Create a file called `service.yaml` with the following configuration.
```yml
        apiVersion: v1
        kind: Service
        metadata:
          name: kuard
        spec:
          ports:
          - port: 80
            targetPort: 8080
            protocol: TCP
          selector:
            app: kuard
```

1.  Deploy and verify the service using the following command.

        % kubectl create -f service.yaml
        service/kuard created
        % kubectl get svc kuard
        NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
        kuard   ClusterIP   10.103.49.171   <none>        80/TCP    13s

1.  Expose this service to the outside world by creating an Ingress that is deployed on Citrix ADC CPX or VPX as Content switching virtual server.

        **Note:**
        Ensure that you change `kubernetes.io/ingress.class` to your ingress class on which Citrix ingress controller is started.

```yml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          annotations:
            kubernetes.io/ingress.class: citrix
          name: kuard
        spec:
          rules:
          - host: kuard.example.com
            http:
              paths:
              - backend:
                  service:
                    name: kuard
                    port:
                      number: 80
                path: /
                pathType: Prefix
  ```

    !!! info "Important"
        Change the value of `spec.rules.host` to the domain that you control. Ensure that a DNS entry exists to route the traffic to Citrix ADC CPX or VPX.

1.  Deploy the Ingress using the following command.

        % kubectl apply -f ingress.yml
        ingress.extensions/kuard created
        root@ubuntu-vivek-225:~/cert-manager# kubectl get ingress
        NAME    HOSTS               ADDRESS   PORTS   AGE
        kuard   kuard.example.com             80      7s

1.  Verify if the ingress is configured on Citrix ADC CPX or VPX using the following command.

        kubectl exec -it cpx-ingress-5b85d7c69d-ngd72 /bin/bash
        root@cpx-ingress-5b85d7c69d-ngd72:/# cli_script.sh 'sh cs vs'
        exec: sh cs vs
        1) k8s-10.244.1.50:80:http (10.244.1.50:80) - HTTP Type: CONTENT
          State: UP
          Last state change was at Thu Feb 21 09:02:14 2019
          Time since last state change: 0 days, 00:00:41.140
          Client Idle Timeout: 180 sec
          Down state flush: ENABLED
          Disable Primary Vserver On Down : DISABLED
          Comment: uid=75VBGFO7NZXV7SCI4LSDJML2Q5X6FSNK6NXQPWGMDOYGBW2IMOGQ====
          Appflow logging: ENABLED
          Port Rewrite : DISABLED
          State Update: DISABLED
          Default: Content Precedence: RULE
          Vserver IP and Port insertion: OFF
          L2Conn: OFF Case Sensitivity: ON
          Authentication: OFF
          401 Based Authentication: OFF
          Push: DISABLED Push VServer:
          Push Label Rule: none
          Listen Policy: NONE
          IcmpResponse: PASSIVE
          RHIstate:  PASSIVE
          Traffic Domain: 0
        Done
        root@cpx-ingress-5b85d7c69d-ngd72:/# exit
        exit

1.  Verify if the page is correctly being served when requested using the `curl` command.

        % curl -sS -D - kuard.example.com -o /dev/null
        HTTP/1.1 200 OK
        Content-Length: 1458
        Content-Type: text/html
        Date: Thu, 21 Feb 2019 09:09:05 GMT


Once you have deployed the sample HTTP application, you can proceed to make the application available over HTTPS. Here the Vault server signs the CSR generated by the cert-manager and a server certificate is automatically generated for the application.

In the following procedure, you use the configured Vault as a certificate authority and configure the cert-manager to use the Vault as signing authority for the CSR.

## Configure HashiCorp Vault as Certificate Authority

In this procedure, you set up an intermediate CA certificate signing request using HashiCorp Vault. This Vault endpoint is used by the cert-manager to sign the certificate for the ingress resources.

### Prerequisites

Ensure that you have installed the `jq` utility.

### Create a root CA

For the sample workflow you can generate your own Root Certificate Authority within the Vault. In a production environment, you should use an external Root CA to sign the intermediate CA that Vault uses to generate certificates. If you have a root CA generated elsewhere, skip this step.

!!! note "Note"
    `PKI_ROOT` is a path where you mount the root CA, typically it is 'pki'. ${DOMAIN} in this procedure is `example.com`

```

% export DOMAIN=example.com
% export PKI_ROOT=pki

% vault secrets enable -path="${PKI_ROOT}" pki

# Set the max TTL for the root CA to 10 years
% vault secrets tune -max-lease-ttl=87600h "${PKI_ROOT}"

% vault write -format=json "${PKI_ROOT}"/root/generate/internal \
 common_name="${DOMAIN} CA root" ttl=87600h | tee \
>(jq -r .data.certificate > ca.pem) \
>(jq -r .data.issuing_ca > issuing_ca.pem) \
>(jq -r .data.private_key > ca-key.pem)

#Configure the CA and CRL URLs:

% vault write "${PKI_ROOT}"/config/urls \
       issuing_certificates="${VAULT_ADDR}/v1/${PKI_ROOT}/ca" \
       crl_distribution_points="${VAULT_ADDR}/v1/${PKI_ROOT}/crl"
```

### Generate an intermediate CA

After creating the root CA, perform the following steps to create an intermediate CSR using the root CA.

1.  Enable pki from a different path `PKI_INT` from root CA, typically `pki\_int`. Use the following command:


        % export PKI_INT=pki_int
        % vault secrets enable -path=${PKI_INT} pki

        # Set the max TTL to 3 year

        % vault secrets tune -max-lease-ttl=26280h ${PKI_INT}

2.  Generate CSR for `${DOMAIN}` that needs to be signed by the root CA. The key is stored internally to the Vault. Use the following command:

        % vault write -format=json "${PKI_INT}"/intermediate/generate/internal \
        common_name="${DOMAIN} CA intermediate" ttl=26280h | tee \
        >(jq -r .data.csr > pki_int.csr) \
        >(jq -r .data.private_key > pki_int.pem)

3.  Generate and sign the `${DOMAIN}` certificate as an intermediate CA using root CA, store it as `intermediate.cert.pem`. Use the following command:

        % vault write -format=json "${PKI_ROOT}"/root/sign-intermediate csr=@pki_int.csr
                format=pem_bundle ttl=26280h \
                | jq -r '.data.certificate' > intermediate.cert.pem

    If you are using an external root CA, skip the preceding step and sign the CSR manually using the root CA.

4.  Once the CSR is signed and the root CA returns a certificate, it needs to added back into the Vault using the following command:

        % vault write "${PKI_INT}"/intermediate/set-signed certificate=@intermediate.cert.pem

5.  Set the CA and CRL location using the following command.

        vault write "${PKI_INT}"/config/urls issuing_certificates="${VAULT_ADDR}/v1/${PKI_INT}/ca" crl_distribution_points="${VAULT_ADDR}/v1/${PKI_INT}/crl"

An intermediate CA is set up and can be used to sign certificates for ingress resources.

### Configure a role

A role is a logical name which maps to policies. An administrator can control the certificate generation through the roles.

Create a role for the intermediate CA that provides a set of policies for issuing or signing the certificates using this CA.

There are many configurations that can be configured when creating roles. For more information, see the [Vault role documentation](https://www.vaultproject.io/api/secret/pki/index.html#create-update-role).

For the workflow, create a role "***kube-ingress***" that allows you to sign certificates of `${DOMAIN}` and its subdomains with a TTL of 90 days.

    # with a Max TTL of 90 days
    vault write ${PKI_INT}/roles/kube-ingress \
              allowed_domains=${DOMAIN} \
              allow_subdomains=true \
              max_ttl="2160h" \
              require_cn=false

## Create Approle based authentication

After configuring an intermediate CA to sign the certificates, you need to provide an authentication mechanism for the cert-manager to use the Vault for signing the certificates. Cert-manager supports Approle authentication method which provides a way for the applications to access the Vault defined roles.

An "***AppRole***" represents a set of Vault policies and login constraints that must be met to receive a token with those policies. For more information on this authentication method, see the [Approle documentation](https://www.vaultproject.io/docs/auth/approle.html).

### Create an Approle

Create an Approle named "***Kube-role***". The `secret_id` for the cert-manager should not be expired to use this Approle for authentication. Hence, do not set a TTL or set it to 0.

    % vault auth enable approle

    % vault write auth/approle/role/kube-role token_ttl=0

### Associate a policy with the Approle

Perform the following steps to associate a policy with an Approle.

1.  Create a file `pki_int.hcl` with the following configuration to allow the signing endpoints of the intermediate CA.

        path "${PKI_INT}/sign/*" {
              capabilities = ["create","update"]
            }

2.  Add the file to a new policy called `kube_allow_sign` using the following command.

        vault policy write kube-allow-sign pki_int.hcl

3.  Update this policy to the Approle using the following command.

        vault write auth/approle/role/kube-role policies=kube-allow-sign

The `kube-role` approle allows you to sign the CSR with intermediate CA.

### Generate the role id and secret id

The role id and secret id are used by the cert-manager to authenticate with the Vault.

Generate the role id and secret id and encode the secret id with Base64. Perform the following:

    % vault read auth/approle/role/kube-role/role-id
    role_id     db02de05-fa39-4855-059b-67221c5c2f63

    % vault write -f auth/approle/role/kube-role/secret-id
    secret_id               6a174c20-f6de-a53c-74d2-6018fcceff64
    secret_id_accessor      c454f7e5-996e-7230-6074-6ef26b7bcf86

    # encode secret_id with base64
    % echo 6a174c20-f6de-a53c-74d2-6018fcceff64 | base64
    NmExNzRjMjAtZjZkZS1hNTNjLTc0ZDItNjAxOGZjY2VmZjY0Cg==


## Configure issuing certificates in Kubernetes


After you have configured Vault as the intermediate CA, and the Approle authentication method for the cert-manager to access Vault, you need to configure the certificate for the ingress.

### Create a secret with Approle secret id

Perform the following to create a secret with Approle secret id.

1.  Create a secret file called `secretid.yaml` with the following configuration.

        apiVersion: v1
        kind: Secret
        type: Opaque
        metadata:
          name: cert-manager-vault-approle
          namespace: cert-manager
        data:
          secretId: "NmExNzRjMjAtZjZkZS1hNTNjLTc0ZDItNjAxOGZjY2VmZjY0Cg=="

    !!! note "Note"
        `data.secretId` is the base64 encoded secret Id generated in [Generate the role id and secret id](#generate-the-role-id-and-secret-id). If you are using an Issuer resource in the next step, the secret must be in the same namespace as the `Issuer`. For `ClusterIssuer`, the secret must be in the `cert-manager` namespace.

2.  Deploy the secret file (`secretid.yaml`) using the following command.

        % kubectl create -f secretid.yaml

### Deploy the Vault cluster issuer

The cert-manager supports two different CRDs for configuration, an `Issuer`, which is scoped to a single namespace, and a `ClusterIssuer`, which is cluster-wide. For the workflow, you need to use `ClusterIssuer`.

Perform the following steps to deploy the Vault cluster issuer.

1.  Create a file called `issuer-vault.yaml` with the following configuration.

        apiVersion: cert-manager.io/v1
        kind: ClusterIssuer
        metadata:
          name: vault-issuer
        spec:
          vault:
            path: pki_int/sign/kube-ingress
            server: <vault-server-url>
            #caBundle: <base64 encoded caBundle PEM file>
            auth:
              appRole:
                path: approle
                roleId: "db02de05-fa39-4855-059b-67221c5c2f63"
                secretRef:
                  name: cert-manager-vault-approle
                  key: secretId

    `SecretRef` is the Kubernetes secret name created in the previous step. Replace `roleId` with the `role_id` retrieved from the Vault.
    An optional base64 encoded caBundle in PEM format can be provided to validate the TLS connection to the Vault Server. When caBundle is set it replaces the CA bundle inside the container running the cert-manager. This parameter has no effect if the connection used is in plain HTTP.

2.  Deploy the file (`issuer-vault.yaml`) using the following command.

        % kubectl create -f issuer-vault.yaml

3.  Using the following command verify if the Vault cluster issuer is successfully authenticated with the Vault.

        % kubectl describe clusterIssuer vault-issuer  | tail -n 7
          Conditions:
            Last Transition Time:  2019-02-26T06:18:40Z
            Message:               Vault verified
            Reason:                VaultVerified
            Status:                True
            Type:                  Ready
        Events:                    <none>

Now, you have successfully setup the cert-manager for Vault as the CA. The next step is securing the ingress by generating the server certificate. There are two different options for securing your ingress. You can proceed with one of the approaches to secure your ingresses.

-	Ingress Shim approach
- Manually creating the `certificate` CRD object for the certificate.


### Ingress-shim approach

In this approach, you modify the ingress annotation for the cert-manager to automatically generate the certificate for the given host name and store it in the specified secret.


1.	Modify the ingress with the `tls` section specifying a host name and secret. Also, specify the cert-manager annotation `cert-manager.io/cluster-issuer` as follows.
```yml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          annotations:
            cert-manager.io/cluster-issuer: vault-issuer
            kubernetes.io/ingress.class: citrix
          name: kuard
        spec:
          rules:
          - host: kuard.example.com
            http:
              paths:
              - backend:
                  service:
                    name: kuard-service
                    port:
                      number: 80
                path: /
                pathType: Prefix
          tls:
          - hosts:
            - kuard.example.com
            secretName: kuard-example-tls
```


1. Deploy the modified ingress as follows.

          % kubectl apply -f ingress.yml
          ingress.extensions/kuard created

          % kubectl get ingress kuard
          NAME    HOSTS               ADDRESS   PORTS     AGE
          kuard   kuard.example.com             80, 443   12s

   This step triggers a `certificate` object by the cert-manager which creates a certificate signing request (CSR) for the domain `kuard.example.com`. On successful signing of CSR, the certificate is stored in the secret name `kuard-example-tls` specified in the ingress.

1. Verify that the certificate is successfully issued using the following command.

          % kubectl describe certificates kuard-example-tls  | grep -A5 Events
          Events:
          Type    Reason      Age   From          Message
          ----    ------      ----  ----          -------
          Normal  CertIssued  48s   cert-manager  Certificate issued successfully


### Create a `certificate` CRD object for the certificate

Once the issuer is successfully registered, you need to get the certificate for the ingress domain `kuard.example.com`.

You need to create a `certificate` resource with the `commonName` and `dnsNames`. For more information, see [cert-manager documentation](https://cert-manager.io/docs/usage/certificate/). You can specify multiple dnsNames which are used for the SAN field in the certificate.

To create a "certificate" CRD object for the certificate, perform the following:

1.  Create a file called `certificate.yaml` with the following configuration.

```yml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: kuard-example-tls
          namespace: default
        spec:
          secretName: kuard-example-tls
          issuerRef:
            kind: ClusterIssuer
            name: vault-issuer
          commonName: kuard.example.com
          duration: 720h
          #Renew before 7 days of expiry
          renewBefore: 168h
          commonName: kuard.example.com
          dnsNames:
          - www.kuard.example.com
  ```

    The certificate has CN=`kuard.example.com` and SAN=`Kuard.example.com,www.kuard.example.com`.
    `spec.secretName` is the name of the secret where the certificate is stored after the certificate is issued successfully.

2.  Deploy the file (`certificate.yaml`) on the Kubernetes cluster using the following command.

       % kubectl create -f certificate.yaml
       certificate.certmanager.k8s.io/kuard-example-tls created

### Verify if the certificate is issued

You can watch the progress of the certificate as it is issued using the following command:

    % kubectl describe certificates kuard-example-tls  | grep -A5 Events
    Events:
      Type    Reason      Age   From          Message
      ----    ------      ----  ----          -------
      Normal  CertIssued  48s   cert-manager  Certificate issued successfully

!!! info "Important"
    You may encounter some errors due to the Vault policies. If you encounter any such errors, return to the Vault and fix it.

After successful signing, a `kubernetes.io/tls` secret is created with the `secretName` specified in the `Certificate` resource.

    % kubectl get secret kuard-example-tls
    NAME                TYPE                DATA   AGE
    kuard-exmaple-tls   kubernetes.io/tls   3      4m20s

### Modify the ingress to use the generated secret

Perform the following steps to modify the ingress to use the generated secret.

1.  Edit the original ingress and add a `spec.tls` section specifying the secret `kuard-example-tls` as follows.

```yml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          annotations:
            kubernetes.io/ingress.class: citrix
          name: kuard
        spec:
          rules:
          - host: kuard.example.com
            http:
              paths:
              - backend:
                  service:
                    name: kuard
                    port:
                      number: 80
                pathType: Prefix
                path: /
          tls:
          - hosts:
            - kuard.example.com
            secretName: kuard-example-tls
```

1.  Deploy the ingress using the following command.

        % kubectl apply -f ingress.yml
        ingress.extensions/kuard created

        % kubectl get ingress kuard
        NAME    HOSTS               ADDRESS   PORTS     AGE
        kuard   kuard.example.com             80, 443   12s

**Verify the Ingress configuration in Citrix ADC**

 Once the certificate is successfully generated, Citrix ingress controller uses this certificate for configuring the front-end SSL virtual server. You can verify it with the following steps.

1.  Log on to Citrix ADC CPX and verify if the Certificate is bound to the SSL virtual server.

        % kubectl exec -it cpx-ingress-668bf6695f-4fwh8 bash
        cli_script.sh 'shsslvs'
        exec: shsslvs
        1) Vserver Name: k8s-10.244.3.148:443:ssl
          DH: DISABLED
          DH Private-Key Exponent Size Limit: DISABLED Ephemeral RSA: ENABLED Refresh Count: 0
          Session Reuse: ENABLED Timeout: 120 seconds
          Cipher Redirect: DISABLED
          SSLv2 Redirect: DISABLED
          ClearText Port: 0
          Client Auth: DISABLED
          SSL Redirect: DISABLED
          Non FIPS Ciphers: DISABLED
          SNI: ENABLED
          OCSP Stapling: DISABLED
          HSTS: DISABLED
          HSTS IncludeSubDomains: NO
          HSTS Max-Age: 0
          SSLv2: DISABLED  SSLv3: ENABLED  TLSv1.0: ENABLED  TLSv1.1: ENABLED  TLSv1.2: ENABLED  TLSv1.3: DISABLED
          Push Encryption Trigger: Always
          Send Close-Notify: YES
          Strict Sig-Digest Check: DISABLED
          Zero RTT Early Data: DISABLED
          DHE Key Exchange With PSK: NO
          Tickets Per Authentication Context: 1
        Done

        root@cpx-ingress-668bf6695f-4fwh8:/# cli_script.sh 'shsslvs k8s-10.244.3.148:443:ssl'
        exec: shsslvs k8s-10.244.3.148:443:ssl

          Advanced SSL configuration for VServer k8s-10.244.3.148:443:ssl:
          DH: DISABLED
          DH Private-Key Exponent Size Limit: DISABLED Ephemeral RSA: ENABLED Refresh Count: 0
          Session Reuse: ENABLED Timeout: 120 seconds
          Cipher Redirect: DISABLED
          SSLv2 Redirect: DISABLED
          ClearText Port: 0
          Client Auth: DISABLED
          SSL Redirect: DISABLED
          Non FIPS Ciphers: DISABLED
          SNI: ENABLED
          OCSP Stapling: DISABLED
          HSTS: DISABLED
          HSTS IncludeSubDomains: NO
          HSTS Max-Age: 0
          SSLv2: DISABLED  SSLv3: ENABLED  TLSv1.0: ENABLED  TLSv1.1: ENABLED  TLSv1.2: ENABLED  TLSv1.3: DISABLED
          Push Encryption Trigger: Always
          Send Close-Notify: YES
          Strict Sig-Digest Check: DISABLED
          Zero RTT Early Data: DISABLED
          DHE Key Exchange With PSK: NO
          Tickets Per Authentication Context: 1
        , P_256, P_384, P_224, P_5216) CertKey Name: k8s-LMO3O3U6KC6WXKCBJAQY6K6X6JO Server Certificate for SNI

        7) Cipher Name: DEFAULT
          Description: Default cipher list with encryption strength >= 128bit
        Done

        root@cpx-ingress-668bf6695f-4fwh8:/# cli_script.sh 'sh certkey k8s-LMO3O3U6KC6WXKCBJAQY6K6X6JO'
        exec: sh certkey k8s-LMO3O3U6KC6WXKCBJAQY6K6X6JO
          Name: k8s-LMO3O3U6KC6WXKCBJAQY6K6X6JO Status: Valid,   Days to expiration:0
          Version: 3
          Serial Number: 524C1D9306F784A2F5277C05C2A120D5258D9A2F
          Signature Algorithm: sha256WithRSAEncryption
          Issuer:  CN=example.com CA intermediate
          Validity
            Not Before: Feb 26 06:48:39 2019 GMT
            Not After : Feb 27 06:49:09 2019 GMT
          Certificate Type: "Client Certificate" "Server Certificate"
          Subject:  CN=kuard.example.com
          Public Key Algorithm: rsaEncryption
          Public Key size: 2048
          Ocsp Response Status: NONE
          2) URI:http://127.0.0.1:8200/v1/pki_int/crl
          3) VServer name: k8s-10.244.3.148:443:ssl Server Certificate for SNI
        Done

      The HTTPS webserver is up with the vault signed certificate. Cert-manager automatically renews the certificate as specified in the `RenewBefore` parameter in the certificate, before expiry of the certificate.

      **Note:** The Vault signing of the certificate fails if the expiry of a certificate is beyond the expiry of the root CA or intermediate CA. You should ensure that the CA certificates are renewed in advance before the expiry.

1. Verify that the application is accessible using the HTTPS protocol.

        % curl -sS -D - https://kuard.example.com -k -o /dev/null
        HTTP/1.1 200 OK
        Content-Length: 1472
        Content-Type: text/html
        Date: Tue, 11 May 2021 20:39:23 GMT