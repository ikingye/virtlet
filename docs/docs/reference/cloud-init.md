# Cloud-init options

Virtlet uses [Cloud-Init](https://cloudinit.readthedocs.io/en/latest/)
data generation mechanism for the following purposes:

* setting the host name based on the name of the pod
* injecting ssh keys
* setting up the VM network
* writing the contents of Secrets and ConfigMaps into the VM
* volume mounting
* setting up block devices based on PVs
* adding user-defined Cloud-Init settings such as startup scripts

See also the list of
[annotations](../vm-pod-spec/#annotations-recognized-by-virtlet) for
the list of annotations that are used for Cloud-Init.

# Example

Below is a pod definition that we'll be using. The `annotations:`
part lists the cloud-init related annotations supported by Virtlet.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-vm
  annotations:
    kubernetes.io/target-runtime: virtlet.cloud

    # override some fields in cloud-init meta-data
    VirtletCloudInitMetaData: |
      instance-id: foobar

    # override some fields in cloud-init user-data
    VirtletCloudInitUserData: |
      users:
      - name: cloudy
        gecos: Magic Cloud App Daemon User
        inactive: true
        system: true

    # this option disables merging of user-data keys.
    # By default the lists and dicts in user-data keys
    # are merged using the standard method (more on this below)
    # VirtletCloudInitUserDataOverwrite: "true"

    # this options makes it possible to write a script
    # in place of the user-data file. In case if this option
    # is present user-data part is not generated.
    # VirtletCloudInitUserDataScript: |
    #   #!/bin/sh
    #   echo hello world

    # it's also possible to use a configmap to override
    # parts of user-data
    VirtletCloudInitUserDataSource: configmap/vm-user-data

    # it's also possible to specify that the whole user-data contents
    # are stored in a ConfigMap under the specified key:
    # VirtletCloudInitUserDataSourceKey: user-data

    # by default, the contents of user-data in a ConfigMap key is specified
    # as plain text, but it can also be encoded using base64:
    # VirtletCloudInitUserDataSourceEncoding: base64

    # Virtlet-specific annotations follow
    # Specify some ssh keys directly
    VirtletSSHKeys: |
      ssh-rsa AAAA... me@localhost
      ssh-rsa AAAA... me1@localhost

    # ssh keys may also be pulled from 'authorized_keys' key
    # in a ConfigMap or a Secret
    VirtletSSHKeySource: secret/mysecret
    # can also use the following:
    # VirtletSSHKeySource: configmap/configmap-with-public-keys
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: extraRuntime
            operator: In
            values:
            - virtlet

  containers:
  - name: ubuntu-vm
    image: virtlet.cloud/cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
    volumeMounts:

    # this will write configmap contents using the `write_file` cloud-init module
    - name: config
      mountPath: /etc/foobar

    # this will write secret contents using the `write_file` cloud-init module
    - name: secret
      mountPath: /etc/baz

    # mount the qcow2 volume under /var/lib/docker
    - name: extra
      mountPath: /var/lib/docker

    # mount a raw device under /var/lib/postgresql
    - name: raw
      mountPath: /var/lib/postgresql

    # mount a ceph volume under /cephdata
    - name: ceph
      mountPath: /cephdata

  volumes:
  - name: config
    configMap:
      name: someconfig
  - name: secret
    secret:
      secretName: somesecret
  - name: extra
    flexVolume:
      driver: "virtlet/flexvolume_driver"
      options:
        type: qcow2
        capacity: 2048MiB
  - name: raw
    flexVolume:
      driver: "virtlet/flexvolume_driver"
      options:
        type: raw
        path: /dev/mapper/somevg
  - name: ceph
    flexVolume:
      driver: "virtlet/flexvolume_driver"
      options:
        type: ceph
        monitor: 10.0.0.1:6789
        user: libvirt
        secret: ....
        volume: rbd-test-image
        pool: libvirt-pool
```

# Cloud-init data generation

The [cloud-init](http://cloudinit.readthedocs.io/en/latest/index.html)
data generated by Virtlet consists of `meta-data`, `user-data` and `network-config`
parts.  It's currently provided by means of
[NoCloud](http://cloudinit.readthedocs.io/en/latest/topics/datasources/nocloud.html)
and [Config Drive](http://cloudinit.readthedocs.io/en/latest/topics/datasources/configdrive.html)
datasource, which in this case involves making an iso9660 image to be
mounted by the VM. An implementation of either
[Amazon EC2](http://cloudinit.readthedocs.io/en/latest/topics/datasources/ec2.html)
or
[OpenStack](http://cloudinit.readthedocs.io/en/latest/topics/datasources/openstack.html)
datasource can be added later.

The cloud-init data is generated based on the following sources:

* pod name
* direct user-data/meta-data values from pod definition
* direct user-data/meta-data values from configmap/secret
* pod volumes and container mounts
* pod network configuration
* ssh keys specified either in pod definition or using a configmap/secret

# Output ISO image format

Virtlet supports two types of Cloud-init ISO9660-based datasources, NoCloud and
ConfigDrive. The user may choose an appropriate one using
`VirtletCloudInitImageType` annotation, which can have either `nocloud` or
`configdrive` as its value. When there's no `VirtletCloudInitImageType`
annotation, Virtlet defaults to `nocloud`.

# Detailed structure of the generated files

The `meta-data` part is a piece of JSON that looks like this:
```json
{
  "instance-id": "foobar",
  "local-hostname": "ubuntu-vm",
  "public-keys": ["ssh-rsa AAAA... me@localhost", "ssh-rsa AAAA... me1@localhost", "..."]
}
```

In case of ConfigDrive, this JSON has `instance-id` repeated as `uuid` and 
`local-hostname` repeated as `hostname`.

We're using JSON format here so as to be compatible with older
cloud-init implementations such as one used in
[CirrOS](https://launchpad.net/cirros) images which are used to test
Virtlet.
The following fields are generated by Virtlet:

* `instance-id` by default contains a name generated from pod name and
  pod namespace name separated with dot (`ubuntu-vm.default`). In this
  case it's overridden using `VirtletCloudInitMetaData` in the pod
  definition. Most of the time this field doesn't change much
  in the behavior of Virtlet VMs.
* `local-hostname` contains the name of the pod so VM's hostname
  defaults just to the pod name
* `public-keys` is a list of ssh public keys to put into the default
  user's `~/.ssh/authorized_keys` file. It's taken either from
  `VirtletSSHKeys` annotation or from `authorized_keys` key in a
  kubernetes Secret on ConfigMap specified via `VirtletSSHKeySource`.

The `user-data` part is only generated if it's not empty. It's a
YAML file (because CirrOS doesn't support `#cloud-config` anyway)
with the following content:
```yaml
#cloud-config
mounts:
- [ /dev/vdc1, /var/lib/docker ]
- [ /dev/vdd, /var/lib/postgresql ]
- [ /dev/vde, /cephdata ]
write_files:
- path: /etc/foobar/some-file-from-configmap.conf
  permissions: '0644'
  encoding: b64
  content: IyB0aGlzIGlzIGEgc2FtcGxlIGNvbmZpZyBmaWxlCiMgdGFrZW4gZnJvbSBhIGNvbmZpZ21hcAo=
- path: /etc/baz/a-file-taken-from-secrets.conf
  permissions: '0600'
  encoding: b64
  content: IyBzb21lIHNlY3JldCBjb250ZW50Cg==
- path: /foobar.txt
  content: this part is from a configmap specified using VirtletCloudInitUserDataSource
users:
- name: cloudy
  gecos: Magic Cloud App Daemon User
  inactive: true
  system: true
```

[Network configuration](http://cloudinit.readthedocs.io/en/latest/topics/network-config.html)
uses YAML to provide data in
[Network Config Version 1](http://cloudinit.readthedocs.io/en/latest/topics/network-config-format-v1.html).

The `user-data` content is generated as follows:

* `mounts` are generated based on `volumeMount` options of the
  container and pod's volumes that are `flexVolume`s and use
  `virtlet/flexvolume_driver`. These are seen as block devices by the
  VM and Virtlet mounts them into appropriate directories
* `write_files` are generated based on configmaps and secrets mounted
  into the container. It also includes `/etc/cloud/environment` file
  (see [Environment variable support](../vm-pod-spec/#environment-variables) for
  more info) and optionally `/etc/cloud/mount-volumes.sh` that can be
  used to mount volumes on systems without udev (see
  [Workarounds for volume mounting](#workarounds-for-volume-mounting-and-raw-block-volumes) below)
* the yaml content specified using optional `VirtletCloudInitUserData`
  annotation as well as the content from Secret or ConfigMap specified
  using another optional `VirtletCloudInitUserDataSource` annotation
  are merged together using a simple algorithm
  [described](http://cloudinit.readthedocs.io/en/latest/topics/merging.html)
  in cloud-init documentation. During the merge, the autogenerated
  `user-data` comes first, then the contents of
  `VirtletCloudInitUserData` and finally the data taken from a Secret
  or ConfigMap specified via `VirtletCloudInitUserDataSource`
* it's possible to disable `user-data` merge algorithm and use the
  simple key replacement scheme by adding
  `VirtletCloudInitUserDataOverwrite: true` annotation
* it's also possible to replace the `user-data` file content entirely
  by adding `VirtletCloudInitUserDataScript` option. This may be
  useful if you want to pass a script there which may be necessary for
  older/simpler cloud-init implementations such as one used by CirrOS.
  Within the content of this script, `@virtlet-mount-script@` can be
  replaced with volume mounting shell commands and commands for making
  symlinks for raw block volumes (see
  [Workarounds for volume mounting](#workarounds-for-volume-mounting-and-raw-block-volumes) below)

# Propagating user-data from Kubernetes objects

In addition to putting user-data document right in the pod definition
using `VirtletCloudInitUserData` annotation, it is possible to have
this document stored in either `ConfigMap` or `Secret` kubernetes
object. This is achieved with yet another annotation, called
`VirtletCloudInitUserDataSource`. It has the following format:
`kind/name` where `kind` is either `configmap` or `secret` and `name`
is the name of appropriate resource. When `virtlet` sees this
annotation, it reads the object it refers to and puts all its keys
into the `user-data` dictionary. This is done at the very beginning of
the `user-data` generation process.  If the pod definition has both
`VirtletCloudInitUserData` and `VirtletCloudInitUserDataSource`
annotations, the `virtlet` will load `user-data` from kubernetes
object and then will merge it with that from
`VirtletCloudInitUserData` (unless `VirtletCloudInitUserDataOverwrite`
is set to `"true"`, in which case `VirtletCloudInitUserData` will
overwrite it).  There's also an option to store the whole contents of
`user-data` in a single key of ConfigMap which can be specified using
`VirtletCloudInitUserDataSourceKey`. By default, the data are stored
there as plain text but you can also specify `base64` encoding via
`VirtletCloudInitUserDataSourceEncoding` annotation.

Similar approach is taken with SSH keys. It is possible to provide VM
with a list of SSH keys obtained from `ConfigMap` or `Secret`
kubernetes objects. In order to do so, one uses `VirtletSSHKeySource`
annotation with the following format: `kind/name/key`.  As for the
user-data, `kind` is one of `configmap`, `secret`, `name` is the name
of resource and `key` is the key name in that resource containing SSH
keys in the same format in `VirtletSSHKeys`. The `key` part is
optional. When using `kind/name` annotation (without `key`), `virtlet`
will look for the `authorized_keys` key. As with the `user-data`
`VirtletSSHKeys` keys are going to be appended to those from
`VirtletSSHKeySource` unless it is set to overwrite them by
`VirtletCloudInitUserDataOverwrite: "true"`.

# Workarounds for volume mounting and raw block volumes

Currenly Virtlet uses `/dev/disk/by-path` to mount volumes specified
using Virtlet flexvolume driver. There's a problem with this approach
however, namely that this depends on `udev` being used by the guest
system. Moreover, some images (e.g. CirrOS example image) may have
limited cloud-init support and may not provide proper `user-data`
handling necessary to mount the block devices. To handle the first of
this problem, Virtlet uses `write_files` section of `user-data` to
write a shell script named `/etc/cloud/mount-volumes.sh` (the script
uses `/bin/sh`) that performs the necessary volume mounts using sysfs
to look up the proper device names under `/dev`. The script also
checks whether the volumes are already mounted so as not to mount them
several times (so it's idempotent).

Another slightly related problem is that when raw block volumes are
used, symlinks must be created inside the VM that lead to the proper
device files under `/dev`. This is done by means of
`/etc/cloud/symlink-devs.sh` script that's injected via cloud-init.
It's also automatically added to the `runcmd` section, so it gets
executed on the first VM boot and linked from
`/var/lib/cloud/scripts/per-boot/` so that it gets executed on
subsequent VM boots, too. In any case, this script gets executed after
`mounts` are processed, so Virtlet updates `mounts` entries in the
user-specified `user-data` that point to the raw block volume devices
to make them work as if the symlinks were in place during the
mounting.

To deal with the systems without proper `user-data` support such as
CirrOS, Virtlet makes it possible to specify `@virtlet-mount-script@`
inside `VirtletCloudInitUserDataScript` annotation. This string will
be replaced with the content that's normally written to
`/etc/cloud/symlink-devs.sh` and `/etc/cloud/mount-volumes.sh` (in
this order). Given that the script starts with `#!/bin/sh`, you can
use the following construct in the pod annotations:

```yaml
VirtletCloudInitUserDataScript: "@virtlet-mount-script@"
```

# Disabling cloud-init based network configuration

As the network configuration is applied only upon the first startup,
it's not used when
[persistent root filesystem](../volumes/#persistent-root-filesystem)
is used, so new network configuration can be passed using DHCP to a
new pod with the same rootfs. Moreover, in some cases it may be
desirable to force Virtlet to use DHCP instead of cloud-init based
network config. To do so, you need to add the following annotation
to the pod:

```yaml
VirtletForceDHCPNetworkConfig: "true"
```

# Additional links

These links may help to understand some basics about cloud-init:

* [Cloud-init documentation](http://cloudinit.readthedocs.io/en/latest/index.html)
* [Booting cloud images with libvirt](http://blog.oddbit.com/2015/03/10/booting-cloud-images-with-libvirt/)
* [Install cloud-init on Ubuntu and use locally... NoCloud](http://www.whiteboardcoder.com/2016/04/install-cloud-init-on-ubuntu-and-use.html)
* [EC2 instance metadata](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
* [Automating Openstack with cloud init run a script on VM's first boot](https://raymii.org/s/tutorials/Automating_Openstack_with_Cloud_init_run_a_script_on_VMs_first_boot.html)
* [Create a Linux Lab on KVM Using Cloud Images](http://giovannitorres.me/create-a-linux-lab-on-kvm-using-cloud-images.html)