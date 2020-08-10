# Kubernetes vault
## Инсталляция hashicorp vault HA в k8s
 - `git clone https://github.com/hashicorp/consul-helm.git kubernetes-vault/consul-helm`
 - `helm upgrade --install consul-helm kubernetes-vault/consul-helm`
 - `git clone https://github.com/hashicorp/vault-helm.git kubernetes-vault/vault-helm`
 - `helm upgrade --install vault kubernetes-vault/vault-helm -f kubernetes-vault/vault-helm.values.yaml`
 - `helm status vault`
```
NAME: vault
LAST DEPLOYED: 2020-08-09 18:52:09.06627 +0300 MSK
NAMESPACE: default
STATUS: deployed
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get vault
```
 - `kubectl get pod -l app.kubernetes.io/instance=vault`
 - `kubectl logs vault-0`

## Инициализируем vault
 - `kubectl exec -it vault-0 -- vault operator init --key-shares=1 --key-threshold=1`
```
Unseal Key 1: gNQ02e6RWtdMJkBxCarbEKy1W79LBOsbkv8X6BDQjGQ=

Initial Root Token: s.Pqobowqr573j334EZRCakBCn

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 1 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```
 - `kubectl logs vault-0`
 - `kubectl exec -it vault-0 -- vault status`
```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.4.2
HA Enabled         true
```

## Распечатаем vault
 - `kubectl exec -it vault-0 -- vault operator unseal`
 - `kubectl exec -it vault-1 -- vault operator unseal`
 - `kubectl exec -it vault-2 -- vault operator unseal`
 - `kubectl exec -it vault-0 -- vault status`
```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.4.2
Cluster Name    vault-cluster-3a236bf2
Cluster ID      134813ae-3058-3951-f92f-ec0918cdf446
HA Enabled      true
HA Cluster      https://vault-0.vault-internal:8201
HA Mode         active
```

## Залогинимся в vault
 - `kubectl exec -it vault-0 -- vault login`
```
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.Pqobowqr573j334EZRCakBCn
token_accessor       gLKkCySuGwscI4e2bOhPZUKC
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```
 - `kubectl exec -it vault-0 -- vault auth list`
```
Path      Type     Accessor               Description
----      ----     --------               -----------
token/    token    auth_token_96720f77    token based credentials
```

## Заведем секреты
 - `kubectl exec -it vault-0 -- vault secrets enable --path=otus kv`
 - `kubectl exec -it vault-0 -- vault secrets list --detailed`
 - `kubectl exec -it vault-0 -- vault kv put otus/otus-ro/config username='otus' password='asajkjkahs'`
 - `kubectl exec -it vault-0 -- vault kv put otus/otus-rw/config username='otus' password='asajkjkahs'`
 - `kubectl exec -it vault-0 -- vault read otus/otus-ro/config`
```
Key                 Value
---                 -----
refresh_interval    768h
password            asajkjkahs
username            otus
```
 - `kubectl exec -it vault-0 -- vault kv get otus/otus-rw/config`
```
====== Data ======
Key         Value
---         -----
password    asajkjkahs
username    otus
```

## Включим авторизацию черерз k8s
 - `kubectl exec -it vault-0 -- vault auth enable kubernetes`
 - `kubectl exec -it vault-0 -- vault auth list`
```
Path           Type          Accessor                    Description
----           ----          --------                    -----------
kubernetes/    kubernetes    auth_kubernetes_4d66077a    n/a
token/         token         auth_token_96720f77         token based credentials
```

## Создадим yaml для ClusterRoleBinding
## Создадим Service Account vault-auth и применим ClusterRoleBinding
 - `kubectl create serviceaccount vault-auth`
 - `kubectl apply --filename kubernetes-vault/vault-auth-service-account.yml`

## Подготовим переменные для записи в конфиг кубер авторизации
 - `export VAULT_SA_NAME=$(kubectl get sa vault-auth -o jsonpath="{.secrets[*]['name']}")`
 - `export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)`
 - `export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)`
 - `export K8S_HOST=$(kubectl cluster-info | grep 'Kubernetes master' | awk '/https/ {print $NF}' | sed 's/\x1b\[[0-9;]*m//g')`

Выражение `sed 's/\x1b\[[0-9;]*m//g'` удаляет из текста цветовые коды ANSI.

## Запишем конфиг в vault
 - `kubectl exec -it vault-0 -- vault write auth/kubernetes/config token_reviewer_jwt="$SA_JWT_TOKEN" kubernetes_host="$K8S_HOST" kubernetes_ca_cert="$SA_CA_CRT"`

## Создадим файл политики
## Создадим политку и роль в vault
 - `kubectl cp kubernetes-vault/otus-policy.hcl vault-0:/tmp`
 - `kubectl exec -it vault-0 -- vault policy write otus-policy /tmp/otus-policy.hcl`
 - `kubectl exec -it vault-0 -- vault write auth/kubernetes/role/otus bound_service_account_names=vault-auth bound_service_account_namespaces=default policies=otus-policy ttl=24h`

## Проверим как работает авторизация
 - Создадим под с привязанным сервис аккоунтом и установим туда curl и jq
```
kubectl run tmp -ti --rm  --serviceaccount=vault-auth --image alpine:3.7
apk add curl jq
```
 - Залогинимся и получим клиентский токен
```
VAULT_ADDR=http://vault:8200
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq
TOKEN=$(curl -k -s --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq '.auth.client_token' | awk -F\" '{print $2}')
```
 - Проверим чтение
```
curl --header "X-Vault-Token: $TOKEN" $VAULT_ADDR/v1/otus/otus-ro/config
curl --header "X-Vault-Token: $TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config
```
 - Проверим запись
```
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token: $TOKEN" $VAULT_ADDR/v1/otus/otus-ro/config
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token: $TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token: $TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config1
```
 - Почему мы смогли записать otus-rw/config1 но не смогли otus-rw/config?
   **Не хватает прав на "update"**. Политика поправлена.

## Use case использования авторизации через кубер
## Заберем репозиторий с примерами
 - `git clone https://github.com/hashicorp/vault-guides.git`
## Запускаем пример
 - `kubectl create configmap example-vault-agent-config --from-file=kubernetes-vault/configs-k8s/`
 - `kubectl get configmap example-vault-agent-config -o yaml`
 - `kubectl apply -f kubernetes-vault/example-k8s-spec.yml --record`
## Проверка
 - `kubectl exec vault-agent-example -ti -- curl http://localhost/index.html`
```
<html>
<body>
<p>Some secrets:</p>
<ul>
<li><pre>username: otus</pre></li>
<li><pre>password: asajkjkahs</pre></li>
</ul>

</body>
</html>
```

## Создадим CA на базе vault
 - Включим pki секретс
```
kubectl exec -it vault-0 -- vault secrets enable pki
kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki
kubectl exec -it vault-0 -- vault write -field=certificate pki/root/generate/internal common_name="exmaple.ru" ttl=87600h > kubernetes-vault/CA_cert.crt
```
 - Пропишем урлы для ca и отозванных сертификатов
```
kubectl exec -it vault-0 -- vault write pki/config/urls issuing_certificates="http://vault:8200/v1/pki/ca" crl_distribution_points="http://vault:8200/v1/pki/crl"
```
 - Создадим промежуточный сертификат
```
kubectl exec -it vault-0 -- vault secrets enable --path=pki_int pki
kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki_int
kubectl exec -it vault-0 -- vault write -format=json pki_int/intermediate/generate/internal common_name="example.ru Intermediate Authority" | jq -r '.data.csr' > kubernetes-vault/pki_intermediate.csr
```
 - Пропишем промежуточный сертификат в vault
```
kubectl cp kubernetes-vault/pki_intermediate.csr vault-0:/tmp
kubectl exec -it vault-0 -- vault write -format=json pki/root/sign-intermediate csr=@/tmp/pki_intermediate.csr format=pem_bundle ttl="43800h" | jq -r '.data.certificate' > kubernetes-vault/intermediate.cert.pem
kubectl cp kubernetes-vault/intermediate.cert.pem vault-0:/tmp
kubectl exec -it vault-0 -- vault write pki_int/intermediate/set-signed certificate=@/tmp/intermediate.cert.pem
```
 - Создадим роль для выдачи сертификатов
```
kubectl exec -it vault-0 -- vault write pki_int/roles/example-dot-ru allowed_domains="example.ru" allow_subdomains=true max_ttl="720h"
```
 - Создадим и отзовем сертификат
```
kubectl exec -it vault-0 -- vault write pki_int/issue/devlab-dot-ru common_name="gitlab.devlab.ru" ttl="24h"
```
```
Error writing data to pki_int/issue/devlab-dot-ru: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/pki_int/issue/devlab-dot-ru
Code: 400. Errors:

* unknown role: devlab-dot-ru
```
```
kubectl exec -it vault-0 -- vault write pki_int/issue/example-dot-ru common_name="gitlab.example.ru" ttl="24h"
```
```
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUC9PMV5N6n2jRlf16/OpGmpN/zgwwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMDA4MDkxNzMwNTZaFw0yNTA4
MDgxNzMxMjZaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANjdSUBUrOUS
VeKX/Vje+txpnYK2CXsB1jJCMDOKWwuBZGlNFCpWsinX3izzvQsDDmjlUgPuGNpk
sLoK12l4TtaVkC7z72mCxF6YmBl3V+ZQgzIxmdcNoKlADmD2np6v9WH9QP8jCkwF
YWw2bvUcgo7yxqRD8Fh32P3ukciNM1nkNeJnZAWQADVpxrYKGbisDjZDNjW5zEnX
SlFL3+b8eXyrQ9J7VxBsBViogz9QCvaq3EVpoCv3MXqz+f90ab1y2Y0tZBUHDu9X
mO+pRXX1GLiViA2YYp2YSqHkTzVke/YAm6xi7zWtU8i2PZ/g18yZKW5vnEWD9Nqx
Uz+QrsXKqWMCAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUftHo4RgXpk9dNp0n8yusRLcuCUIwHwYDVR0jBBgwFoAU
lal7nd+Ap6c9TR1BYX1CsTVflnMwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
q1wmpxJCuDAw/CvAdrfrnjUoUTBsop7d0tfsNXOQun5rXflH25vJnl05TWR5Lp9O
sThNQUA99kGB1Sgs3ozpkAkqSkG6K35BTC7mWK9h2ABBqEDyjYUmxKZ0hdB3jLfA
kKLTWKh16V6XcSEJB9W3d/IyKHHP/c+/AAWB56nZ14fqup1EIj99liGHD9TftbTJ
m8rz93ZOafu17Tg+V8dGDWjy8Z6qDVHqWNMhRjRpkJZ1D2XeRcusUK2gYXhixmJb
25mkHV3sr3oGUfCLfDZnaCsBTZPP3FX+keFMliaVUVui+kqFtU4jM4vMx83C1m7O
7ChvTzETYTVK7pCBHNnR2Q==
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDZzCCAk+gAwIBAgIUed41F7rY+dZg1irMBvkmkmM5JiAwDQYJKoZIhvcNAQEL
BQAwLDEqMCgGA1UEAxMhZXhhbXBsZS5ydSBJbnRlcm1lZGlhdGUgQXV0aG9yaXR5
MB4XDTIwMDgwOTE3MzYxNFoXDTIwMDgxMDE3MzY0NFowHDEaMBgGA1UEAxMRZ2l0
bGFiLmV4YW1wbGUucnUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDj
0YLeiLOBriHFnXU0uIvjnbpQVkkE9g1XqD4RYA+2+BqosuqjLYx0oqxuJyxRVxWs
tOSFuxgcNn/2C5Jn6O2GqTYaNoqEbySbntOu+REWTx83skdUj8u2lL2emYVJ26iv
E8FeNlP06qrSV0kVfme4tNcDQ2N1VKx0Eo6vErivbMsRwD1viz+UFjtTE2cOrXKS
G9ISyLuhfj3a5FDOc4Ev6edGmDOYVVW2XIn8fNGpcQMpjau4DNqBv2a+QfVF3XbA
PU0jCRgBQJ2vyMf2/KK84WXs8W/NINzaUTEhv4wMNTXrlruonXhqLg2jPHSgz7Ms
MCLXjmWqTDTTtnRghA/nAgMBAAGjgZAwgY0wDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQUBTD6dvtwG/FdqejO
y6nYZkUU+xcwHwYDVR0jBBgwFoAUftHo4RgXpk9dNp0n8yusRLcuCUIwHAYDVR0R
BBUwE4IRZ2l0bGFiLmV4YW1wbGUucnUwDQYJKoZIhvcNAQELBQADggEBAKpNuRKq
xlz5lynxGEaEjFDE9yda/xcKtJGh70rQ90BU/i7KbGKoxXdnDO7OHuq2DNoCHIMx
HtlzEk5rlW3qbNWchY80EQILAALFoFYTxs6wETY6cOY0tSSTjNHxNbSfdA1DyVBa
LCxuM9WVMhEAA9reVTIcIo0ot3T097aDfgx9qObiDhUlzHKnhSEsnMw1e4qlUCuH
E+R0Rbce743wKZ4L7qlzoCAoJFo6RR9Y0O4+tl5Ax46y2xy7cz/Hrb5nzQkVKGK9
9bWGJLJbWuodG/GdMo603HT69HtZLg5Ao6MVbUcnsV5kw6ADVJAadtlI0W1NmW1I
vVrqnvA3ZVIxDPo=
-----END CERTIFICATE-----
expiration          1597081004
issuing_ca          -----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUC9PMV5N6n2jRlf16/OpGmpN/zgwwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMDA4MDkxNzMwNTZaFw0yNTA4
MDgxNzMxMjZaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANjdSUBUrOUS
VeKX/Vje+txpnYK2CXsB1jJCMDOKWwuBZGlNFCpWsinX3izzvQsDDmjlUgPuGNpk
sLoK12l4TtaVkC7z72mCxF6YmBl3V+ZQgzIxmdcNoKlADmD2np6v9WH9QP8jCkwF
YWw2bvUcgo7yxqRD8Fh32P3ukciNM1nkNeJnZAWQADVpxrYKGbisDjZDNjW5zEnX
SlFL3+b8eXyrQ9J7VxBsBViogz9QCvaq3EVpoCv3MXqz+f90ab1y2Y0tZBUHDu9X
mO+pRXX1GLiViA2YYp2YSqHkTzVke/YAm6xi7zWtU8i2PZ/g18yZKW5vnEWD9Nqx
Uz+QrsXKqWMCAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUftHo4RgXpk9dNp0n8yusRLcuCUIwHwYDVR0jBBgwFoAU
lal7nd+Ap6c9TR1BYX1CsTVflnMwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
q1wmpxJCuDAw/CvAdrfrnjUoUTBsop7d0tfsNXOQun5rXflH25vJnl05TWR5Lp9O
sThNQUA99kGB1Sgs3ozpkAkqSkG6K35BTC7mWK9h2ABBqEDyjYUmxKZ0hdB3jLfA
kKLTWKh16V6XcSEJB9W3d/IyKHHP/c+/AAWB56nZ14fqup1EIj99liGHD9TftbTJ
m8rz93ZOafu17Tg+V8dGDWjy8Z6qDVHqWNMhRjRpkJZ1D2XeRcusUK2gYXhixmJb
25mkHV3sr3oGUfCLfDZnaCsBTZPP3FX+keFMliaVUVui+kqFtU4jM4vMx83C1m7O
7ChvTzETYTVK7pCBHNnR2Q==
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEA49GC3oizga4hxZ11NLiL4526UFZJBPYNV6g+EWAPtvgaqLLq
oy2MdKKsbicsUVcVrLTkhbsYHDZ/9guSZ+jthqk2GjaKhG8km57TrvkRFk8fN7JH
VI/LtpS9npmFSduorxPBXjZT9Oqq0ldJFX5nuLTXA0NjdVSsdBKOrxK4r2zLEcA9
b4s/lBY7UxNnDq1ykhvSEsi7oX492uRQznOBL+nnRpgzmFVVtlyJ/HzRqXEDKY2r
uAzagb9mvkH1Rd12wD1NIwkYAUCdr8jH9vyivOFl7PFvzSDc2lExIb+MDDU165a7
qJ14ai4Nozx0oM+zLDAi145lqkw007Z0YIQP5wIDAQABAoIBAQC8o2zTyymoBYHd
WeYFA5KBpMbzYp8PxpWBscPDK2GXxZR9f7id6UdWBKT2iOU/bPZ7jUV0Hll2cwI9
v5M5CzwytsYfqm3D/yu22Cq7xWyKpnVY7vv1XyP1SPBB9SjS4Vmprpf85MtcDzvm
83OGoqZL4SHwh8pBCx3I9tzCxqO6TLFeysyftG/euRcxMxchmFNpr5Uh6jmhxXsO
cHQ2ic9PAq8/MEwh1VyyiHQxfpMVoaRGi7s3OFk+puPNiHIvgsiFWwf5P4RmvZI6
dQFb4GCqIQA73hmK03LrKz3cGhVc/ud+EfquMywBbWykq6MC+z54zrxE5BbfcwbZ
k4jXQQ4hAoGBAPMqhgdLwRb14AWoVhL1UBpNvXHIAuXzH16XUXOkNooZ1eJmTwZL
MmSGF4AMZYvqEuKe3xq3G3enT19SrEpp6a+43MlWCU+1YUdTeV5LJ7rMzF90HwDS
9nh9pAEXn8J0izySJymaOorQDdpXbpW2B2vk2vwhhlienQj49e4LOuK/AoGBAO/X
nv+8NQpYVOIb8uiJatUEQzV6eEMgkQ/8mLESDl3NIHQC+wLaRIc741+nthuoMR9h
XbDQZHZ0WagMmeyG617nR/8WMFsAA914QDkpZNpz60YPTwteE98z73a1+T98t2D9
AOnKnZDTJ1U5VEFLAj2A9Lhq6ILHHtyKfv8lCSTZAoGAf0yCr+0bn66GYc/Xh8M+
9RY/mAJSahlWEcn7zSNpnfCahRR0SGIzdmawhMt4mb+ntVXgjHbRfVlsdwWrxqUd
vm1zwD83TrAwxgtQHWoQ2Xz/fPUoieDnQPrdUekRLNagUcxdjiz8etEif2yIKv4J
cpVzgsz2LQyUPy8+aCke4bcCgYEAv0sb/t7u8wxWz2z5Rezsb3AR5uKCbw/Xg4e1
hW1gVgJYgw8pgzHxfGcQx+dtAQwZ+exfnLnpluzf4YADeLp3ml8fdl4NPVd6vba+
ipjwXqgcG+nz4p4rfVfgA6/KV4+yd0Hz64R2Pd+cPIYYJGeeJs3m4fwq7LvCaqZv
+jJg46kCgYEArUsD1lURdvBl1SbW0mNY/rajMFOb3DCvdyUY6D3PpgFkllepa1Yz
BkvnXznM+WiaxDPCBOVGyKbizxpijNrme87iBMOPFZ5uEQ+YNatuFgJCzkKMr+CS
3WGQq0rgRMd2wkxwMp3omiG9c15nZjDPR1SQWGCSXty/lh0PR/Bp59U=
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       79:de:35:17:ba:d8:f9:d6:60:d6:2a:cc:06:f9:26:92:63:39:26:20
```
```
kubectl exec -it vault-0 -- vault write pki_int/revoke serial_number="79:de:35:17:ba:d8:f9:d6:60:d6:2a:cc:06:f9:26:92:63:39:26:20"
```

## Включить TLS
### Как запустить проект:
```
openssl genrsa -out kubernetes-vault/vault_gke.key 4096
openssl req -config kubernetes-vault/vault_gke_csr.cnf -new -key kubernetes-vault/vault_gke.key -nodes -out kubernetes-vault/vault.csr
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: vaultcsr
spec:
  request: $(cat kubernetes-vault/vault.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
kubectl certificate approve vaultcsr
kubectl get csr vaultcsr -o jsonpath='{.status.certificate}'  | base64 --decode > kubernetes-vault/vault.crt
kubectl create secret tls vault-certs --cert=kubernetes-vault/vault.crt --key=kubernetes-vault/vault_gke.key

helm upgrade --install vault kubernetes-vault/vault-helm -f kubernetes-vault/vault-helm.https.values.yaml

kubectl delete pod -l app.kubernetes.io/instance=vault
kubectl exec -it vault-0 -- vault operator unseal
kubectl exec -it vault-1 -- vault operator unseal
kubectl exec -it vault-2 -- vault operator unseal
kubectl exec -it vault-0 -- vault login

kubectl attach tmp -ti
VAULT_ADDR=https://vault:8200
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
TOKEN=$(curl -k -s --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq '.auth.client_token' | awk -F\" '{print $2}')
```
### Как проверить работоспособность:
 - `curl -k --header "X-Vault-Token: $TOKEN" $VAULT_ADDR/v1/otus/otus-ro/config`
```
 - `curl -k --header "X-Vault-Token: $TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config`
```

## Настроить автообновление сертификатов
### Как запустить проект:
```
kubectl cp kubernetes-vault/pki-policy.hcl vault-0:/tmp
kubectl exec -it vault-0 -- vault policy write pki-policy /tmp/pki-policy.hcl
kubectl exec -it vault-0 -- vault write auth/kubernetes/role/pki bound_service_account_names=vault-auth bound_service_account_namespaces=default policies=pki-policy ttl=24h
kubectl apply -f kubernetes-vault/example-k8s-spec.https.yml
```
### Как проверить работоспособность:
 - `kubectl exec vault-inject-example -ti -- curl -kv https://localhost/index.html`
