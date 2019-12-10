# 创建自签根证书和中间证书

## 1. 创建CA配置文件

```shell
touch ca.cnf
```

填入  

```shell
[ policy_strict ]
countryName                         = match
stateOrProvinceName                 = match
organizationName                    = match
organizationalUnitName              = optional
commonName                          = supplied
emailAddress                        = optional

[ req ]
default_bits                        = 2048
default_md                          = sha256
default_keyfile                     = privkey.pem
distinguished_name                  = req_distinguished_name

[ req_distinguished_name ]
countryName                         = Country Name (2 letter code)
countryName_min                     = 2
countryName_max                     = 2
countryName_default                 = CN
stateOrProvinceName                 = State or Province Name (full name)
stateOrProvinceName_default         = Shanghai
localityName                        = Locality Name (eg, city)
localityName_default                = Hongkou
organizationName                    = Organization Name (eg, company)
organizationName_default            = Douzi
organizationalUnitName              = Organizational Unit Name (eg, section)
organizationalUnitName_default      = Douzi Root CA
commonName                          = Common Name (eg, fully qualified host name)
commonName_default                  = ca.og.gl
commonName_max                      = 64

[ v3_ext ]
subjectKeyIdentifier                = hash
authorityKeyIdentifier              = keyid:always,issuer:always
basicConstraints                    = critical,CA:TRUE
keyUsage                            = critical, digitalSignature, keyCertSign, cRLSign
```

## 2. 创建CA根证书

```shell
openssl genrsa -out CA_Root.key 2048
openssl req -new -key CA_Root.key -out CA_Root.csr -config ca.cnf
openssl x509 -sha256 -req -in CA_Root.csr -signkey CA_Root.key -out CA_Root.crt -days 3650 -extfile ca.cnf -extensions v3_ext
```

## 3. 创建CA中间证书

```shell
openssl genrsa -out CA_Intermediate.key 2048
openssl req -new -key CA_Intermediate.key -out CA_Intermediate.csr -config ca.cnf
openssl x509 -sha256 -req -in CA_Intermediate.csr -out CA_Intermediate.crt -CA CA_Root.crt -CAkey CA_Root.key -set_serial 01 -extfile ca.cnf -extensions v3_ext
```

## 4. 创建业务证书

1. 创建业务配置文件

   ```shell
   touch user.cnf
   ```

   填入  

   ```shell
   [ policy_strict ]
   countryName                         = match
   stateOrProvinceName                 = match
   organizationName                    = match
   organizationalUnitName              = optional
   commonName                          = supplied
   emailAddress                        = optional

   [ req ]
   default_bits                        = 2048
   default_md                          = sha256
   default_keyfile                     = privkey.pem
   distinguished_name                  = req_distinguished_name

   [ req_distinguished_name ]
   countryName                         = Country Name (2 letter code)
   countryName_min                     = 2
   countryName_max                     = 2
   countryName_default                 = CN
   stateOrProvinceName                 = State or Province Name (full name)
   stateOrProvinceName_default         = Shanghai
   localityName                        = Locality Name (eg, city)
   localityName_default                = Hongkou
   organizationName                    = Organization Name (eg, company)
   organizationName_default            = Douzi
   organizationalUnitName              = Organizational Unit Name (eg, section)
   organizationalUnitName_default      = Douzi Root CA
   commonName                          = Common Name (eg, fully qualified host name)
   commonName_default                  = ca.og.gl
   commonName_max                      = 64

   [ v3_ext ]
   subjectKeyIdentifier                = hash
   authorityKeyIdentifier              = keyid:always,issuer:always
   basicConstraints                    = critical,CA:FALSE
   keyUsage                            = critical, digitalSignature, keyEncipherment
   extendedKeyUsage                    = serverAuth, clientAuth
   ```

2. 创建业务证书

   ```shell
   openssl genrsa -out CA_Server.key 2048
   openssl req -new -key CA_Server.key -out CA_Server.csr -config user.cnf
   openssl x509 -sha256 -req -in CA_Server.csr -out CA_Server.crt -CA CA_Intermediate.crt -CAkey CA_Intermediate.key -CAcreateserial -extfile user.cnf -extensions v3_ext
   ```

## 5. 附录

```shell
# ----------------------- #
#         keyUsage        #
# ----------------------- #
# digitalSignature   - 数字签名
# nonRepudiation     - 不拒绝
# keyEncipherment    - 密钥加密
# dataEncipherment   - 数据加密
# keyAgreement       - 密钥协议
# keyCertSign        - 密钥证书签名
# cRLSign            - CRL签名
# encipherOnly       - 仅加密
# decipherOnly       - 仅解密

# ----------------------- #
#     extendedKeyUsage    #
# ----------------------- #
# serverAuth         - 服务器认证
# clientAuth         - 客户端认证
# codeSigning        - 代码签名
# emailProtection    - 电子邮件保护
# OCSPSigning        - OCSP签名
# timeStamping       - 时间戳
```
