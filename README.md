# ArchitectureGuide
studying of docker, containerd, kubernetes, k3s, k8s, ELK stack etc


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
* ### docker-compose
  * ### What is docker-compose
    * ![img4.png](images%2Fimg4.png)   
      Lets say that you want to create a service.   
      You would want to create a database, a backend server, and a frontend server.   
      Well, docker compose got you covered, you can deploy all of these servers with a single command, `docker compose up`.   
      <br>
      It is just like a composer in a orchestra, handling multiple pods just in one go.
  * ### `docker-compose.yaml`
    * ### Basics
      ```yaml
      version: "3"
      services:
        frontend:
          build: frontend_file/.
          ports:
          - "8081:8080"
        backend:
          build: backend_file/.
          ports:
          - "8082:3000"
        database:
          image: "mysql:8.2.0"
      ```
      This `docker-compose.yaml` means,   
      1. Create 3 services named `frontend`, `backend`, `database`.   
      2. The `Dockerfile` located for each service is in `frontend_file/.`, `backend_file/.`.   
      3. For the database, use the image `mysql:8.2.0` from docker hub.   
      4. Connect the `frontend` pods port 8080 to 8081 of the host.
      5. Connect the `backend` pods port 3000 to 8082 of the host.
    * ### Volumes
      In docker, you can run `docker run -v <host-path>:<container-path> <image-name>` command in order use a path of the host as volumes to mount on your image.   
      <br>
      This is handy when you need to create multiple pods and shared volumes between services.
      ```yaml
      version: "3"
      services:
        database:
        image: mysql:8.2.0
        volumes:
         - ./host-file-path:/var/lib/mysql
      ```
      The volumes is an array of `<host_path>:<container_path>` that you want to mount.   
      After you mount a volume, changes either in the host or the container will affect each other as it is a "mounted" volume.   
      <br>
      try the `docker exec -it <mycontainer> bash` command, to double check.   
      Some IDEs like jetbrains allow you to connect by UI.    
      ![img5.png](images%2Fimg5.png)
    * ### Environment variables
      ![img6.png](images%2Fimg6.png)   
      if you have ran the `image: mysql:8.2.0` image above, you will see this error.
      This is because the environment variable for the mysql connection is not set.   
      ```yaml
      version: "3"
      services:
        my-service-name:
          image: my-image-name
        environment:
          - MYSQL_ROOT_PASSWORD=value1
          - MYSQL_ALLOW_EMPTY_PASSWORD=value2
          - MYSQL_RANDOM_ROOT_PASSWORD=value3
      ```
      You can also set environment variables in docker-compose.   
      We did learn that `ENV` in Dockerfile also changes the environment variable.   
      It would be a matter of taste where to put it, so just pick one, and do not cross use it.
    * ### depends_on
      This keyword is for a service to start only after the service that it depends on.   
      The frontend should never start before the backend.
      ```yaml
      version: '3'

      services:
        backend:
          image: backend-image-location

        frontend:
          image: frontend-image-location
          depends_on:
          - backend
      ```
    * ### Replicas
      Just note that this feature exists, but do not use this unless you are using docker swarm.
      Most companies do not use swarm over k8s.
      ```yaml
      version: "3"
        services:
          database:
            image: "mysql:8.2.0"
            ports:
            - "8080:3000"
            deploy:
              replicas: 3
      ```
      Also, you cannot bind 3 replicas ports with localhost port of 8080.   
      So you should create 3 services separately or remove the host port.   
      <br>
      If you are thinking of creating a load balancer within `docker-compose` and redistribute traffic, I strongly advise you to just use
      kubernetes service and deployments.   
      <br>
      It is possible with nginx, but not recommended. kubernetes should handle replicas and traffic, not docker.   

## Kubernetes
* ### Why do we need microservices? [Mastering chaos - Netflix](https://www.youtube.com/watch?v=CZ3wIuvmHeM&t=1218s)   
  ![img7.png](images%2Fimg7.png)
  From the year 2000, Netflix had a monolith architecture that scales horizontally.   
  They used two databases "STORE" and "BILLING" which had no replicas whatsoever.   
  As time goes on, their main server has become this massive chunk of code which was over their heads, dies constantly, takes all day to boot and debug.
  * #### Massive chunk of code
  * #### Massive dependencies
  * #### Massive cost on boot and debugging
  * #### High usage of threads and processes
  * #### New features can kill the whole service
  * #### A bad query can kill the whole service
  * #### Massive cost updating databases and columns
  ![img8.png](images%2Fimg8.png)
  That is where the microservice architecture comes in.   
  This architecture is where a service is divided into multiple parts.   
  <br>
  Netflix
  * user-api
  * product-api
  * platform-api
  * Persistence (Databases)
  * ...
      
  <br>

  This did solve the main problems of a monolith architecture, but it has its downfalls.   
  * #### Cascading failures
    When a feature accesses multiple api, a single point that fails will result in another point to fail and continues until it kills the whole system.   
    ![img9.gif](images%2Fimg9.gif)   
    In Netflix, they have solved it by a static response to be returned in case of a failure.    
    This blocks and isolates the failed endpoint. and continues the service with failure in mind.
  * #### Database Persistence
    If a service wants to change data to 3 different regions of databases, but cannot access to some of them, should the process fail?
    Should the process write to one, and apply the changes later?   
    <br>
    Now netflix used a nosql database called "Cassandra" which manages multiple database nodes and keeps persistence with each other nodes.   
    <br>
    This might be a overkill for most companies, but it's good to know a service that handles databases the "microservice" way.
    ![img10.jpeg](images%2Fimg10.jpeg)
  * #### Stateful Service
    If a node keeps a "state", it means that it stores data within that node that is relevant to the service.
    The "user" must keep contacting this "service node" because it keeps it's "state" within.   
    If a service is "stateful" the loss is a "cost" that results in a failure of a service.   
    If a service is "stateless" the loss of a node has no impact on the service as the user can contact another node.
* ## Kubernetes components
  Kubernetes is a microservice management tool constructed with multiple components.   
  <br>
  * Master nodes
    * API Server
      * Kubernetes dashboard
      * API
      * Kubectl
    * Control Manager
    * Scheduler
    * etcd
    * Deployment
      * Service
  * Worker nodes
    * Pods
      * Containers

  I will be giving examples on how to set up a kubernetes server without a cloud provider or tools.   
  If you want to manually configure kubernetes manually, you can use [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/),
  [kpops](https://kops.sigs.k8s.io/) or [kubespray](https://kubespray.io/#/).   
  If you want to configure kubernetes in AWS, you can use [EKS](https://aws.amazon.com/eks/?nc1=h_ls) or [EKSCTL](https://github.com/eksctl-io/eksctl).   
  In this guide, I will use kubeadm studied from the [official documentation](https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/).
  * ### Installation
    * ### Open ports
      In order for the Master node and worker nodes to function, you need these [specific network rules](https://v1-28.docs.kubernetes.io/docs/reference/networking/ports-and-protocols/).
      ![img12.png](images%2Fimg12.png)   
      <br>
      Now if you are using AWS, you can configure security groups for inbound and outbound rules.
      ![img13.png](images%2Fimg13.png)   
      <br>
      You can use netcat (nc) in order to check if the ports are open and connectable between nodes.   
      For example, if you want to check if the port `10250` is open in your worker node,   
      run `nc -l 10250` in your worker,   
      run `nc -zv <worker-node-ec2-ip> 10250` in your master.  
      <br>
      your master netcat should return something like this.   
      ![img14.png](images%2Fimg14.png)   
      
    * ### Container runtime
      Now the kubernetes [documents](https://v1-28.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/)
      notes that you can run with a multiple of options like containerd, docker engine, CRI-O. I will just install docker with the simple `sudo apt install` command.   
      Kubernetes did drop docker support for dockershim from its project and uses containerd, but docker uses containerd under the hood, so we shouldn't be worried about it.   
      [Install docker on ubuntu](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
      ```shell
      # Add Docker's official GPG key:
      sudo apt-get update
      sudo apt-get install ca-certificates curl gnupg
      sudo install -m 0755 -d /etc/apt/keyrings
      curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      sudo chmod a+r /etc/apt/keyrings/docker.gpg
      
      # Add the repository to Apt sources:
      echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
      sudo apt-get update
      sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
      ```
    * ### Installing kubeadm, kubelet and kubectl
      This installation is using the `apt` package install method, so if you are using redhat, or any other distributions, check [instructions](https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl).
      ```shell
      sudo apt-get update
      # apt-transport-https may be a dummy package; if so, you can skip that package
      sudo apt-get install -y apt-transport-https ca-certificates curl gpg
      curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
      echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
      sudo apt-get update
      sudo apt-get install -y kubelet kubeadm kubectl
      sudo apt-mark hold kubelet kubeadm kubectl
      ```
    * ### Installing master node
      `sudo kubeadm init --pod-network-cidr={your cidr ip}/{your cidr mask}`    
      <br>
      `--pod-network-cidr=172.31.0.0/16` this command is for configuring the cidr block for pods.   
      `--control-plane-endpoint` specifies the endpoint (load balancer or IP) for the control plane. This is used in HA setups where the control plane is distributed across multiple nodes.   
      `--upload-certs` is used to upload certificates to the Kubernetes configuration directory. This is essential for joining additional control plane nodes to the cluster.   
      <br>
      If have successfully installed the master node, this should pop up.   
      <br>
      ![img15.png](images%2Fimg15.png)
      You must configure the `./kube/config` file afterward.
      ```shell
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/kubelet.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      # restart kubelet in order to let the configuration apply
      sudo systemctl restart kubelet.service
      ```
    * ### Installing worker node
      If you have successfully installed the master node, you will get a response like this.   
      `kubeadm join 172.31.14.79:6443 --token 8xwmlv.gs5787m8y16ty0zx \
      --discovery-token-ca-cert-hash sha256:0ddf616b72ad2065e6312b17f03c0a77e090547a5e8190296141a5833a5d6a6c`   
      <br>
      Run the command on the worker node. and you should get a response like this.   
      <br>
      ![img16.png](images%2Fimg16.png)   
      <br>
      If you return to your master node, and check `kubectl get nodes`,
      ![img17.png](images%2Fimg17.png)   
      Congrats!
    * ### errors
    * #### unknown service runtime.v1.RuntimeService
      If you have any errors, run the `kubeadm reset` to reset changes made by kubeadm.   
      Errors I have encountered   
      https://github.com/kubernetes-sigs/cri-tools/issues/1089
      ```
      validate service connection: validate CRI v1 runtime API for endpoint "unix:///run/containerd/containerd.sock": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService
      ```
      This error happens because kubeadm cannot communicate with containerd via a sock communication tool `crictl`.
      ```
      sudo crictl -r unix:///run/containerd/containerd.sock ps
      # expected result
      CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
      ```
      This command will change the `/etc/containerd/config.toml` so that it uses the default configuration.   
      Change the `disabled_plugins = ["ctl"]` into `disabled_plugins = []`
      ```
      sudo su
      containerd config default | tee /etc/containerd/config.toml
      sudo vim /etc/containerd/config.toml
      systemctl restart containerd
      ```
      
    * #### Get "https://{server_ip_address}:6443/api?timeout=32s": dial tcp {server_ip_address}:6443: connect: connection refused
      If the {server_ip_address} is localhost, check the command `kubectl config view` command.   
      This command would show that `~/.kube/config` is not set.   
      https://stackoverflow.com/questions/76841889/kubectl-error-memcache-go265-couldn-t-get-current-server-api-group-list-get
      ```shell
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/kubelet.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      ```
      
      If your configurations are set, it means that you did not reboot kubernetes to set changed configurations.
      ```shell
      # restart kubelet in order to let the configuration apply
      sudo systemctl restart kubelet.service
      ```
    * #### Unable to connect to the server: tls: failed to verify certificate: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")
      This error happens because the configurations that is set on `$HOME/.kube/config` is having problems with certification.   
      ```yaml
      user:
         client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
         client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
      ```
      The .pem files might not exist, if this happens on the control plane you can copy the admin.conf as the configurations.   
      ```shell
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      ```
  * ### Service
    A service is network control configurations for pods.   
    There are a various types you can assign as your service.
    * ClusterIp 
    * NodePort
    * LoadBalancer
    * ExternalName
    <br>   
    <br>

    This is a basic configuration of a `service.yaml`   
    Lets go through the basics and see what each component do.
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      type: NodePort
      selector:
        app: my-app
    ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
    ```
    ### `spec.type`
    * Note that the `kind` must be as `Service` and you can specify which service it is by `spec.type`.   
      If you do not specify, it will be of type `ClusterIp` as default.   
  
    ### `spec.selector`
    * The `spec.selector` you see above is `app:my-app`.   
      This is how services know which pod is assigned to this service.   
      You can assign `selectors` to a specific pod by `deployment.yaml`.
    * ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-deployment
      spec:
        replicas: 3
        selector:
          matchLabels:
            app: my-app
        template:
          metadata:
            labels:
              app: my-app
          spec:
            containers:
              - name: my-container
                image: my-image
      ```
      This is the deployments example.   
      You can see two instances of `app:my-app`.   
      <br>
      `spec.selector.matchLables` = `app:my-app`   
      This means that the Deployment will identify the pod by `app:my-app` label.   
      <br>
      `spec.template.metadata.labels` = `app:my-app`   
      This menas that the template for each pods will have the label `app:my-app` when created.   
      <br>
      So you would need 3 instances of `app:my-app` needed.   
      Two in deployment.yaml and one in service.yaml.   
      Note that all labels need to match if multiple is given.   
    ### `ports.port` && `ports.targetPort`
    * `ports.port` is the port that the service opens to be accessed.
    * `ports.targetPort` is the port that the service will access to the pods.
    * ![img18.png](images%2Fimg18.png)
    * You can get the internal(cluster-ip) and external-ip of the service and its open ports by the command   
      `kubectl get service <my-service-name>`   
      ![img19.png](images%2Fimg19.png)


## Local Servers
* ### Why would you want to escape AWS?
  The tech industry is currently so tangles with cloud providers.   
  In a way that we cannot think or come up with a idea that does not use or rely on such services.    
  But in some cases, it is better to go locally and manage your own servers in order to cut cost.
  [$400,000,000 Saved - NO MORE AWS](https://www.youtube.com/watch?v=XAbX62m4fhI&t=1173s)   
  I am not in the position or experience to say such claims, but I do understand such points taken and give some cases.   
  * #### Using extreme computing hardware. (ex: AI).
  * #### If your service relies on computing power itself as a source of income.
  * #### If you are a student or a hobbyist sick of paying per month to AWS. (Me)
* ### Why did I chose to migrate some of my projects out of AWS?
  I have created a k8s cluster on aws trying to learn and deploy a microservice on my own.
  ![img20.png](images%2Fimg20.png)   
  But currently, the k8s requires a 2GB and a 2vCPU machine.   
  And k8s requires you to have 1 control plane, and 1 worker in order to get the full ordeal.   
  This would require me to get at least 2x t2.medium.
  ![img21.png](images%2Fimg21.png)   
  The monthly cost ramps up to 170$ per month if you consider all the additional cost for networking and rdb.   
  Which in fact, can be installed in a computer that you own in your house.   
  <br>
  This is my bills by the way. XD   
  ![img22.png](images%2Fimg22.png)   
* ### The cost return factor with custom bought computers.   
  Now, if you do plan to go local, you should be quite careful to chose the right hardware.   
  It is up to you because there are so many vendors, and so many choices to be made in such occasions.   
  <br>
  But to be simple, I will compare the t2.medium with raspberry pi 5.   
  <br>
  A [t2.medium](https://stackoverflow.com/questions/57652339/what-is-vcpu-in-aws) has a max of 3.3 Ghz and 4GB of memory.   
  A [raspbarry pi 5](https://www.amazon.com/Raspberry-4GB-Board-Heatsinks-4pcs/dp/B0CQ2WGZFG/ref=sr_1_3?crid=QKWTOVV89UF8&dib=eyJ2IjoiMSJ9.saE66Pe4RTBORKkp4RmGt3EVnb6tNZPsQeVds8HlABfxfd0Dnu-eNWay9ndfmcUoUJHAk_tAJ5g7JJ98nq9-E2mQwFWj8Lu0f3nkH1UWLgNsseL2i17Kz-c4a2rFDxUgNizoES7ePqjgLxMsQaz4iEOSP8UENMlt7-43k5CSfMBkalrgthzee8S_1nlaee0wzZ67FDpwpCQsDahg_91TQUxnCeeIVwO-8HRWucf1-74.czBrQXCn5eBHQtwvP3bHzZWJnwBod-23e1faxiUTfc8&dib_tag=se&keywords=raspberry+pi+5+4gb&qid=1713170981&sprefix=raspberry+pi+5+4g%2Caps%2C246&sr=8-3) has 2.4 Ghz on default and 4GB of memory.   
  <br>
  (at the time of this writing)   
  A t2.medium would cost you 42.5$ per month.   
  A raspberry pi would cost you 73$ for purchase.   
  <br>
  If you run your local server for more than 2 months, you can get your ROI for the initial cost.   
  The same can be said for GPU intensive servers, NAS servers.   
  It rounds up to **2 months ~ 1 year** to get back your initial investment in most cases.   
* ### It is not all roses and flowers
  The sole reason that AWS has become dominate in the first place, is because it was such a hassle in order to manage a local server.   
  The ISP that you have may not be sufficient to handle such traffic as in AWS.   
  If you have such servers managed by you or your company, it can also be a huge risk for downtime.   
  Like the example of the [kakao datacenter on fire](https://www.youtube.com/watch?v=0RmDw3t4JPM).   
  Business critical information such as money transfer can be lost in such occasions.   
  You should think local servers as a **choice that can be made, not as a belief.**
* ### Installing k8s in local servers
  
