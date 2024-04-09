# Lab Two: Deploy Greengrass on prplOS

## Step 1 - Connect to the LCM SDK contaienr

1.1 - Take note of the container Id that corresponds to the SDK

```bash

buntu@ip-172-31-16-224:~$ sudo docker ps
CONTAINER ID   IMAGE                                                               COMMAND                  CREATED       STATUS       PORTS                                             NAMES
7604f1d718ed   registry:2                                                          "/entrypoint.sh /etc…"   2 hours ago   Up 2 hours   0.0.0.0:443->443/tcp, :::443->443/tcp, 5000/tcp   test-registry
dd2946e6e6a9   registry.gitlab.com/prpl-foundation/sdk/lcm/lcm_sdk_x86-64:v3.0.0   "/home/lcmuser/init.…"   2 hours ago   Up 2 hours                                                     lcm_sdk

```

1.2 - Attach to the SDK container

```bash
sudo docker exec -it lcm_sdk /bin/bash
# lcmuser@dd2946e6e6a9:/sdkworkdir$
```

## Step 2 - Create recipe

2.1 - Create new recipe for Nucleus

```bash

mkdir /home/lcmuser/ggv2-image 

cd  /home/lcmuser/ggv2-image

wget https://d2s8p88vqu9w66.cloudfront.net/releases/greengrass-2.12.4.zip -O /sdkworkdir/downloads/greengrass-2.12.4.zip

wget https://raw.githubusercontent.com/aws-greengrass/aws-greengrass-nucleus/main/LICENSE -O /sdkworkdir/downloads/LICENSE

wget https://raw.githubusercontent.com/aws-greengrass/aws-greengrass-nucleus/main/LICENSE -O /home/lcmuser/ggv2-image/LICENSE

devtool add ggv2-image  /home/lcmuser/ggv2-image

```

2.2 - Edit the recipe contents 

```bash

sudo apt install vim

devtool edit-recipe ggv2-image

```

```bash
SUMMARY = "AWS IoT Greengrass Nucleus - Binary Distribution"
DESCRIPTION = "The Greengrass nucleus component provides functionality for device side orchestration of deployments and lifecycle management for execution of Greengrass components and applications."
HOMEPAGE = "https://github.com/aws-greengrass/aws-greengrass-nucleus"
LICENSE     = "Apache-2"

S                          = "${WORKDIR}"
GG_BASENAME                = "greengrass/v2"
GG_ROOT                    = "${D}/${GG_BASENAME}"
LIC_FILES_CHKSUM           = "file://LICENSE;md5=34400b68072d710fecd0a2940a0d1658"

SRC_URI = "\
    file://greengrass-2.12.4.zip; \
    file://LICENSE;name=license; \
    "

SRC_URI[license.md5sum]    = "34400b68072d710fecd0a2940a0d1658"
SRC_URI[license.sha256sum] = "09e8a9bcec8067104652c168685ab0931e7868f9c8284b66f5ae6edae5f1130b"


RDEPENDS:${PN} += "\
    ca-certificates \
    corretto-11-bin \
    python3-core \
    python3-json \
    python3-numbers \
    sudo \
    "

do_configure[noexec] = "1"
do_compile[noexec]   = "1"

do_install() {

    # directories
    install -d ${GG_ROOT}/config
    install -d ${GG_ROOT}/alts
    install -d ${GG_ROOT}/alts/init
    install -d ${GG_ROOT}/alts/init/distro
    install -d ${GG_ROOT}/alts/init/distro/bin
    install -d ${GG_ROOT}/alts/init/distro/conf
    install -d ${GG_ROOT}/alts/init/distro/lib

    install -m 0440 ${WORKDIR}/LICENSE                         ${GG_ROOT}
    install -m 0640 ${WORKDIR}/bin/loader                      ${GG_ROOT}/alts/init/distro/bin/loader
    install -m 0640 ${WORKDIR}/conf/recipe.yaml                ${GG_ROOT}/alts/init/distro/conf/recipe.yaml
    install -m 0740 ${WORKDIR}/lib/Greengrass.jar              ${GG_ROOT}/alts/init/distro/lib/Greengrass.jar


}
FILES:${PN} = "/${GG_BASENAME} \
               ${sysconfdir} \
               ${systemd_unitdir}"


inherit systemd
SYSTEMD_AUTO_ENABLE = "enable"
SYSTEMD_SERVICE:${PN} = "greengrass.service"

inherit useradd

USERADD_PACKAGES = "${PN}"
GROUPADD_PARAM:${PN} = "-r ggc_group"
USERADD_PARAM:${PN} = "-r -M -N -g ggc_group -s /bin/false ggc_user"
GROUP_MEMS_PARAM:${PN} = ""

do_package_qa[noexec] = "1"

INSANE_SKIP:${PN} += "already-stripped ldflags file-rdeps"
```

## Step 3 - Build container image

3.1 - Build recipe 

```bash

devtool build gglite-testing
# NOTE: Starting bitbake server...
# NOTE: Tasks Summary: Attempted 655 tasks of which 655 didn't need to be rerun and all succeeded.
```

3.2 - Build image 
```bash
devtool build-image
# NOTE: Starting bitbake server...
# INFO: Successfully built image-lcm-amx-ubus-usp-lcmsampleapp. You can find output files in /sdkworkdir/tmp/deploy/images/container-x86-64

```

3.3 - Copy the container image to the registry

```bash
skopeo copy \
    oci:/sdkworkdir/tmp/deploy/images/container-x86-64/image-lcm-amx-ubus-usp-lcmsampleapp-container-x86-64.rootfs-oci \
    docker://{_YOUR_EC2_PUBLIC_IP_}/gglite-testing:latest \
    --dest-tls-verify=false
```

## Step 4 - Deploy containner

4.1 - Open ubus-cli and subscribe to SoftwareModules using `SoftwareModules.?&`

```bash
ubus-cli
#  > SoftwareModules.?&
# Added subscription for SoftwareModules.
# >
```

4.2 - Create new execution environment and enable it

```bash
SoftwareModules.AddExecEnv(Name="generic-new", Vendor="Cthulhu", Version="3.5.2", ParentExecEnv="", InitialRunLevel=-1, AllocatedMemory=-1, AllocatedDiskSpace=1000000, AllocatedCPUPercent=100,  MaxBandwidthUpstream=-1, MaxBandwidthDownstream=-1) 

SoftwareModules.ExecEnv.2.Enable=1


```

4.3 - Use the `InstallDU` method to deploy the custom container using the new execution environment

```bash
SoftwareModules.InstallDU(URL="docker://{_YOUR_EC2_PUBLIC_IP_}/ggv2-v1:latest", UUID="bd50974c-0fcf-48f0-82e3-2210d4a7ef70", ExecutionEnvRef="generic-new", NetworkConfig = { "AccessInterfaces" = [{"Reference" = "Wan"}] })
```

4.4 - Check the container status

```bash
lxc-ls --fancy
# NAME                                 STATE   AUTOSTART GROUPS IPV4          IPV6 UNPRIVILEGED
# 774736eb-0ed2-5f54-aa76-e99b67a048e3 RUNNING 0         -      192.168.5.100 -    false

```

4.5 - Attach to the container

```bash
lxc-attach  -n 774736eb-0ed2-5f54-aa76-e99b67a048e3
# root@a2fbbddf-e374-5235-bbfd-8bc7beb429bb:/#

```

## Step 5 - Configure Nucleus


5.1  - Check the version

```bash
java -jar /greengrass/v2/alts/init/distro/lib/Greengrass.jar --version
# AWS Greengrass v2.12.4


``` 

5.2 - Create config file 

```bash
cat <<EOF > /greengrass/config.yaml
---
system:
  certificateFilePath: "/greengrass/device.pem.crt"
  privateKeyPath: "/greengrass/private.pem.key"
  rootCaPath: "/greengrass/AmazonRootCA1.pem"
  rootpath: "/greengrass/v2"
  thingName: "GGvLitePrplOS"
services:
  aws.greengrass.Nucleus:
    componentType: "NUCLEUS"
    version: "2.12.4"
    configuration:
      awsRegion: "eu-west-1"
      iotRoleAlias: "GreengrassV2TokenExchangeRoleAlias"
      iotDataEndpoint: "{_YOUR_IOT_CORE_ATS_ENDPOINT_}"
      iotCredEndpoint: "{_YOUR_IOT_CORE_CREDENTIALS_PROVIDER_ENDPOINT_}"
EOF

```
5.3 - Use the AWS IoT Console to:

1. Create an AWS IoT Thing called `GGvLitePrplOS`
2. Create an AWS IoT Policy
3. Create a certificate
4. Associate the IoT Policy you just created to the certificate and attach the certificate to the IoT thing you just created


5.4 - Copy the private key 

```bash
cat <<EOF > /greengrass/private.pem.key
-----BEGIN RSA PRIVATE KEY-----
EXAMPLE
-----END RSA PRIVATE KEY-----
EOF
```

5.5 - Copy the certificate 

```bash
cat <<EOF > /greengrass/device.pem.crt
-----BEGIN CERTIFICATE-----
EXAMPLE
-----END CERTIFICATE-----
EOF
```

5.6 - Copy the root CA

```bash
cat <<EOF > /greengrass/AmazonRootCA1.pem
-----BEGIN CERTIFICATE-----
MIIDQTCCAimgAwIBAgITBmyfz5m/jAo54vB4ikPmljZbyjANBgkqhkiG9w0BAQsF
ADA5MQswCQYDVQQGEwJVUzEPMA0GA1UEChMGQW1hem9uMRkwFwYDVQQDExBBbWF6
b24gUm9vdCBDQSAxMB4XDTE1MDUyNjAwMDAwMFoXDTM4MDExNzAwMDAwMFowOTEL
MAkGA1UEBhMCVVMxDzANBgNVBAoTBkFtYXpvbjEZMBcGA1UEAxMQQW1hem9uIFJv
b3QgQ0EgMTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALJ4gHHKeNXj
ca9HgFB0fW7Y14h29Jlo91ghYPl0hAEvrAIthtOgQ3pOsqTQNroBvo3bSMgHFzZM
9O6II8c+6zf1tRn4SWiw3te5djgdYZ6k/oI2peVKVuRF4fn9tBb6dNqcmzU5L/qw
IFAGbHrQgLKm+a/sRxmPUDgH3KKHOVj4utWp+UhnMJbulHheb4mjUcAwhmahRWa6
VOujw5H5SNz/0egwLX0tdHA114gk957EWW67c4cX8jJGKLhD+rcdqsq08p8kDi1L
93FcXmn/6pUCyziKrlA4b9v7LWIbxcceVOF34GfID5yHI9Y/QCB/IIDEgEw+OyQm
jgSubJrIqg0CAwEAAaNCMEAwDwYDVR0TAQH/BAUwAwEB/zAOBgNVHQ8BAf8EBAMC
AYYwHQYDVR0OBBYEFIQYzIU07LwMlJQuCFmcx7IQTgoIMA0GCSqGSIb3DQEBCwUA
A4IBAQCY8jdaQZChGsV2USggNiMOruYou6r4lK5IpDB/G/wkjUu0yKGX9rbxenDI
U5PMCCjjmCXPI6T53iHTfIUJrU6adTrCC2qJeHZERxhlbI1Bjjt/msv0tadQ1wUs
N+gDS63pYaACbvXy8MWy7Vu33PqUXHeeE6V/Uq2V8viTO96LXFvKWlJbYK8U90vv
o/ufQJVtMVT8QtPHRh8jrdkPSHCa2XV4cdFyQzR1bldZwgJcJmApzyMZFo6IQ6XU
5MsI+yMRQ+hDKXJioaldXgjUkK642M4UwtBV8ob2xJNDd2ZhwLnoQdeXeGADbkpy
rqXRfboQnoZsG4q5WTP468SQvvG5
-----END CERTIFICATE-----
EOF
```

5.7 - Start Nucleus

```bash
sudo -E java -Droot="/greengrass/v2" -Dlog.store=FILE \
	-jar /greengrass/v2/alts/init/distro/lib/Greengrass.jar \
	--init-config /greengrass/config.yaml 
```