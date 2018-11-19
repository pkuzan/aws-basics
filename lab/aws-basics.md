## AWS Basics
Version 1.3.9  
19th November 2018  

### Prerequisites
Prerequisites should be in a separate document delivered to attendees before the workshop.  

The following task will require Java 8 and Maven to be installed.
An IDE such as Intellij is useful, but not mandatory.
#### Java
Download from [Java JDK 8 Download Link](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

#### Maven
Download from [https:/maven.apache.org/download.cgi](https://maven.apache.org/download.cgi)
Select the Binary zip archive.
Unzip   

The link below contains information on how to set Maven and Java on the path.
[https://www.java.com/en/download/help/path.xml](https://www.java.com/en/download/help/path.xml)

To check everything is ok, open a Command or Terminal window and type in the following commands:

`java -version`  
It should respond with a response similar to the below.
![](images/java-version.png?raw=true)

Same for Maven  
`mvn -version`  
![](images/mvn-version.png?raw=true)

#### IDE (Optional)
Download Intellij Community from [https://www.jetbrains.com/idea](https://www.jetbrains.com/idea)

#### Browser
It is recommended that either Chrome or Firefox are used for this lab.

#### Credentials
You will have been provided AWS authentication details.

* AWS Account ID
* Username
* Temporary password - you will need to change this on first login.

#### Conventions
In the subsequent tasks, please substitute :
* `${your-username}` with your AWS  username
* `${your-account-id}` with your AWS Account ID 

### Intro to Lab
In this lab attendees will gain hands on experience in provisioning and maintaining Apollo AWS resources by leveraging pre-created Service Catalog products 
and creating an AWS deployable Spring Boot application. 
 
### Generate Spring Boot Application.
In your browser, navigate to [https://start.spring.io/](https://start.spring.io/)
Change the artifact name to `${your-username}-aws-basics` and add Actuator and Rest Repositories dependencies.
Press the Generate Project button and a zip file named `${your-username}-aws-basics.zip` will be downloaded.
![](images/initializr.png?raw=true)

Unzip the `${your-username}-aws-basics.zip` file.
Open a Command or Terminal
Change directory to the directory where the zip file was unzipped.
Execute the following commands.
```
mvn package
java -jar target/${your-username}-aws-basics-0.0.1-SNAPSHOT.jar
```

Insert the URL below into your browser
```
http://localhost:8080/actuator/health
```
If the applications has started successfully you should see the following response.
```json
{"status":"UP"}
```

Stop the application by pressing CTRL + C in the Terminal / Command window.

### Add SSL Support
Connected Public Cloud accounts only permit https on port 443. To enable this, we need to generate a self-signed certificate and configure the 
application to use the certificate and port 443.

##### Edit application.properties
Edit the application.properties file in the src/main/resources directory.
Add the following text.

```properties
server.port=443
server.ssl.key-alias=selfsigned
server.ssl.key-password=password
server.ssl.key-store=classpath:keystore.jks
server.ssl.key-store-provider=SUN
server.ssl.key-store-type=JKS
```

##### Generate self-signed SSL Certificate

Open a Command or Terminal window, change directory to `src/main/resources` and enter the following command.
```
keytool -genkey -alias selfsigned -keyalg RSA -keysize 2048 -validity 700 -keypass password -storepass password -keystore keystore.jks
```
You will be asked the following questions :

|Question | Answer |
| --- | --- |
|What is your first and last name? |**your name**|
|What is the name of your organizational unit?|tech-labs|
|What is the name of your organization?|aws-basics|
|What is the name of your City or Locality?|**your city**|
|What is the name of your State or Province?|**your state or province**|
|What is the two-letter country code for this unit?|**your country code**|

Enter `yes` at the `[no]:` prompt and press enter, a file named keystore.jks will be created.

##### Start the Application
As before, build and start the application, remembering to change directory back to the application root - hint: it has `pom.xml` in it.

```
# Mac / Linux
cd ../../..
# Windows
cd ..\..\..

mvn package
java -jar target/${your-username}-aws-basics-0.0.1-SNAPSHOT.jar
```
Note, if you are running a Mac or Linux, you may have to run the application with elevated privileges, as ports
less than 1024 are restricted. 
```
#Mac or Linux 
sudo java -jar target/${your-username}-aws-basics-0.0.1-SNAPSHOT.jar
```
In your browser enter the following URL.
```
https://localhost/actuator/health
```
As you are using a self-signed certificate, your browser will respond with a warning.

![](images/ssl-warning.png?raw=true)

The message will vary depending on the browser. Ignore the warning and proceed. 
The response should be as before.

### Launch Auto-Scaling Group
Auto Scaling Group is Service Catalog Product that creates an Auto Scaling Group and Load Balancer.

* Login to the [AWS Console](https://console.aws.amazon.com/iam/home?region=us-east-1)  
   * Ensure you are in the N. Virginia Region (US_East-1)
   * Use credentials you were given before the workshop
* In the AWS Console, select Services, Service Catalog
* Click on Products List in the tool bar on the left
   * To display the toolbar, click on the 3 horizontal lines on the top left of the screen 
   * ![](images/service-catalog-products.png?raw=true)
* Click `auto_scaling` Product Name
* Click `Launch Product` Button
* Enter a Provisioned Product Name
   * `${your-username}-asg e.g. user5-asg`
* Select the latest Product Version
* Press `NEXT`
* Enter Parameters below
  * Remember to substitute your AWS account id (NOT user id) in the `SSLCertificateArn` parameter value

|Parameter | Value |
| --- | --- |
| SealId | 00001 |
| Prefix | leave blank |
| Environment | DEV |
| DataDogMonitoring | False |
| ApplicationLogs | leave blank |
| InstanceType | t2.micro |
| KeyName | aws-basics-lab-key-pair |
| AMIUpdateDays | sun |
| AMIUpdateHours | 0 |
| AMIMaxAge | 7 |
| EnableAutoScaling | False |
| InstanceCount | 1 |
| HealthCheckEndpoint | / |
| SSLCertificateArn | arn:aws:iam::${your-account-id}:server-certificate/EJBSelfSignedCert|
| Subnetids | Select a single subnet with an IP range starting with 10.0  |
| CPULow | 35 |
| CPUHigh | 80 |
| EnableStickySessions | False |
| StickySessionCookieName | JSESSIONID |
| RDS | False |
| InstanceVolumeSize | 10 |
| UserData | leave blank |

* Then press `NEXT`
* Confirm the Tag Options by pressing `NEXT`
* Leave notifications unchecked and press `NEXT`
* Review the parameters and press `LAUNCH`

Your product launch will be complete once its status changes to Succeeded.  
The page can be refreshed by pressing the button labeled with 2 orange arrows.
![](images/refresh.png?raw=true)

### Launch Code Deploy Application and Deployment Group
Code Deploy needs to be configured to know what Auto Scaling Group to deploy to.
As above, we will launch a Service Catalog product. However, before we proceed, we need to 
know the name of the Auto Scaling Group which we have just created.  

#### Find Auto Scaling Group Name
The easiest way to do this is from Service Catalog.

* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named `${your-username}-asg`
* Scroll down and find the Outputs section.
* Copy to the clipboard the value for the key named `AutoScalingGroupName`.

#### Launch Product
* In the AWS Console, select Services, Service Catalog
* Click on Products List in the tool bar on the left
* Click on the `codedeploy` Product
* Click `Launch Product`
* Enter a Provisioned Product Name
   * ${your-username}-code-deploy e.g. user5-code-deploy
* Select the latest Product Version
* Press `NEXT`
* Enter Parameters below

|Parameter | Value |
| --- | --- |
| SealId | 00001 |
| AutoScalingGroupName | paste from clipboard |
| CustomResourceMetadata |leave blank |
| DeploymentConfig | OneAtATime |

* Then press `NEXT`
* Confirm the Tag Options by pressing `NEXT`
* Leave notifications unchecked and press `NEXT`
* Review the parameters and press `LAUNCH`


### Setup Application for Code Deploy
Your application needs to be enhanced in order for it to be deployable with Code Deploy.

Create the following directories inside of your application - `src/main` will already exist.
* src/main/code-deploy
* src/main/code-deploy/scripts
* src/main/code-deploy/appspec
* src/main/code-deploy/assembly

At the end of this task, you'll end up with a directory structure like this.

```
aws-basics
│   README.md
│   pom.xml    
│
└───src
│   └───main
│       └───java
│       │   │      ...
│       └───resources
│       │   │───application.properties
│       └───code-deploy
│           └───scripts
│           │   │───clean_up.sh
│           │   │───start_server.sh
│           │   │───stop_server.sh
│           └───appspec
│           │   │───appspec.yml
│           └───assembly
│               │─── zip.xml
```


#### appspec.yml
AWS Code Deploy requires a deployment descriptor named `appspec.yml` that describes
the deployment lifecycle.

In the appspec directory, create a file named `appspec.yml` and copy the contents below.
Remember this is a YAML file, be careful about indentation!
```yaml
version: 0.0
os: linux
files:
  - source: /lib
    destination: /opt/aws-basics
hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
      timeout: 120
      runas: root
    - location: scripts/clean_up.sh
      timeout: 120
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 120
      runas: root

```

#### Bash Scripts
Code Deploy calls shell scripts during the deployment lifecycle.

We are going to create 3 files in the scripts directory.

##### start_server.sh
Warning, do not just copy an paste the script below, the name of the jar file will vary depending on your 
username.
```
#!/bin/bash
java -jar /opt/aws-basics/${your-username}-aws-basics.jar 2> /dev/null > /dev/null < /dev/null &
```

##### stop_server.sh
```
#!/bin/bash

# Stop server
lab_running=`pgrep -f aws-basics`
if [[ -n  $lab_running ]]; then
    pkill -f aws-basics
fi
```

##### clean_up.sh
Warning, do not just copy an paste the script below, the name of the jar file will vary depending on your 
username.
```
#!/bin/bash

if [ -f /opt/aws-basics/${your-username}-aws-basics.jar ]; then rm /opt/aws-basics/${your-username}-aws-basics.jar; fi

```

#### Maven Assembly Plugin
Code Deploy requires that application binaries, scripts and deployment descriptor are packaged 
in either a tarball or a zip file. We'll use the Maven Assembly plugin to create a zip file.

You'll need to edit pom.xml in the root of your application. Add the maven-assembly-plugin to the 
plugins section as in the example below. The spring-boot-maven plugin will already be defined.  

The easiest way to do this is to replace the entire build section. If you just add the assembly plugin,
remember to add the `finalName` element or your jar file in the zip will be incorrectly named. 

##### POM.XML
```xml
    <build>
        <finalName>${project.artifactId}</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>2.6</version>
                <executions>
                    <execution>
                        <id>make-zip</id>
                        <phase>package</phase>
                        <goals>
                            <goal>
                                single
                            </goal>
                        </goals>
                        <configuration>
                            <appendAssemblyId>false</appendAssemblyId>
                            <descriptors>
                                <descriptor>src/main/code-deploy/assembly/zip.xml</descriptor>
                            </descriptors>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

##### zip.xml
This file configures the assembly plugin to :-
* Create a zip file in the target dir
* Copy appspec.yml to the root
* Copy the application jar file to /lib
* Copy the scripts to /scripts

Create a file called zip.xml in assembly folder and copy the contents below.

```xml
<assembly xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3"
          xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3 http://maven.apache.org/xsd/assembly-1.1.3.xsd">
    <id>zip</id>
    <formats>
        <format>zip</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <fileSets>
        <fileSet>
            <directory>${project.build.directory}</directory>
            <includes>
                <include>${project.artifactId}.jar</include>
            </includes>
            <outputDirectory>lib</outputDirectory>
        </fileSet>
        <fileSet>
            <directory>${basedir}/src/main/code-deploy/scripts</directory>
            <includes>
                <include>*.sh</include>
            </includes>
            <outputDirectory>scripts</outputDirectory>
            <lineEnding>unix</lineEnding>
        </fileSet>
        <fileSet>
            <directory>${basedir}/src/main/code-deploy/appspec</directory>
            <includes>
                <include>appspec.yml</include>
            </includes>
            <outputDirectory>.</outputDirectory>
            <lineEnding>unix</lineEnding>
        </fileSet>
    </fileSets>
</assembly>
```

### Deploy
Next we need to build the application and zip file.
Open a Command or Terminal window, change to the directory that contains your application.
Hint - it has pom.xml in it.
Type the following command
```
mvn package
```
If you have been successful, a file named `${your-username}-aws-basics.zip` will be created in the target folder.

#### Copy zip file to S3
We will now upload the zip file to the code S3 bucket.
 
* In the AWS Console, Select Services, S3
* Click on the bucket whose name starts with `apollo-fcn-cd-code-uu-id-`
   * Note the actual bucket name will have a random number appended to its name
* Click on the `00001` folder 
* Click `Upload`
* Click `Add Files`
* Navigate to the zip file `target/${your-username}-aws-basics.zip` and press Open
* Press `Next`
* Press `Next`
* Under Encryption select `Amazon S3 Master Key`
   * Server side encryption is mandatory - this selects the key to use
* Press `Next`
* Press `Upload`

Your zip file should now be in the `apollo-fcn-cd-code-uu-id-XXXXX/00001` S3 bucket.
 

#### Create a CodeDeploy Deployment

Quick recap.
* We have created a Spring Boot application ready for Code Deploy
* Provisioned an Auto Scaling Group
* Provisioned a CodeDeploy Application and Deployment Group
* Copied zip file to S3.

The next task is to deploy the zip file to the EC2 instance(s) in the Auto Scaling Group.  
To complete this task, you'll need 2 pieces of information :
1. URL of your zip file in S3 
2. Name of you CodeDeploy Application.  

It is recommended that you paste these values into a temporary text document. 

##### Get S3 URL of the CodeDeploy zip file 
* In the AWS Console, Select Services, S3
* Click on Bucket whose name starts with `apollo-fcn-cd-code-uu-id-`
* Click on Folder named `00001` 
* To show versions, click on the `Show` button next to the Version label
* Click on the latest version of your zip file - the one that starts with your username.
* Make a note of the Link at the bottom of the page

##### Get Name of CodeDeploy Application
* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your CodeDeploy, it will be named ${your-username}-code-deploy
* Scroll down and find the Outputs section.
* Make a note of the value of the key named Application.

##### Create Deployment
* In the AWS Console, select Services, CodeDeploy
* Click on Applications in the left-hand menu (this may be selected by default).
![](images/code-deploy-applications.png?raw=true)
* Paste your application name in the search text box and select it.
![](images/code-deploy-application.png?raw=true)
* Select the `Deployments tab`
* Press the `Create deployment` button
![](images/code-deploy-create-deployment.png?raw=true)
* In the Deployment groups drop-down, select your Deployment Group - there should be only one. 
* Ensure the Revision Type `My Application is is stored in Amazon S3` is selected.
* Paste the S3 URL into the `Revision Location` text box. 
![](images/code-deploy-deployment.png?raw=true)
* Leave all other settings default.
* Press the orange `Create deployment` button at the bottom of the screen.

All being well you should see a green Deployment Succeeded message.
![](images/code-deploy-succeeded.png?raw=true)

#### Test Running Application
In order to test our deployment, we need to get the URL of the Load Balancer.  

* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named ${your-username}-asg
* Scroll down and find the Outputs section.
* Copy the value of the URL key to the clipboard.

In your browser paste the load balancer URL. You will be presented with the same SSL certificate warning as before.  
Ignore the warning and you should see a response similar to below.

```json
{
  "_links" : {
    "profile" : {
      "href" : "https://sc-198952-elasticl-1fqdz6nnkmk5j-234455335.us-east-1.elb.amazonaws.com/profile"
    }
  }
}
```

### Scale Horizontally
#### Increase Number of EC2 Instances
We will use Service Catalog to increase the number of running EC2 instances in our Auto Scaling Group.
* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named `${your-username}-asg`
* On the right of the screen click on the down-arrow on the `ACTIONS` button and select Update
* Select the latest Product Version
* Press the `NEXT` button
* Scroll down to the Scaling Configurations section and locate the `InstanceCount` parameter
* Change the value from 1 to 2
* Then press the `NEXT` button
* Then press `UPDATE`

To validate that scale-up has occurred, we'll look at the load balancer, we should see 2 in-service instances.  
It will take a couple of minutes for the EC2 instance to spin-up and for the load balancer health-check to put it in-service.
* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named `${your-username}-asg`
* In the Outputs section locate the key named `CloudformationStackARN`
* The value is a link, click on it
   * You'll now be navigated to the CloudFormation Stack that Service Catalog created
* Expand the Resources section
* Find the resource with a Logical ID of `ElasticLoadBalancer`, the Physical ID will be a link, click it.
* You'll be navigated to the Load Balancer screen
* Click on the `Instances` tab and you should see 2 in-service instances.
![](images/scale-up.png?raw=true)

### Create Rest Endpoint and Redeploy (optional)
Create a file named `HexConverter.java` in `src/main/java/com/example/${your-username}awsbasics`, the directory should 
already exist.  
Copy the contents below into the newly created file. Make sure you enter your username in the package name.
E.g. `com.example.user5awsbasics`.

```java
package com.example.${your-username}awsbasics;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HexConverter {
    @GetMapping("/api/tohex")
    public String toHex(@RequestParam("number") int intToParse) {
        return Integer.toHexString(intToParse);
    }

    @GetMapping("/api/fromhex")
    public String fromHex(@RequestParam("number") String hexToParse) {
        int i = Integer.parseInt(hexToParse, 16);
        return Integer.toString(i);
    }
}
```

#### Rebuild and Deploy the Application

```
mvn package
```
Copy the newly created zip file to S3 as in the step above.  
Create a new Deployment as above.

#### Test the Endpoint
Enter this URL into your browser, the load balancer URL was discovered in a task above.

```
${load-balancer-url}/api/tohex?number=255
```
The response in the browser will be 
```
ff
```
Now test the `fromhex` endpoint.
```
${load-balancer-url}/api/fromhex?number=2a
```
The response in the browser will be 
```
42
```

#### Optional - Improve Error Handling
What happens if an invalid number is passed to the endpoint?  
Enhance the code to return a more meaningful response.

### SSH onto EC2 (optional)
#### Get EC2 IP Address
* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named `${your-username}-asg`
* In the Outputs section locate the key named CloudformationStackARN
* The value is a link, click on it
   * You'll now be navigated to the CloudFormation Stack that Service Catalog created
* Expand the Resources section
* Locate the resource with a Logical ID of AutoScalingGroup
* The Physical ID for this resource is a link, click on it
   * This will take you to the Auto Scaling Group Details
* Click on the Instances tab at the bottom of the screen
   * The EC2 instances in the ASG will be listed 
* The Instance ID for an instance is a link, click it
   * Details of the EC2 instance will now be displayed
* Find the `IPv4 Public IP` property at the bottom of the screen and copy the IP Address

#### Download ssh key-pair from S3.  
* In the AWS Console, Select Services, S3
* Click on Bucket whose name starts with `apollo-fcn-cd-code-uu-id-`
* Click on Folder named `00001`
* Mac
   - click on `aws-basics-lab-key-pair.pem`
   - download to ~/.ssh
   - the file may get renamed as it downloads to`aws-basics-lab-key-pair.cer`, rename it back to .pem.  
* Windows
   - click on `aws-basics-lab-key-pair.ppk`
   - download to a temp folder

#### Windows Only 
#####  Install Putty
Download from [https://www.putty.org/](https://www.putty.org/)  

##### Connect Using Putty
Open Putty
In the `Host Name (or IP Address)` text box enter:  
`ec2-user@${EC2 IPv4 Public IP}`   
(substitute with the IP address of your EC2 instance)  
![](images/putty-1.png?raw=true)

Then click on Connection, SSH, Auth and enter the path to the PPK file you downloaded in the step above.
and press the `Open` button.  

![](images/putty-2.png?raw=true)  

#### Mac / Linux Only 
##### Set permissions on the key pair
```
chmod 600 ~/.ssh/aws-basics-lab-key-pair.pem
```
##### SSH onto EC2
Open a Terminal and enter the following:  
`ssh -i ~/.ssh/aws-basics-lab-key-pair.pem ec2-user@${EC2 IPv4 Public IP}`   
(substitute with the IP address of your EC2 instance)

#### Useful Directories
Your application will copied to `/opt/aws-basics/`, this is specified in `appspec.yml`.  
You can manually start your application from this directory.

CodeDeploy deployments
`/opt/codedeploy-agent/`
Your deployments will be copied to:  
`/opt/codedeploy-agent/deployment-root/${deployment-group-uuid}/${deployment-id}`  
Inside this directory you'll see deployment logs and the unpacked zip file.  
This is the best place to look if you have CodeDeploy issues.

CloudFormation log
`/var/log/cfn-init.log`

CodeDeploy logs
`/var/log/aws/codedeploy-agent`


#### Enable Logging (Optional)
The skeleton application generated in the first task included the Actuator and Rest starters.
These starters depend on `spring-boot-starter-logging` so there is very little work required to enable logging.

##### Edit application.properties
Edit the application.properties file in the src/main/resources directory.
Add the following text, remembering to substitute your user id.

```properties
logging.level.org.springframework=INFO
logging.level.com.example.${your-username}awsbasics=DEBUG
```
This will enable logging for the Spring framework and your application.

##### Add some logging to your application

Add the following imports to the `${your-username}AwsBasicsApplication` class.
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
```

Add the following declaration to the same class - remember to replace the placeholder with your user id.
 
```java
private static final Logger LOG = LoggerFactory.getLogger(${your-username}AwsBasicsApplication.class);
```

Add the following as the last line in the main method, after the `SpringApplication.run` command.

```java
LOG.debug("******* Application Started *******");
```
Run the application locally as in the task above and observe the logging.
 
##### Configure a File Appender
By default, all logging goes to the console which is useful when the application is 
run locally, however is not much use when running in the Cloud.

Add the following line to application.properties in the `src/main/resources` directory.
```properties
logging.file=/var/log/aws-basics/application.log
```
The `start_server.sh` script will also need to be modified. It will:
* Create a logging directory
* Rotate the log file if it exists
* Touch the file
  
Remember to substitute your username in the jar file name. 

```
#!/bin/bash
mkdir -p /var/log/aws-basics
if [ -f /var/log/aws-basics/application.log ]; then mv /var/log/aws-basics/application.log "/var/log/aws-basics/application.log.$(date +"%Y%m%d_%H%M%S")"; fi
touch /var/log/aws-basics/application.log

java -jar /opt/aws-basics/${your-username}-aws-basics.jar 2> /dev/null > /dev/null < /dev/null &
```

Rebuild your application and re-deploy as in the steps above.
SSH onto an EC2 instance.

```
cd /var/log/aws-basics
less application.log 
```
Hint, when using `less`:
* down / up arrow moves a line at a time
* space moves a page at a time
* q quits  

[More information on less](https://en.wikipedia.org/wiki/Less_(Unix))

### Configure CodeDeploy to wait for successful deployment
You may have noticed that CodeDeploy reports a successful deployment even if your application fails to start.  
A simple solution is to modify the start script to exit 0 only on successful application start.   
Note, the logging exercise must have already been completed for this to work.    
The log file is tailed until specific text is found. If the text is not found, CodeDeploy will time-out.

Add the following to `start_server.sh` after java -jar is called.
   
```
tail -f /var/log/aws-basics/application.log | while read LOGLINE
do
    [[ "${LOGLINE}" == *"Tomcat started on port(s): 443 (https)"* ]] && pkill -P $$ tail
done
exit 0
```

Re-deploy your application.

### Clean-up
### Terminate ASG
* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on the button with 3 vertical dots to the left your Auto Scaling Group, it will be named `${your-username}-asg`
![](images/service-catalog-pp-update.png?raw=true)  
* Select `Terminate provisioned product`

### Terminate CodeDeploy
* As above but for the CodeDeploy provisioned product, it will be named `${your-username}-code-deploy`

