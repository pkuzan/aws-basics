## AWS Basics

### Prerequisites
The following will require Java 8 and Maven to be installed.
An IDE such as Intellij is useful, but not mandatory.
#### Java
Download from [https://www.java.com/en/download/](https://www.java.com/en/download/)

#### Maven
Download from [https://maven.apache.org/download.cgi](https://maven.apache.org/download.cgi)
Select the Binary zip archive.
Unzip   
Add to path TODO

#### IDE (Optional)
Download Intellij Community from [https://www.jetbrains.com/idea](https://www.jetbrains.com/idea)

#### Credentials
You will have been provided AWS authentication details.

* AWS Account ID
* Username
* Temporary password 

#### Conventions
In the subsequent tasks, please substitute :
* ${your-sid} with your SID
* ${your-account-id} with your AWS Account ID 

### Intro to Lab
AWS blah blah

### Generate Spring Boot Application.
In your browser, navigate to [https://start.spring.io/](https://start.spring.io/)
Change the artifact name to ${your-sid}-aws-basics and add Actuator and Rest Repositories dependencies.
Press the Generate Project button and a zip file named ${sid}-aws-basics.zip will be downloaded.
![](images/initializr-create.png?raw=true)

Unzip the ${your-sid}-aws-basics.zip file.
Open a Command or Terminal
Change directory to the directory where the zip file was unzipped.
Execute the following commands.
```
mvn package
java -jar target/${your-sid}-aws-basics-0.0.1-SNAPSHOT.jar
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
java -jar target/${sid}-aws-basics-0.0.1-SNAPSHOT.jar
```
Note, if you are running a Mac or Linux, you may have to run the application with elevated privileges, as ports
less than 1024 are restricted. 
```
#Mac or Linux 
sudo java -jar target/${your-sid}-aws-basics-0.0.1-SNAPSHOT.jar
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

* Login to the AWS Console [https://console.aws.amazon.com](https://console.aws.amazon.com)  
   * Ensure you are in the N. Virginia Region (US_East-1)
   * Use credentials you were given before the workshop
* In the AWS Console, select Services, Service Catalog
* Click on Products List in the tool bar on the left
   * To display the toolbar, click on the 3 horizontal lines on the top left of the screen 
   * ![](images/service-catalog-products.png?raw=true)
* Click `auto_scaling` Product Name
* Click `Launch Product` Button
* Enter a Provisioned Product Name
   * `${your-sid}-asg e.g. a123456-asg`
* Select the latest Product Version
* Press `NEXT`
* Enter Parameters below
  * Remember to substitute your account id in the `SSLCertificateArn` parameter value

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

### Launch Code Deploy Application and Deployment Group
Code Deploy needs to be configured to know what Auto Scaling Group to deploy to.
As above, we will launch a Service Catalog product. However, before we proceed, we need to 
know the name of the Auto Scaling Group which we have just created.  

#### Find Auto Scaling Group Name
The easiest way to do this is from Service Catalog.

* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named `${your-sid}-asg`
* Scroll down and find the Outputs section.
* Copy to the clipboard the value for the key named `AutoScalingGroupName`.

#### Launch Product
* In the AWS Console, select Services, Service Catalog
* Click on Products List in the tool bar on the left
* Click on the `codedeploy` Product
* Click `Launch Product`
* Enter a Provisioned Product Name
   * ${your-sid}-code-deploy e.g. a123456-code-deploy
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

Create the following directories inside of your application - src/main will already exist.
* src/main/code-deploy
* src/main/code-depoly/scripts
* src/main/code-depoly/appspec
* src/main/code-depoly/assembly

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
AWS Code Deploy requires a deployment descriptor named `appsepc.yaml` that describes
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

#### bash scripts
Code Deploy calls shell scripts during the deployment lifecycle.

We are going to create 3 files in the scripts directory.

##### start_server.sh
Warning, do not just copy an paste the script below, the name of the jar file will vary depending on your 
sid.
```
#!/bin/bash
java -jar /opt/aws-basics/${your sid}-aws-basics.jar 2> /dev/null > /dev/null < /dev/null &
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
sid.
```
#!/bin/bash

if [ -f /opt/aws-basics/${your sid}-aws-basics.jar ]; then rm /opt/aws-basics/${your sid}-aws-basics.jar; fi

```

#### Maven Assembly Plugin
Code Deploy requires that application binaries, scripts and deployment descriptor are packaged 
in either a tarball or a zip file. We'll use the Maven Assembly plugin to create a zip file.

You'll need to edit pom.xml in the root of your application. Add the maven-assembly-plugin to the 
plugins section as in the example below. The spring-boot-maven plugin will already be defined.
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
mvn install
```
If you have been successful, a file named ${your-sid}-aws-basics.zip will be created in the target folder.

#### Copy zip file to S3
We will now upload the zip file to the code S3 bucket.
 
* In the AWS Console, Select Services, S3
* Click on the bucket whose name starts with `apollo-fcn-cd-code-uu-id-`
   * Note the actual bucket name will have a random number appended to its name
* Click on the 00001 folder 
   * 00001 represents your Seal Id
* Click Upload
* Click Add Files
* Navigate to the zip file `target/${your-sid}-aws-basics.zip` and press Open
* Press Next
* Press Next
* Under Encryption select Amazon S3 Master Key
   * Server side encryption is mandatory - this selects the key to use
* Press Next
* Press Upload

Your zip file should now be in the /00001 S3 folder.
 

#### Create a CodeDeploy Deployment

Quick recap.
* Created a Spring Boot application ready for Code Deploy
* Provisioned an Auto Scaling Group
* Provisioned Code Deploy Application and Deployment Group
* Copied zip file to S3.

The next task is to deploy the zip file to the EC2 instance(s) in the Auto Scaling Group.  
To complete this task, you'll need 2 pieces of information :
1. URL to you zip file in S3 
2. Name of you CodeDeploy Application.  

It is recommended that you paste these values into a temporary text document. 

##### Get S3 URL of the CodeDeploy zip file 
* In the AWS Console, Select Services, S3
* Click on Bucket whose name starts with apollo-fcn-cd-code-uu-id-
* Click on Folder named 00001 (this represents your seal id)
* To show versions, click on the `Show` button next to the Version label
* Click on the latest version of your zip file - the one that starts with your SID.
* Make a note of the Link at the bottom of the page

##### Get Name of CodeDeploy Application
* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your CodeDeploy, it will be named ${your-sid}-code-deploy
* Scroll down and find the Outputs section.
* Make a note of the value of the key named Application.

##### Create Deployment
* In the AWS Console, select Services, CodeDeploy
* Paste your application name in the search text box and select it.
* Select your Deployment Group radio button - there should only be one. 
* Press the down-arrow on Actions button and select Deploy new revision.
* Add the following parameters:

|Parameter | Value |
| --- | --- |
| Application | leave default value |
| Compute platform | EC2/On-premises |
| Deployment group | leave default value |
| Deployment type | in-place |
| Repository type | My application is stored in Amazon S3 |
| Revision location | paste the S3 URL from the above step |
| File Type | .zip |
| Deployment description | Add some witty comment if you feel obliged |
| Deployment configuration | CodeDeployDefault.OneAtATime |

* Leave all other settings default.
* Press the Deploy button.

For more information click on the Deployment ID link.  
All being well you should see a green Deployment Succeeded message.
![](images/deployment-succeeded.png?raw=true)

#### Test Running Application
In order to test our deployment, we need to get the URL of the Load Balancer.  

* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named ${your-sid}-asg
* Scroll down and find the Outputs section.
* Copy the value of the URL key to the clipboard.

In your browser paste the URL. You will be presented with the same SSL certificate warning as before.  
Ignore the error and you should see a response similar to below.

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
* Click on your Auto Scaling Group, it will be named ${your-sid}-asg
* On the right of the screen click on the down-arrow on the `ACTIONS` button and select Update
* Select the latest Product Version
* Press the `NEXT` button
* Scroll down to the Scaling Configurations section and locate the `InstanceCount` parameter
* Change the value from 1 to 2
* Then press the `NEXT` button
* Then press `UPDATE`

TODO - Check the new instance in ASG and ELB instances in-service 

### Scale Vertically
#### Increase EC2 Instance Size
As above, Service Catalog will be used to increase the size of the EC2 instances in our Auto Scaling Group.
* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named ${your-sid}-asg
* On the right of the screen click on the down-arrow on the `ACTIONS` button and select Update
* Select the latest Product Version
* Press the `NEXT` button
* Scroll down to the  EC2 Instance Configuration section and locate the `InstanceType` parameter
* Change the value from t2.micro to t2.small
* Then press the `NEXT` button
* Then press `UPDATE`

TODO Check the size in ASG

### Create Rest Endpoint and Redeploy (optional) Needs Work
Create a file named `EchoController.java` in `src/main/java/com/example/${your-sid}awsbasics`, the directory should 
already exist.  
Copy the contents into the newly created file. Make sure you enter your SID in the package name.

```java
package com.example.${your-sid}awsbasics;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class EchoController {

    @GetMapping("/api/echo")
    public String echo(@RequestParam("input") String input) {
        return input;
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
${load-balancer-url}/api/echo?input=hello
```
The response in the browser should be 
```
hello
```

### SSH onto EC2 (optional)
#### Get EC2 IP Address
* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named ${your-sid}-asg
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
* Find the IPv4 Public IP property at the bottom of the screen and copy the IP Address

#### Download ssh key-pair from S3.  
* In the AWS Console, Select Services, S3
* Click on Bucket whose name starts with apollo-fcn-cd-code-uu-id-
* Click on Folder named 00001 (this represents your seal id)
* Click on aws-basics-lab-key-pair.pem
* Click download 
   * On a Mac or Linux download to ~/.ssh

#### Windows Only 
#####  Install Putty
Download from [https://www.putty.org//](https://www.putty.org/)  
TODO - Someone with a Windows machine!

##### Convert PEM to PPK
TODO - Someone with a Windows machine!
* Open PEM in Putty
* Export as PPK 

##### Connect Using Putty
TODO - Someone with a Windows machine!

#### Mac / Linux Only 
##### Set permissions on the key
```
chmod 600 ~/.ssh/aws-basic-lab-key-pair.pem
```
##### SSH onto EC2
```
ssh -i ~/.ssh/aws-basic-lab-key-pair.pem ec2-user@${ec2-ip-address}
```
#### Useful Directories

/opt/codedeploy-agent/

#### Enable Logging (Optional)
TODO