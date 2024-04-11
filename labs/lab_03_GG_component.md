
# Lab Three: Create a Greengrass component

## Step 1 - Create an S3 bucket to store artifacts

1.1 - Create bucket

```bash
aws s3 mb {YOUR-S3-BUCKET}
```

## Step 2 - Upload artifacts

2.1 - Clone the git repository that contains the component’s artifacts and recipe

```bash
cd ~/
git clone https://github.com/aws-samples/aws-iot-gg-onnx-runtime.git
```

2.2 -  Navigate to the artifacts folder and zip the files

```bash
cd ~/aws-iot-gg-onnx-runtime/artifacts/com.demo.onnx-imageclassification/1.0.0 
zip -r greengrass-onnx.zip .
```

2.3 - Upload the zip file to the Amazon S3 bucket that you created in the initial setup.

```bash
aws s3 cp greengrass-onnx.zip s3://{YOUR-S3-BUCKET}/greengrass-onnx.zip
```

## Step 3 - Publish artifacts

3.1 - Open the recipe file aws-iot-gg-onnx-runtime/recipes/com.demo.onnx-imageclassification-1.0.0.json in a text editor. 

```bash
cd ~/aws-iot-gg-onnx-runtime/recipes/
vi com.demo.onnx-imageclassification-1.0.0.json
```

```json
"Artifacts": [
    {
      "URI": "s3://{YOUR-S3-BUCKET}/greengrass-onnx.zip",
      "Unarchive": "ZIP"
    }
  ]

```

3.2 Before publishing the component, make sure that you are using the same region where you created the resources in the initial setup. 

```bash
aws configure set default.region eu-west-1
```

3.3 - Publish the ONNX Runtime componen and the omponent that will perform the image classification.

```bash
aws greengrassv2 create-component-version --inline-recipe fileb://com.demo.onnxruntime-1.0.0.json

aws greengrassv2 create-component-version --inline-recipe fileb://com.demo.onnx-imageclassification-1.0.0.json

```

3.4 - To verify that the components were published successfully, navigate to the AWS IoT Console, go to Greengrass Devices >> Components. In the My Components tab, you should see the two components that you just published.

## Step 4 - Deploy the component to a target device

4.1 - To deploy the component to the target device, go to the AWS IoT Console, navigate to Greengrass Devices >> Deployments and choose Create.

4.2 - Fill in the deployment name as well as the name of the core device you want to deploy to.

4.3 - In the Select Components section, select the component com.demo.onnx-imageclassification.

4.4 - Leave all other options as default and choose Next until you reach the Review section of your deployment and then choose Deploy.

## Step 5 - Review inferencing results

5.1 - Review the logs in `/greengrass/v2/logs/greengrass.log`

5.2 - Review the logs in `/greengrass/v2/logs/com.demo.onnxruntime.log`

5.3 - Review the logs in `/greengrass/v2/logs/com.demo.onnx-imageclassification.log`

5.4 - Go to the AWS IoT Console test client and subscribe to the topic `demo/onnx`

## Step 6 - Inferece using external resources

6.1 - Update the deployment unit. Create a HostObject that maps a local directory to the inference directory inside the container

```bash
mkdir /var/images-test/

ubus-cli

SoftwareModules.?&

SoftwareModules.DeploymentUnit.2.Update(RetainData = 1, HostObject = [{ "Source"="/var/images-test/", "Destination"="/greengrass/v2/packages/artifacts-unarchived/com.demo.onnx-imageclassification/1.0.0/greengrass-onnx/images/", Options = "type=mount"}], NetworkConfig = { "AccessInterfaces" = [{"Reference" = "Wan"}] })

```

6.2 - Check the results inside the container

```bash
 df -h | grep greengrass
# tmpfs                   995.3M    800.0K    994.5M   0% /greengrass/v2/packages/artifacts-unarchived/com.demo.onnx-imageclassification/1.0.0/greeng
# root@ad44f4d4-57ff-5a25-9724-5c6206bbce17:/# 

```

6.3 - Download a test image into the local directory

```bash
cd /var/images-test/

wget -4 https://raw.githubusercontent.com/aws-samples/aws-iot-gg-onnx-runtime/ca42901c5910ca3f518440639e0fc15066d95f73/artifacts/com.demo.onnx-imageclassification/1.0.0/images/negative-space-hot-air-balloon.jpg
```

6.4 - Start Nucleus

```bash
lxc-attach  -n 774736eb-0ed2-5f54-aa76-e99b67a048e3

sudo -E java -Droot="/greengrass/v2" -Dlog.store=FILE \
	-jar /greengrass/v2/alts/init/distro/lib/Greengrass.jar \
	--init-config /greengrass/config.yaml 
```

6.5 - Checl results using the AWS IoT Console test client

FIN