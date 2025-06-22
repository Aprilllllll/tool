
# [Purpose]

This article will help you go through enabling workload identity in an AKS cluster and use it's benefit to run az login in a pod that use configured service account

# [Tutorial]

## [Enable workload identity Addon]
1. Create or update an AKS cluster with workload identity addon with flag '--enable-oidc-issuer' and '--enable-workload-identity':
<img width="1208" alt="1" src="https://github.com/user-attachments/assets/37557377-d2b2-4c4f-bbad-3b801f3c327e" />


2. Once done, get the oidc issuer url by ```#az aks show -g <group name> -n <cluster name> --query "oidcIssuerProfile.issuerUrl" -o tsv```
<img width="1076" alt="2" src="https://github.com/user-attachments/assets/b79ace12-9200-497c-97ce-1f3bb456139e" />

## [Create federation identity]

3. Create a managed identity by
```#az identity create --name <identity name> --resource-group <resource group> --location <location> --subscription <subscription>```
and record the client ID here

<img width="1207" alt="3" src="https://github.com/user-attachments/assets/020f9004-3c3b-4a2d-9a03-2772dc143a86" />

## [Create service account]

4. Create a service account with annotation azure.workload.identity/client-id: <managed identity client id>
<img width="693" alt="4" src="https://github.com/user-attachments/assets/21f07814-9082-4be1-8686-33fe640db557" />

5. Create federation identity credential so AKS can work as an OIDC issuer:
```#az identity federated-credential create --name <FEDERATED_IDENTITY_CREDENTIAL_NAME> --identity-name <managed identity name> --resource-group <group name> --issuer <aks oidc issuer url> --subject system:serviceaccount:<service account namespace>:<service account name> --audience api://AzureADTokenExchange```

<img width="1207" alt="5" src="https://github.com/user-attachments/assets/29bded20-7b31-4428-9b11-abd51e06a7f3" />


## [Make role assignment to allow login with MI]

6. Now create a role assignment of subscription Contributor for the managed identity bond with federation identity, so we can use it to login:

<img width="864" alt="6" src="https://github.com/user-attachments/assets/682c4dfa-81cd-4a0f-ba19-3ba5e02a1f69" />

## [Login with MI in pod shell]

7. By now we have everything setup, we can create a pod running azurecli to test if our federation identity and role assignment work, you may use this pod template

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: azurecli2
  namespace: default
  labels:
    azure.workload.identity/use: "true"
spec:
  containers:
  - name: azurecli
    image: mcr.microsoft.com/azure-cli:2.49.0
    command:
      - sleep
      - "infinity"
  serviceAccountName: <service account name>
```

Example:


<img width="427" alt="7" src="https://github.com/user-attachments/assets/7164defd-f25f-4078-87b3-0dc775dea59a" />


8. Enter pod shell:

<img width="581" alt="8" src="https://github.com/user-attachments/assets/a22f9060-d3b5-4042-af8d-980a11012bc0" />

9. Get the token at /var/run/secrets/azure/token/azure-identity-token

<img width="824" alt="9" src="https://github.com/user-attachments/assets/d4bab181-f34f-439f-a048-a127f57448d6" />

10. Use command to login
```#az login --service-principal -u <MI client ID> --tenant <Tenant ID> --federated-token <token from step 9>```

<img width="656" alt="10" src="https://github.com/user-attachments/assets/ecfea82c-7873-4b3a-a3e0-18d01edcd748" />

11. Login succeed:

<img width="549" alt="11" src="https://github.com/user-attachments/assets/25ebd264-8a38-426e-b4a0-5f9048a11077" />


# [Reference]

https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster
https://github.com/Azure/azure-cli/issues/24756


