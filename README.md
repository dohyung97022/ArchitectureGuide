# ArchitectureGuide
studying of docker, containerd, kubernetes, k3s, ELK stack etc


## Docker
* ### Dockerfile
  * ### What is a dockerfile?
    * A dockerfile is a set of commands to configure and create a image.   
      It is possible to upload and read images only by the docker hub and automate prerequisites by shell, but it is ideal and faster to create and load images with 
      your servers specific needs.
    * #### `docker build -t my-image-name .`   
      This command will create a image called `my-image-name` reading a `Dockerfile` in the path `.`.
  * ### What are the components of a dockerfile?
    * ### `FROM`
      This keyword is used to import the base image of the dockerfile.
      ```dockerfile
      FROM python:3.10.13-alpine3.18
      ```
      The most common case of `FROM` is by the hubs repository:tagName.   
      Tag names are created by the providers of the image and mostly contains of OS information and versions.   
      ex : python3.10 installed in alpine3.18 OS.
    * ### `WORKDIR`
      This keyword specifies starting directory within the image.   
      If the directory does not exists, it will create the specified directory.   
      You can always change the directory multiple times in order to specify different locations for executions like `COPY` or `RUN`.
    * ### `COPY`, `ENV`
      ```dockerfile
      COPY /location-to-copy /in-image-location
      ```   
      Copies the file in the scope of the dockerfile project to the images location.   
      ```dockerfile
      ENV LOG_PATH=/etc/project/log
      ```
      Creates a environment variable in the image.
    * ### `RUN` vs `CMD`
      ```dockerfile
      FROM python:3.10.13-alpine3.18
      WORKDIR /etc/project
      COPY main.py
      
      RUN mkdir external-files \
      && apk --no-cache update

      RUN apk --no-cache add curl
      
      CMD curl www.google.com \
      python main.py
      ```   
      Both `RUN` and `CMD` receives shell commands but has a fundamental difference.   
      <br/>
      The `RUN` command is for creating the dockerfile.   
      And the `CMD` command is for commands to run at execution.   
      <br/>
      If you run `docker build .` with the dockerfile above, you will get this.   
      ![img3.png](images%2Fimg3.png)   
      Note that each separation of `RUN` is separated as image layers.   
      You should divide them to reasonable layers in order to debug efficiently.   
      <br/>
      If you run `docker run` to the created image, you will get a http response of google and run your python script.    
    * ### Expose vs `-p`
      ```dockerfile
      FROM tomcat:10.1.17-jdk21-temurin-jammy
      EXPOSE 8080/tcp
      ```
      This dockerfile creates a tomcat container and opens the default port used by tomcat, 8080.   
      ```shell
      docker build -t test-container .
      docker run test-container
      ```
      If you run the `docker ps` command, you can see `8080/tcp` in the PORTS section.   
      ![img.png](images/img.png)    
      <br />
      But if you create a dockerfile without `EXPOSE`
      ```dockerfile
      FROM tomcat:10.1.17-jdk21-temurin-jammy
      ```
      and run docker with using the `-p` command
      ```shell
      docker build -t test-container .
      docker run -p 8888:8080/tcp test-container
      ```
      ![img.png](images/img2.png)
      `docker ps` command, returns `0.0.0.0:8888->8080/tcp`.   
      ### What does this mean?
      When a docker port needs to be accessed with other docker containers, `EXPOSE` is sufficient.   
      `EXPOSE` does not map the opened port with the hosts port.   
      You should use it if the application only needs to be accessed by other containers within the scope.   
      <br/>
      In the other hand `docker run -p <hostPort>:<containerPort>/tcp` opens and maps the port to be accessed by the host.   
      Therefor the `docker ps` shows `0.0.0.0:8888 (the hosts port)->8080/tcp (mapped to the containers port)`.      
      <br/>
      It would be best if we use this accordingly to our applications use case.   