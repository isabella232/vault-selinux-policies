# vault-selinux-policies

This repo contains a baseline SELinux Targeted Policy, and CircleCI scripts to package the policy into RPMs (targeting CentOS and Fedora) for [HashiCorp Vault](https://www.vaultproject.io).

It is _not_ recommended to run this in Production without extensive testing first!

## Overview

This repo holds the raw SELinux policy files, the packaging scripts _and_ validation scripts to package and validate via CircleCI.

### Caveats

* The SELinux policy file contexts, used as part of SELinux labelling, map to where the packaged versions of Vault install their config, data and log files. The Vault packages are available from: https://learn.hashicorp.com/tutorials/vault/getting-started-install
* The SELinux policies do allow fairly open outbound network traffic, although not outbound HTTP access by default.
* The SELinux policies haven't been tested with Vault+Consul, or other storage engines, except for Integrated Storage.
* The current policy is in Enforced Mode (which will block and audit), but can be edited into Permissive Mode (which will audit only).

### Layout

The [products/vault_selinux/](products/vault_selinux/) folder contains the raw SELinux config files, plus [package.sh](products/vault_selinux/package.sh). This script gets executed by [circle](.circleci/config.yml).

The [products/vault_selinux/](products/vault_selinux/) folder contains the [ci/validate.sh](products/vault_selinux/ci/validate.sh) script. This gets executed by [circle](.circleci/config.yml) too.

### Booleans

While the current baseline provides fairly open access, there are some features that are gated by SELinux [Booleans](https://wiki.gentoo.org/wiki/SELinux/Tutorials/Using_SELinux_booleans).

* `vault_outbound_udp_dns` - if set will allow Vault to query DNS via UDP
* `vault_outbound_http` - if set will allow Vault to send outbound HTTP requests

To enable the booleans:

```
sudo setsebool vault_outbound_udp_dns on
sudo setsebool vault_outbound_http on
```

If you'd like to persist these setting:

```
sudo setsebool -P vault_outbound_udp_dns on
sudo setsebool -P vault_outbound_http on
```

## Editing the source

To edit the underlying SELinux policies, edit the `vault*` files in [products/vault_selinux/](products/vault_selinux/).

The file context mappings are in `vault.fc` - the interfaces are in `vault.if` and the main policy is in `vault.te`.

The `vault.sh` is used to compile, install and bundle the policy into an RPM file. This is the main way to update the policy on the system, and was generated with `sepolicy generate --init -n vault /usr/sbin/vault`

The `vault_selinux.spec` is also generated by the above, and is used to create the RPM. Although this file has been modified.

## CI

Pushing to the main branch of this repo will execute CircleCI jobs: https://app.circleci.com/pipelines/github/hashicorp/vault-selinux-policies

These jobs will save RPM artifacts in the package steps, one for CentOS, and one for Fedora.

## Testing locally

This has only been tested on CentOS and Fedora, and requires some pre-requisites. The AWS steps below offer a more thorough example of how to test this on CentOS and Fedora.

### CentOS

Install Vault:

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install vault
```

Install SELinux Policy development pre-requisites:

```
sudo yum -y install policycoreutils-devel setools-console rpm-build selinux-policy-devel selinux-policy-targeted
```

Clone this repo, update versions, then run the `vault.sh` script.

```
cd products/vault_selinux
sed -i "s^#VERSION#^0.1.1^g" vault.te
sed -i "s^#VERSION#^0.1.1^g" vault_selinux.spec
sudo ./vault.sh
```

To re-install, after making changes to the SELinux files, you can re-run this script.

### Fedora

Install Vault:

```
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo
sudo dnf -y install vault
```

Install SELinux Policy development pre-requisites:

```
sudo dnf -y install policycoreutils-devel setools-console rpm-build
```

Clone this repo, update versions, then run the `vault.sh` script.

```
cd products/vault_selinux
sed -i "s^#VERSION#^0.1.1^g" vault.te
sed -i "s^#VERSION#^0.1.1^g" vault_selinux.spec
sudo ./vault.sh
```

To re-install, after making changes to the SELinux files, you can re-run this script.

## Testing on AWS

To test that the RPM packages are installable the `terraform` directory contains logic to spin up 2 Centos instances.

```
cd terraform
make up
```

SSH to the instances then run:

```
sudo cloud-init status --wait
```

Wait until the cloud-init stuff has finished, this will install Vault from the latest available package.

SCP the appropriate RPM to the instances.

Then from the instances (or dnf/yum install), for example:

```
sudo rpm -ivh vault_selinux-1.1-1.el7.noarch.rpm
```

You can check that the policy has applied properly by checking appropriate `vault_t` contexts in:

```
ls -alZ /opt/vault
ls -alZ /var/log/vault
ls -alZ /usr/sbin/vault
ls -alZ /etc/vault.d
```

Or by running `semanage` commands, similar to our [validation](products/vault_selinux/ci/validate.sh) logic:

```
sudo semanage module -l | grep vault
sudo semanage port -l | grep vault_cluster_port_t
```

You should then be able to start the vault server on the instance:

```
sudo systemctl start vault.service
```

From the centos user you can:
```
export VAULT_ADDR=http://127.0.0.1:8200
vault status
```

Hopefully, if you tail the audit log you shouldn't see AVC errors for vault:

```
sudo tail -f /var/log/audit/audit.log | grep vault
```

After you've initialized and unsealed vault you can also enable the audit engine to write to /var/log/vault/vault.log

```
vault audit enable file file_path=/var/log/vault/vault.log
```

Don't forget to tear down your infra after:

```
make down
```

from the terraform folder.

## End to End Testing on AWS

First, ensure you can `make up` and `make down` as above.

Second, place a copy of the CentOS RPM into the `terraform` folder as `vault_selinux.rpm`.

Then run `make integration` from within the `terraform` folder. This will spin up the cluster, as above, then interact with the instances, deploying the RPM, and ensuring that they can function.
