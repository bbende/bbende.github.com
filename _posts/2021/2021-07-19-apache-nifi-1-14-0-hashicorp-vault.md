---
layout: post
title: "Apache NiFi 1.14.0 - HashiCorp Vault Integration"
description: ""
author: Joe Gresock
category: "Development"
tags: [NiFi,Vault]
---
{% include JB/setup %}

#### Guest author: Joe Gresock

With the release of Apache NiFi 1.14.0 come several new features, including encryption support from HashiCorp Vault.  In this post we look at how to encrypt sensitive properties in NiFi's configuration files using Vault.

### Introduction

How do you protect sensitive values in your Apache NiFi configuration files?  The [nifi-toolkit](http://nifi.apache.org/download.html) provides the [Encrypt-Config Tool](http://nifi.apache.org/docs/nifi-docs/html/toolkit-guide.html#encrypt_config_tool) to serve this purpose.  Traditionally, the Encrypt-Config Tool has used AES-GCM encryption, transforming an unprotected configuration file from this:
```
nifi.security.keystore=./conf/keystore.jks
nifi.security.keystoreType=JKS
nifi.security.keystorePasswd=myUnprotectedPassword:-(
nifi.security.keyPasswd=
nifi.security.truststore=./conf/truststore.jks
nifi.security.truststoreType=JKS
nifi.security.truststorePasswd=myOtherUnprotectedPassword)-:
```
to this:
```
nifi.security.keystore=./conf/keystore.jks
nifi.security.keystoreType=JKS
nifi.security.keystorePasswd=+UY1zFjnyeAwNIam||J/bKxNVjj+f5rjG7YjRB3+WVuHmx/+FDS/BpYtACVY5GKecfzWcfXQ
nifi.security.keystorePasswd.protected=aes/gcm/256
nifi.security.keyPasswd=
nifi.security.truststore=./conf/truststore.jks
nifi.security.truststoreType=JKS
nifi.security.truststorePasswd=cK36Q94q22mTZ6Yg||HlOtptD1KxHv3LovT2ClQw2H0CmzZClvhWxAtg8/EUz033cs1/cpE4LT42Im
nifi.security.truststorePasswd.protected=aes/gcm/256
```

Now, starting with Apache NiFi 1.14.0, it's possible to outsource encryption to HashiCorp Vault, using the [Transit Secrets Engine](https://www.vaultproject.io/docs/secrets/transit). In this post, we walk through integrating Vault into the Encrypt-Config Tool and NiFi itself.  Note that this post uses *nix commands, so although the process is the same on Windows, the commands will be different.

### Setup

To get started, we'll need to download and install HashiCorp Vault, NiFi 1.14.0, and nifi-toolkit.
* First, follow the relevant Vault installation guide for your system: [https://www.vaultproject.io/downloads](https://www.vaultproject.io/downloads)
* Then head over to the Apache nifi downloads page: [https://nifi.apache.org/download.html](https://nifi.apache.org/download.html)
  * Download `nifi-toolkit-1.14.0-bin.zip`
  * Download `nifi-1.14.0-bin.zip`

For the purposes of this guide, we'll unzip both `nifi-toolkit-1.14.0-bin.zip` and `nifi-1.14.0-bin.zip` in the same parent directory, so that you should see this:
```
$ ls
nifi-1.14.0    nifi-toolkit-1.14.0
```

We'll start Vault using development mode, which allows us to easily interact with the Vault server without having to [unseal](https://www.vaultproject.io/docs/concepts/seal) the server.  Run the following:
```
vault server -dev > init.log
```
This will launch Vault in development mode, and will write something like the following to init.log:
```
WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: fUJ6zxqlo+vsWFlCxMK9MKrhgpsmxdNla727mu+5nuY=
Root Token: s.3qsioJdA3YXptvEQURDCijUw

Development mode should NOT be used in production installations!
```

In another terminal, create a file containing this root token, which we'll use later:
```
TOKEN=$(grep "Root Token" init.log | cut -d ":" -f2 | xargs)
echo "vault.token=$TOKEN" > ~/vault-auth.properties
```

Run the following to enable the Transit Secrets Engine, which allows Vault to encrypt/decrypt values at runtime but not store them in the Vault server:
```
vault secrets enable transit
vault write -f transit/keys/nifi-transit
```

This last command created a Vault path called `"nifi-transit"`.  Take note of this value, because we'll be seeing it again later.

### Integrating Vault into nifi and nifi-tookit

Now that we have Vault running and configured, we'll configure nifi-toolkit to talk to Vault.  Since the toolkit operates on actual NiFi configuration files, that's where we'll start.

First, open up `nifi-1.14.0/conf/bootstrap-hashicorp-vault.conf` in your preferred editor, and configure it as follows:
```
# HTTP or HTTPS URI for HashiCorp Vault is required to enable the Sensitive Properties Provider
vault.uri=http://127.0.0.1:8200

# Transit Path is required to enable the Sensitive Properties Provider Protection Scheme 'hashicorp/vault/transit/{path}'
vault.transit.path=nifi-transit

# Token Authentication example properties
# vault.authentication=TOKEN
# vault.token=<token value>

# Optional file supports authentication properties described in the Spring Vault Environment Configuration
# https://docs.spring.io/spring-vault/docs/2.3.x/reference/html/#vault.core.environment-vault-configuration
#
# All authentication properties must be included in bootstrap-hashicorp-vault.conf when this property is not specified.
# Properties in bootstrap-hashicorp-vault.conf take precedence when the same values are defined in both files.
# Token Authentication is the default when the 'vault.authentication' property is not specified.
vault.authentication.properties.file=<full/path/to/vault-auth.properties>
```

Make sure you've specified the full path to the `vault-auth.properties` file that you created when installing Vault.  Notice that we could have actually stored the `vault.token` directly in the `bootstrap-hashicorp-vault.conf` file, but it's considered better practice to keep your authentication properties separate, in case you decide to rotate your Vault authentication token or other credentials.

At this point, we've integrated Vault into NiFi. Now copy this configuration file into nifi-toolkit and we'll be ready to start encrypting files:
```
cp nifi-1.14.0/conf/bootstrap-hashicorp-vault.conf nifi-toolkit-1.14.0/conf/
```

### Encrypting files using the HASHICORP_VAULT_TRANSIT protection scheme

Now that NiFi comes configured secure by default as of 1.14.0, we already have some sensitive values to protect in `nifi-1.14.0/conf/nifi.properties`, but we have to start NiFi first in order to generate them:

```
cd nifi-1.14.0
./bin/nifi.sh start
tail -f logs/nifi-app.log
```

Once you see the following, `Ctrl-C` from tailing the log and stop NiFi:
```
./bin/nifi.sh stop
```

Now there should be some passwords in our `nifi-1.14.0/conf/nifi.properties` file.  We can encrypt them using the following commands:
```
cd ../nifi-toolkit-1.14.0
# The breakdown of this command is as follows:
# -b specifies the NiFi bootstrap.conf, which specifies nifi.bootstrap.protection.hashicorp.vault.conf=./conf/bootstrap-hashicorp-vault.conf
# -n specifies the nifi.properties file to encrypt
# -o specifies the location to output the encrypted nifi.properties file.  If we left out this argument, it would simply encrypt nifi.properties in place.
# -S specifies the protection scheme.  If we left out this argument, it would use the default AES_GCM protection scheme.
./bin/encrypt-config.sh -b ../nifi-1.14.0/conf/bootstrap.conf \
    -n ../nifi-1.14.0/conf/nifi.properties \
    -o nifi.properties.encrypted \
    -S HASHICORP_VAULT_TRANSIT
```

We're prompted to enter a password (must be at least 12 characters), which will be used to configure the `nifi-1.14.0/conf/bootstrap.conf` with a `nifi.bootstrap.sensitive.key`.  This value is not used by the HashiCorp Vault protection scheme, but is required for other protection schemes.  Since different properties are permitted to be protected by different protection schemes at the same time, the key is still generated from the password you enter.

After you enter a password, you should see output like the following:
```
06:45:33 INFO [main] org.apache.nifi.properties.AbstractBootstrapPropertiesLoader: Determined default application properties path to be 'conf/nifi.properties'
06:45:33 INFO [main] org.apache.nifi.properties.NiFiPropertiesLoader: Loaded 202 properties from ../nifi-1.14.0/conf/nifi.properties
06:45:33 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Loaded NiFiProperties instance with 202 properties
06:45:34 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Protected nifi.security.keyPasswd with hashicorp/vault/transit/nifi-transit -> 	vault:v1:2DpftAUtY6A7YLCStaUQtMacOxhX9buAKIQOTJgyCI8JUJuoOWmn7jj40bbqm7ILwNKC1SF5TgXzpXvu
06:45:34 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Updated protection key nifi.security.keyPasswd.protected
06:45:34 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Protected nifi.security.keystorePasswd with hashicorp/vault/transit/nifi-transit -> 	vault:v1:fyvi2zs4vFmYjIvgLSmlNMOwZL88QJzRt9s2TKClWdCLYA16wrJdQkwvmwa58N6gJOECTD0Z3tr1j77T
06:45:34 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Updated protection key nifi.security.keystorePasswd.protected
06:45:34 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Protected nifi.security.truststorePasswd with hashicorp/vault/transit/nifi-transit -> 	vault:v1:X3GIns9HbAfMJN9N8zorKK8zl+Vq4dPCzbNFVgir4pD1VZogFlKoI6BV/2oG/zq2TnJBITvwW1nwHL3S
06:45:34 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Updated protection key nifi.security.truststorePasswd.protected
06:45:34 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Protected nifi.sensitive.props.key with hashicorp/vault/transit/nifi-transit -> 	vault:v1:NmURB5C6nIQ+7Nc62RRHdTuuTotJDdsIU0qYu8uxiO/giUl9Y9cHRpFL13+wC24sqwWvqOyLZi7lSGWF
06:45:34 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Updated protection key nifi.sensitive.props.key.protected
06:45:34 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Final result: 205 keys including 4 protected keys
```

Our new protected properties can be seen in `nifi.properties.encrypted`:
```
nifi.security.keystore=./conf/keystore.p12
nifi.security.keystoreType=PKCS12
nifi.security.keystorePasswd=vault:v1:fyvi2zs4vFmYjIvgLSmlNMOwZL88QJzRt9s2TKClWdCLYA16wrJdQkwvmwa58N6gJOECTD0Z3tr1j77T
nifi.security.keystorePasswd.protected=hashicorp/vault/transit/nifi-transit
nifi.security.keyPasswd=vault:v1:2DpftAUtY6A7YLCStaUQtMacOxhX9buAKIQOTJgyCI8JUJuoOWmn7jj40bbqm7ILwNKC1SF5TgXzpXvu
nifi.security.keyPasswd.protected=hashicorp/vault/transit/nifi-transit
nifi.security.truststore=./conf/truststore.p12
nifi.security.truststoreType=PKCS12
nifi.security.truststorePasswd=vault:v1:X3GIns9HbAfMJN9N8zorKK8zl+Vq4dPCzbNFVgir4pD1VZogFlKoI6BV/2oG/zq2TnJBITvwW1nwHL3S
nifi.security.truststorePasswd.protected=hashicorp/vault/transit/nifi-transit
```

Notice that the `.protected` properties indicate `hashicorp/vault/transit/nifi-transit`, which tells NiFi to use the HashiCorp Vault Transit Sensitive Property Provider using a Vault path prefix of `"nifi-transit"`.  If we want, we can configure the encryption parameters of this path in Vault.

Now, what if you already had configuration files encrypted with AES-GCM?  The following command would migrate the encryption from AES-GCM to Vault:
```
cd nifi-toolkit-1.14.0
# This won't work in our walkthrough since nifi.properties are not protected with AES_GCM, but the command is provided for future reference
# Note the addition of -H, which indicates the old protection scheme.  This can technically be left off since it's the default value, but is provided for clarity
# Also note the addition of -m, which tells the tool to migrate from the old scheme to the new scheme.
./bin/encrypt-config.sh -b ../nifi-1.14.0/conf/bootstrap.conf \
    -n ../nifi-1.14.0/conf/nifi.properties \
    -o nifi.properties.encrypted \
    -S HASHICORP_VAULT_TRANSIT \
    -H AES_GCM \
    -m
```

### Starting NiFi

Now that we've successfully encrypted our `nifi.properties` using HashiCorp Vault, let's start it up and see it decrypt the values!

```
cp nifi.properties.encrypted ../nifi-1.14.0/conf/nifi.properties
cd ../nifi-1.14.0
./bin/nifi.sh start
tail -f logs/nifi-app.logs
```

Since you've already configured `nifi-1.14.0/conf/bootstrap-hashicorp-vault.conf`, NiFi should use that configuration to connect to Vault when it encounters the `hashicorp/vault/transit/nifi-transit` values in `nifi.properties`, allowing it to decrypt the protected properties.  If everything works as planned, you should be able to view NiFi's UI at [https://localhost:8443/nifi](https://localhost:8443/nifi).
