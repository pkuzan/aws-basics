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

### Intro to Lab
AWS blah blah

### Generate Spring Boot Application.
In your browser, navigate to [https://start.spring.io/](https://start.spring.io/)
In this and subsequent tasks, please substitute ${sid} with your SID.
Change the artifact name to ${sid}-aws-basics and add Actuator and Rest Repositories dependencies.
Press the Generate Project button and a zip file named ${sid}-aws-basics.zip will be downloaded.
![](images/initializr.png?raw=true)

Unzip the ${sid}-aws-basics.zip file.
Open a Command or Terminal
Change directory to the directory where the zip file was unzipped.
Execute the following commands.
```
mvn package
java -jar target/${sid}-aws-basics-0.0.1-SNAPSHOT.jar
```

Insert the URL below into your browser
```
http://localhost:8080/actuator/health
```
If the applications has started successfully you should see the following response.
```
{"status":"UP"}
```

Stop the application by pressing CTRL + C in the Terminal / Command window.

### Add SSL Support
Public cloud only permits https on port 443. To enable this, we need to generate a self-signed certificate and configure the 
application to use the certificate and port 443.

##### application.properties
Edit the application.properties file in the src/main/resources directory.
Add the following text.

```
server.port=443
server.ssl.key-alias=selfsigned
server.ssl.key-password=password
server.ssl.key-store=classpath:keystore.jks
server.ssl.key-store-provider=SUN
server.ssl.key-store-type=JKS
```

##### Generate self-signed SSL Certificate

Open a Command or Terminal window, change directory to src/main/resources and enter the following command.
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

Enter yes when prompted and a file named keystore.jks will be created.

##### Start the Application
As before, build and start the application.

```
mvn package
java -jar target/${sid}-aws-basics-0.0.1-SNAPSHOT.jar
```
Note, if you are running a Mac or Linux, you may have to run the application with elevated privileges, as ports
less than 1024 are restricted. 
```
#Mac or Linux 
sudo java -jar target/${sid}-aws-basics-0.0.1-SNAPSHOT.jar
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

#### Login to AWS Console
TODO
[https://console.aws.amazon.com](https://console.aws.amazon.com)  
Ensure you are in the N. Virginia Region (US_East-1)

* In the AWS Console, select Services, Service Catalog
* Click on Products List in the tool bar on the left
* Click auto_scaling
* Click Launch Product
* Enter a Provisioned Product Name
   * ${sid}-asg e.g. a123456-asg
* Select the latest Product Version
* Press Next
* Enter Parameters below

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
| SSLCertificateArn | arn:aws:iam::198952797270:server-certificate/EJBSelfSignedCert|
| Subnetids | TODO |
| CPULow | 35 |
| CPUHigh | 80 |
| EnableStickySessions | False |
| StickySessionCookieName | JSESSIONID |
| RDS | False |
| InstanceVolumeSize | 10 |
| UserData | leave blank |

* Then press next
* Confirm the Tag Options by pressing next
* Leave notifications unchecked and press next
* Review the parameters and press launch

### Launch Code Deploy Application and Deployment Group
Code Deploy needs to be configured to know what Auto Scaling Group to deploy to.
As above, we will launch a Service Catalog product. However, before we proceed, we need to 
know the name of the Auto Scaling Group which we have just created.  

#### Find Auto Scaling Group Name
The easiest way to do this is from Service Catalog.

* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your Auto Scaling Group, it will be named ${sid}-asg
* Scroll down and find the Outputs section.
* Copy to the clipboard the value for the key named AutoScalingGroupName.

#### Launch Product
* In the AWS Console, select Services, Service Catalog
* Click on Products List in the tool bar on the left
* Click codedeploy
* Click Launch Product
* Enter a Provisioned Product Name
   * ${sid}-code-deploy e.g. a123456-code-deploy
* Select the latest Product Version
* Press Next
* Enter Parameters below

|Parameter | Value |
| --- | --- |
| SealId | 00001 |
| AutoScalingGroupName | paste from clipboard |
| CustomResourceMetadata |leave blank |
| DeploymentConfig | OneAtATime |

* Then press next
* Confirm the Tag Options by pressing next
* Leave notifications unchecked and press next
* Review the parameters and press launch


### Setup Application for Code Deploy
Your application needs to be enhanced in order for it to be deployable with Code Deploy.


Create the following directories inside of your application.
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
AWS Code Deploy requires a deployment descriptor named appsepc.yaml that describes
the deployment lifecycle.

In the appspec directory, create a file named appspec.yml and copy the contents below.
```
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
plugins section as in the example below.
##### POM.XML
```
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

```
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
First thing we need to do is build the application and zip file.
Open a Command or Terminal window, change to the directory that contains your application.
Hint - it has pom.xml in it.
Type the following command
```
mvn install
```
If you have been successful, a file named ${sid}-aws-basics.zip will be created in the target folder.

#### Login to AWS Console
TODO
[https://console.aws.amazon.com](https://console.aws.amazon.com)

#### Copy zip file to S3
We will now upload the zip file to the code S3 bucket.
 
* In the AWS Console, Select Services, S3
* Click on the apollo-fcn-cd-code-uu-id- bucket
   * Note the actual bucket name will have a random number appended to its name
* Click on the 00001 folder 
   * 00001 represents your Seal Id
* Click Upload
* Click Add Files
* Navigate to the zip file target/${sid}-aws-basics.zip and press Open
* Press Next
* Press Next
* Under Encryption select Amazon S3 Master Key
   * Server side encryption is mandatory - this selects the key to use
* Press Next
* Press Upload

Your zip file should now be in the /00001 S3 folder.
 

#### Create a Deployment

Quick recap.
* Made the Spring Boot application ready for Code Deploy
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
* To show version, click on the Show button next to the Version label
* Click on the latest version of your zip file - the one that starts with your SID.
* Make a note of the Link at the bottom of the page

##### Get Name of CodeDeploy Application
* In the AWS Console, select Services, Service Catalog
* Select Provisioned products list 
* Click on your CodeDeploy, it will be named ${SID}-code-deploy
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
* Click on your Auto Scaling Group, it will be named ${SID}-asg
* Scroll down and find the Outputs section.
* Copy the value of the URL key to the clipboard.

In your browser paste the URL. You will be presented with the same HTTPS certificate warning as before.  
Ignore the error and you should see a response similar to below.

```
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

### Scale Vertically

#### Increase EC2 Instance Size

### Create Rest Endpoint and Redeploy (optional)

### SSH onto EC2 (optional)

Download ssh key-pair from S3.  
Copy to ~/.ssh  
Set permissions  

```
chmod 600 ~/.ssh/aws-basic-lab-key-pair.pem
```

```
ssh -i ~/.ssh/aws-basic-lab-key-pair.pem ec2-user@${ec2-ip-address}
```

/opt/codedeploy-agent/