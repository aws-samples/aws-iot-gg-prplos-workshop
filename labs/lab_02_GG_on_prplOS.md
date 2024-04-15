# Lab Two: Deploy Greengrass on prplOS

## Step 1 - Connect to the LCM SDK contaienr


1.1 - Attach to the SDK container

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
    python3-profile \
    python3-core \
    python3-json \
    python3-numbers \
    python3-pip \
    python3-numpy \
    netcat \
    strace \
    sudo \
    libstdc++6 \
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

2.3 - Clone `aws-meta` into the layers directory and add it to `bblayers.conf`

```bash
cd /sdkworkdir/layers/
git clone -b honister https://github.com/aws4embeddedlinux/meta-aws 
vi /sdkworkdir/conf/bblayers.conf
```

```
BBPATH = "${TOPDIR}"
SDKBASEMETAPATH = "${TOPDIR}"
BBLAYERS := " \
    ${SDKBASEMETAPATH}/layers/poky/meta \
    ${SDKBASEMETAPATH}/layers/poky/meta-openembedded/meta-oe \
    ${SDKBASEMETAPATH}/layers/poky/meta-openembedded/meta-python \
    ${SDKBASEMETAPATH}/layers/poky/meta-openembedded/meta-networking \
    ${SDKBASEMETAPATH}/layers/poky/meta-openembedded/meta-filesystems \
    ${SDKBASEMETAPATH}/layers/poky/meta-openembedded/meta-webserver \
    ${SDKBASEMETAPATH}/layers/meta-aws \
    ${SDKBASEMETAPATH}/layers/meta-lcm/meta-amx \
    ${SDKBASEMETAPATH}/layers/meta-lcm/meta-usp \
    ${SDKBASEMETAPATH}/layers/poky/meta-yocto-bsp \
    ${SDKBASEMETAPATH}/layers/meta-bsp/meta-lcm-containers \
    ${SDKBASEMETAPATH}/layers/poky/meta-lcm-basic \
    ${SDKBASEMETAPATH}/layers/meta-bsp/meta-virtualization \
    ${SDKBASEMETAPATH}/workspace \
    "

```

## Step 3 - Build container image

3.1 - Build recipe 

```bash

devtool build ggv2-image
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
    docker://{_YOUR_EC2_PUBLIC_IP_}/ggv2-testing:latest \
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
SoftwareModules.InstallDU(URL="docker://{_YOUR_EC2_PUBLIC_IP_}/ggv2-testing:latest", UUID="8e2ec7c2-4a8f-5ff1-b241-f2c0e275868b", ExecutionEnvRef="generic-new", NetworkConfig = { "AccessInterfaces" = [{"Reference" = "Wan"}] })
```

4.4 - Check the container status

```bash
lxc-ls --fancy
# NAME                                 STATE   AUTOSTART GROUPS IPV4          IPV6 UNPRIVILEGED
# c9c9f519-2719-544b-82e6-0571578b8154 RUNNING 0         -      192.168.5.100 -    false

```

4.5 - Attach to the container

```bash
lxc-attach  -n c9c9f519-2719-544b-82e6-0571578b8154
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
EXAMPLE
-----END CERTIFICATE-----
EOF
```

5.7 - Start Nucleus

```bash
sudo -E java -Droot="/greengrass/v2" -Dlog.store=FILE \
	-jar /greengrass/v2/alts/init/distro/lib/Greengrass.jar \
	--init-config /greengrass/config.yaml 
```