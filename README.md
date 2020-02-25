# Springboot Application Logging in OKE to Oracle Object Storage with Logback and Fluentd

This demo showcases how to configure the logging of a Spring Boot application (running in OKE) to Oracle Object Storage using Logback and Fluentd. The log sent to stdout will be in structured format which is JSON to enable easier integration with log aggregation and log indexing tools later on. Fluentd is used scrape the logs before sending to these tools, however, in this demo, Fluentd will send it to Oracle Cloud Infrastructure Object Storage.

## Demo Overview

This demo uses a Spring Boot Application in the previous demo (https://github.com/eugeniapay/spring-rest-service). The steps below will show how to configure Logback and Fluentd for this Spring Boot application which is running on OKE to Oracle Object Storage using S3 API.

The completed demo can be found at Github https://github.com/eugeniapay/spring-rest-service-logback.

This steps have been tested on Oracle Cloud Infrastructure Gen 2 as of 21 Feb 2020. Should you encounter any problems, please feel free to feedback to Eugenia (eugenia.pay@oracle.com).

[Special Note] This is a work-in-progress version, more details will be provided in the near future. Please stay tuned.

## Pre-Requisite

Before you can proceed with the following demo, please ensure that you have the following aaccounts:-
- OCI Gen 2 Account with Container Pipeline (Wercker), OKE, OCI Registry Access.
- GitHub Account

In addition, you should also have completed the following two guides:-
- http://github.com/yyeung13/oke_ocir_demo: Setup OCIR/OKE as well as the OCI CLI to manage the kubernetes cluster
- https://github.com/yyeung13/wercker_oke_demo: Use Wercker, Oracle Kubernetes Engine and OCI Registry for complete DevOps of a demo Springboot application
  
Lastly, you should have Fluentd image (fluent/fluentd-kubernetes-daemonset:v1.8.1-debian-s3-1.0) pushed into OCIR. Please refer to https://github.com/yyeung13/oke_ocir_demo > Step 7 on how to push images to OCIR.

## Demo Details

### Oracle Object Storage
1. Login to OCI from http://cloud.oracle.com. You will need your OCI Gen 2 tenancy name, username and password to login. Should you require a trial account, you can contact Oracle.

2. Upon Successful Login, navigate from the hamburger icon on the top left hand corner to 'Object Storage' -> 'Object Storage'.

3. Create New Bucket. (Please ensure you are in the right compartment)
    - Click on 'Create Bucket'
    - Specify the following:
      - Name: epdemo_bucket (You will assign it to a env variable named 'S3_BUCKET' later)
      - Storage Tier: Standard
      - Encryption: Encrypt Using Oracle Managed Keys
    - Click 'Create Bucket'
    - You should see the new bucket created with visibility as 'Public'

4. Ensure you have configured a Policy to manage this bucket in OCI.

```
Allow group [YOUR GROUP] to manage buckets in tenancy

Allow group [YOUR GROUP] to manage objects in tenancy

```

5. Nagivate from the profile icon on the top right hand corner to 'Tenancy: XXX'.

6. Under Object Storage Settings, copy the Object Storage Namespace. You will assign it to a env variable named 'S3_ENDPOINT' later.

7. From the top right menu bar, navigate from the region displayed > 'Manage Regions'.

8. Copy the Region Identifier that you are in (The region which you are in will be displayed as the first row and bold). You will assign it to a env variable named 'S3_BUCKET_REGION' later.

### OCI - Customer Secret Keys
1. At OCI Console, nagivate from the profile icon on the top right hand corner to 'User Settings'.

2. Click 'Customer Secret Keys' > 'Generate Secret Keys'.

3. Enter a friendly description for the key and click 'Generate Secret Key'. The generated Secret Key is displayed in the Generate Secret Key dialog box. At the same time, Oracle generates the Access Key that is paired with the Secret Key. The newly generated Customer Secret key is added to the list of Customer Secret Keys.

4. Copy the Secret Key immediately, because you can't retrieve the Secret Key again after closing the dialog box for security reasons and you will assign it to a env variable named 'AWS_SECRET_ACCESS_KEY' later.

5. Click 'Close'.

6. Copy the Access Key by clicking the Show or Copy action to the left of the Name of a particular Customer Secret key. You will assign it to a env variable named 'AWS_SECRET_ACCESS_ID' later.

### Fluentd Deployment
1. Ensure Fluentd image (fluent/fluentd-kubernetes-daemonset:v1.8.1-debian-s3-1.0) is pushed into OCIR. Please refer to https://github.com/yyeung13/oke_ocir_demo > Step 7 on how to push images to OCIR.

2. Create a file called 'clusterrole.yaml':-
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: default
  namespace: default
```

3. Change the namespace if you are using another one.

4. From OCI CLI terminal, run 'kubectl create -f clusterrole.yaml'.

5. Create a file called 'configmap.yaml' to provide custom Fluentd configurations:-
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-config
  namespace: default
data:
  fluent.conf: |
    @include "#{ENV['FLUENTD_SYSTEMD_CONF'] || '../systemd'}.conf"
    #@include "#{ENV['FLUENTD_PROMETHEUS_CONF'] || '../prometheus'}.conf"
    @include ../kubernetes.conf
    @include conf.d/*.conf

    <match **>
      @type s3
      @id out_s3
      @log_level info
      s3_bucket "#{ENV['S3_BUCKET_NAME']}"
      s3_endpoint "#{ENV['S3_ENDPOINT']}"
      s3_region "#{ENV['S3_BUCKET_REGION']}"
      s3_object_key_format %{path}%Y-%m-%d-cluster-log-%{index}.%{file_extension}
      <inject>
        time_key time
        tag_key tag
        localtime false
      </inject>
      <buffer>
        @type file
        path /var/log/fluentd-buffers/s3.buffer
        timekey 180
        timekey_use_utc true
        chunk_limit_size 256m
      </buffer>
    </match>
```
6. You may change the format of the file name of how the log files are uploaded to Oject Storage at 's3_object_key_format'.

7. Fluentd will upload the logs to Object Storage in every 180 seconds. You may change the interval to your requirement at 'timekey'.

8. From OCI CLI terminal, run 'kubectl create -f configmap.yaml'.

9. Create a file called 'fluentd.yaml' with the following contents:-
```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: default
  labels:
    k8s-app: fluentd-logging
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccount: default
      serviceAccountName: default
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image:  <region_code>.ocir.io/<tenancy_namespace>/fluentd-kubernetes-daemonset:v1.8.1-debian-s3-1.0
        imagePullPolicy: Always
        env:
          - name:  AWS_ACCESS_KEY_ID
            value: "[YOUR CUSTOMER SECRET KEY ID]"
          - name:  AWS_SECRET_ACCESS_KEY
            value: "[YOUR CUSTOMER SECRET KEY]"
          - name: S3_BUCKET_NAME
            value: "[YOUR BUCKET NAME]"
          - name: S3_BUCKET_REGION
            value: "[YOUR REGION IDENTIFIER]"
          - name: S3_ENDPOINT
            value: "https://[YOUR OBJECT STORAGE NAMESPACE].compat.objectstorage.us-ashburn-1.oraclecloud.com/"
          - name: FLUENT_UID
            value: "0"
          - name: FLUENTD_CONF
            value: "override/fluent.conf"
          - name: FLUENTD_SYSTEMD_CONF
            value: "disable"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log/
        - name: u01data
          mountPath: /u01/data/docker/containers/
          readOnly: true
        - name: fluentconfig
          mountPath: /fluentd/etc/override/
      terminationGracePeriodSeconds: 30
      imagePullSecrets:
        - name: ocirsecret
      volumes:
      - name: varlog
        hostPath:
          path: /var/log/
      - name: u01data
        hostPath:
          path: /u01/data/docker/containers/
      - name: fluentconfig
        configMap:
          name: fluent-config
```

10. Update the 5 values which I have put as '[YOUR_xxxx]' to what you have copied from the above sections.

11. If the OCIR secret is not named 'ocirsecret', change the secret name in the above yaml file accordingly too.

12. From OCI CLI terminal, run 'kubectl create -f fluentd.yaml'.

13. To verify deployment, use 'kubectl get po' to check if the fluentd pods are running.

### Spring Boot Application
1. Fork the demo from Github https://github.com/eugeniapay/spring-rest-service into your own GitHub.

2. Update 'pom.xml' to add Logback dependencies 

- From Github web console, click on the file 'pom.xml' and edit it
- This XML file contains information about the project and configuration details used by Maven to build the project.
- Add the following lines to 'dependencies' node.

```
		<dependency>
      			<groupId>net.logstash.logback</groupId>
      			<artifactId>logstash-logback-encoder</artifactId>
  		</dependency>
		<dependency>
		    	<groupId>ch.qos.logback</groupId>
		    	<artifactId>logback-classic</artifactId>
		</dependency>
		<dependency>
		    	<groupId>ch.qos.logback</groupId>
		    	<artifactId>logback-core</artifactId>
		</dependency>    
```

  
3. Update 'build.gradle'

- From Github web console, click on the file 'build.gradle' and edit it
- This file is the build configuration script.
- Add the following line to 'dependencies' section.

```
		implementation 'net.logstash.logback:logstash-logback-encoder:6.3'
```			

4. Create 'logback-spring.xml' to send Spring Boot application log in JSON-Format to stdout.

- From Github web console, navigate to 'src/main/resources'.
- Create a new file called 'logback-spring.xml' and add the following lines:-

```
<configuration>
    <appender name="consoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>
    <logger name="jsonLogger" additivity="false" level="DEBUG">
        <appender-ref ref="consoleAppender"/>
    </logger>
    <root level="INFO">
        <appender-ref ref="consoleAppender"/>
    </root>
</configuration>
```

5. Update Java codes to add in logging
- From Github web console, navigate to 'src/main/java/com/example/restservice'.
- Update 'GreetingController.java' to include the logger and logging. You may change the logging content to your requirement.

```
package com.example.restservice;

import java.util.concurrent.atomic.AtomicLong;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class GreetingController {

	private static final String template = "Good Evening, %s!";
	private final AtomicLong counter = new AtomicLong();

	private static final org.slf4j.Logger LOGGER = org.slf4j.LoggerFactory.getLogger(GreetingController.class);	

	@GetMapping("/greeting")
	public Greeting greeting(@RequestParam(value = "name", defaultValue = "World") String name) {
		int cnt = (int) counter.incrementAndGet();
		
		LOGGER.info("--[/greeting] id : " + cnt + ", name: " + name + " ---");
		return new Greeting(cnt, String.format(template, name));
	}
}
```

6. Re-build
- Ensure all existing spring-rest's pods, deployments and services are deleted from OKE.
- From Github web console, click 'readme.MD' and make any changes and commit it to trigger a build in wercker automatically. 
- Ensure the image is pushed to OCIR and its pod and Load Balancer are cerated in OKE.
- From OCI CLI terminal, run 'kubectl get po' to confirm the Pods are running
- At the same console, run 'kubectl get services' to confirm the service for 'spring_rest_service' is running and identify its public IP. In case the public IP (or external IP as it shows on screen) is shown as 'pending', try again later as sometimes it takes time to secure the public IP. If the status is still pending after 5 minutes, please check if you have any other pods not running smoothly. You may need to delete those pods first and try again as I encounter similar problem with trial OCI Gen 2 accounts
- If you are using my completed project from https://github.com/eugeniapay/spring-rest-service-logback, please note that all names are added with '-log'. For example, 'spring-rest-log-deployment', 'spring-rest-service-log', etc.
- From a browser, acccess http://<public_ip>:8080/greeting?name=Mary. You should see something like below:- (Note that the ID may change as well as its greetings because it is constantly used for various demos)

```
'{"id":1,"content":"Good Morning, Mary!"}'
```
- From OCI CLI terminal, run 'kubectl logs -f [POD NAME OF THE SPRING REST APP] and you will see the logs are in JSON format together with the logging from /greeting.
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.0.RELEASE)

{"@timestamp":"2020-02-24T03:22:17.155Z","@version":"1","message":"Starting RestServiceApplication on spring-rest-log-deployment-7ffd974d4-zt8sl with PID 1 (/app.jar started by spring in /)","logger_name":"com.example.restservice.RestServiceApplication","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:17.167Z","@version":"1","message":"No active profile set, falling back to default profiles: default","logger_name":"com.example.restservice.RestServiceApplication","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:18.898Z","@version":"1","message":"Tomcat initialized with port(s): 8080 (http)","logger_name":"org.springframework.boot.web.embedded.tomcat.TomcatWebServer","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:18.914Z","@version":"1","message":"Initializing ProtocolHandler [\"http-nio-8080\"]","logger_name":"org.apache.coyote.http11.Http11NioProtocol","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:18.915Z","@version":"1","message":"Starting service [Tomcat]","logger_name":"org.apache.catalina.core.StandardService","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:18.915Z","@version":"1","message":"Starting Servlet engine: [Apache Tomcat/9.0.27]","logger_name":"org.apache.catalina.core.StandardEngine","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:19.015Z","@version":"1","message":"Initializing Spring embedded WebApplicationContext","logger_name":"org.apache.catalina.core.ContainerBase.[Tomcat].[localhost].[/]","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:19.016Z","@version":"1","message":"Root WebApplicationContext: initialization completed in 1775 ms","logger_name":"org.springframework.web.context.ContextLoader","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:19.477Z","@version":"1","message":"HV000001: Hibernate Validator 6.0.17.Final","logger_name":"org.hibernate.validator.internal.util.Version","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:19.904Z","@version":"1","message":"Initializing ExecutorService 'applicationTaskExecutor'","logger_name":"org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:20.299Z","@version":"1","message":"Exposing 14 endpoint(s) beneath base path '/actuator'","logger_name":"org.springframework.boot.actuate.endpoint.web.EndpointLinksResolver","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:20.347Z","@version":"1","message":"Starting ProtocolHandler [\"http-nio-8080\"]","logger_name":"org.apache.coyote.http11.Http11NioProtocol","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:20.379Z","@version":"1","message":"Tomcat started on port(s): 8080 (http) with context path ''","logger_name":"org.springframework.boot.web.embedded.tomcat.TomcatWebServer","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:20.382Z","@version":"1","message":"Started RestServiceApplication in 4.245 seconds (JVM running for 4.943)","logger_name":"com.example.restservice.RestServiceApplication","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:47.993Z","@version":"1","message":"Initializing Spring DispatcherServlet 'dispatcherServlet'","logger_name":"org.apache.catalina.core.ContainerBase.[Tomcat].[localhost].[/]","thread_name":"http-nio-8080-exec-1","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:47.994Z","@version":"1","message":"Initializing Servlet 'dispatcherServlet'","logger_name":"org.springframework.web.servlet.DispatcherServlet","thread_name":"http-nio-8080-exec-1","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:48.003Z","@version":"1","message":"Completed initialization in 9 ms","logger_name":"org.springframework.web.servlet.DispatcherServlet","thread_name":"http-nio-8080-exec-1","level":"INFO","level_value":20000}
{"@timestamp":"2020-02-24T03:22:48.062Z","@version":"1","message":"--[/greeting] id : 1, name: Mary ---","logger_name":"com.example.restservice.GreetingController","thread_name":"http-nio-8080-exec-1","level":"INFO","level_value":20000}
```

7. View Logs at Oracle Object Storage
- At OCI Web Console, navigate to the bucket which you have created and you will see logs are uploaded to the bucket. 
- Check the logs and you should see the above logs are being sent to Oracle Object Storage by Fluentd.
```
2020-02-24T03:22:48+00:00	kubernetes.var.log.containers.spring-rest-log-deployment-7ffd974d4-zt8sl_default_spring-rest-log-b3ab8df4d4d842052f767e5734fdd33e16226c28bf009bfa123e6f3ab90a1dfc.log	{"log":"{\"@timestamp\":\"2020-02-24T03:22:48.062Z\",\"@version\":\"1\",\"message\":\"--[/greeting] id : 1, name: Mary ---\",\"logger_name\":\"com.example.restservice.GreetingController\",\"thread_name\":\"http-nio-8080-exec-1\",\"level\":\"INFO\",\"level_value\":20000}\n","stream":"stdout","docker":{"container_id":"b3ab8df4d4d842052f767e5734fdd33e16226c28bf009bfa123e6f3ab90a1dfc"},"kubernetes":{"container_name":"spring-rest-log","namespace_name":"default","pod_name":"spring-rest-log-deployment-7ffd974d4-zt8sl","container_image":"iad.ocir.io/apacaseanset01/epdemo/spring_rest_log:latest","container_image_id":"docker-pullable://iad.ocir.io/apacaseanset01/epdemo/spring_rest_log@sha256:0cb89d75891e9cf830a697764b008b0efd363cf9994e42571461e87c0c591f5d","pod_id":"d9336135-56b4-11ea-a31e-0a580aedb166","host":"10.0.10.2","labels":{"app":"spring-rest-log","pod-template-hash":"7ffd974d4"},"master_url":"https://10.96.0.1:443/api","namespace_id":"dd1a1f9a-43f0-11ea-af41-0a580aed3e63"},"tag":"kubernetes.var.log.containers.spring-rest-log-deployment-7ffd974d4-zt8sl_default_spring-rest-log-b3ab8df4d4d842052f767e5734fdd33e16226c28bf009bfa123e6f3ab90a1dfc.log","time":1582514568.062426}

```
