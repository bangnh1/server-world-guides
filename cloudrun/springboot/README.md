# How to setup cloud run with springboot?

## Context

- openjdk-8u302
- gcloud ver.352
- mvn 3.8.1

## Building the Spring Application

1. Create skeleton project

- [Create skeleton project](https://start.spring.io/)

```
Maven project
Java
Spring Boot version: 2.5.3
Group: com.example
Artifact: demo
Name: demo
Description: Demo project for Spring Boot
Package name: dev.truaro.blog.gcpcloudrunback
Dependencies: Spring Web
```

2. Modify pom.xml

```
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
```

3. Add api hello and hello-world

```
$ vi src/main/java/com/example/demo/DemoApplication.java

@RestController
class GreetingController {

	@RequestMapping("/hello/{name}")
	String hello(@PathVariable String name) {
		return "Hello, " + name + "!";
	}
}

@RestController
class HelloWorldController {

	@RequestMapping("/")
	public String helloWorld() {
		return "Hello World";
	}

}

```

4. Configure server post and base path

```
$ vi src/main/resources/application.properties
server.port=${PORT:8080}
server.servlet.context-path=/api
```

5. Build docker image

- [Create dockerfile](./Dockerfile)

```
$ gcloud init
$ docker build -t asia.gcr.io/${PROJECT_ID}/springboot:latest
$ gcloud auth configure-docker
$ gcloud services enable containerregistry.googleapis.com
$ docker push asia.gcr.io/${PROJECT_ID}/springboot:latest
```

## Deploy to Cloud Run

1. Create Cloud Run service description file

- [Create Cloud Run service description file](./gcp-cloudrun-springboot.yaml)

2. Create service account

```
$ gcloud iam service-accounts create gcp-cloudrun-springboot \
    --display-name="gcp-cloudrun-springboot"
```

3. Deploy to Cloud Run

```
$ gcloud services enable run.googleapis.com
$ gcloud beta run services replace gcp-cloudrun-springboot.yaml \
  --platform=managed \
  --region=asia-northeast1
```

4. Allow public access to invoke your service

```
$ gcloud run services add-iam-policy-binding gcp-cloudrun-springboot \
  --platform=managed \
  --region=asia-northeast1 \
  --member="allUsers" \
  --role="roles/run.invoker"
```

## Create custom domain

1. Install firebase cli

```
$ npm install -g firebase-tools
```

2. Login firebase

```
$ firebase login
```

- [Create firebase.json](./firebase.json)

3. Deploy

```
$ firebase deploy --project <<PROJECT_NAME>>
```

- [Create custom domain](https://firebase.google.com/docs/hosting/custom-domain)
