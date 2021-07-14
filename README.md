# API Security Hands-on Session

## Local Lab Setup 

### Pre-requisites :information_source:
In case you want to deploy the below described lab on your local machine, then you need to make sure that the below tools/packages are installed:
- **Docker Engine**. The lab will be deployed as a Docker container, so a Docker engine is a must have. :smiley:
The easiest way is to install Docker Desktop :
  - [Install Docker Desktop on Mac](https://docs.docker.com/docker-for-mac/install/)
  - [Install Docker Desktop on Windows](https://docs.docker.com/docker-for-windows/install/)
- **Git** :atom:
  - you can  install [Git Desktop on Mac/Windows](https://desktop.github.com/)
  - on mac you can install the Git package via Brew: 
    `
    brew install git
    `
- **curl**: typically already installed on your machine. You can check this with `curl --version`

### Step-by-Step Instructions
* Clone the API-Security GitHub repo and from within your terminal head into the api-security directory
`git clone https://github.com/StevenDuckaert/api-security.git`

#### Kong Gateway & Cassandra Deployment
* Pull the latest Kong image we will be using for the API Gateway
`docker pull kong`
* To make image management easier, it is always recommended to tag the image. In the below command we tag the Kong image with *kong-gw*
`docker image list | grep kong`
`docker tag <<id-from_the_above_output>> kong-gw`
* We'll create a Docker network to ensure all containers are provisioned into the same network
`docker network create noname_default`
* Start a Cassandra container in the just created Docker network
`docker run -d --name kong-db --network=noname_default -p 9042:9042 cassandra:3`
* Once the Cassandra container is up - this typically takes less then 10 seconds, we will need to populate the DB
    ```
    docker run --rm \
    --network=noname_default \
    -e "KONG_DATABASE=cassandra" \
    -e "KONG_PG_HOST=kong-db" \
    -e "KONG_PG_USER=kong" \
    -e "KONG_PG_PASSWORD=kong" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-db" \
    kong-gw kong migrations bootstrap
    ```
* You can now start the Kong Gateway
   ```
   docker run -d --name kong \
        --network=noname_default \
        -e "KONG_DATABASE=cassandra" \
        -e "KONG_PG_HOST=kong-db" \
        -e "KONG_PG_USER=kong" \
        -e "KONG_PG_PASSWORD=kong" \
        -e "KONG_CASSANDRA_CONTACT_POINTS=kong-db" \
        -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
        -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
        -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
        -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
        -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
        -e "KONG_PlUGINS=bundles" \
        -p 127.0.0.1:8000:8000 \
        -p 127.0.0.1:8443:8443 \
        -p 127.0.0.1:8001:8001 \
        -p 127.0.0.1:8444:8444 \
        kong-gw
   ```
* When the container is up, you can verify the Kong GW via:
`curl -i -X GET --url http://127.0.0.1:8001/services`
The output should be similar to:
    ```
    HTTP/1.1 200 OK
    Date: Wed, 14 Jul 2021 09:17:02 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Access-Control-Allow-Origin: *
    Content-Length: 23
    X-Kong-Admin-Latency: 224
    Server: kong/2.5.0

    {"next":null,"data":[]}%
    ```
#### Vulnerable API Deployment
There are several projects whereby you can quickly explore API vulnerabilities. The one I really like and which will be used in this lab is the [VAmPI](https://github.com/erev0s/VAmPI) project. This is a vulnerable API made with Flask and it includes vulnerabilities from the OWASP top 10 vulnerabilities for APIs. It includes a switch on/off to allow the API to be vulnerable or not while testing. The image we will pull has the ON switch configured. You can find a bit more details about the vulnerabilities at [erev0s.com](https://erev0s.com/blog/VAPI-vulnerable-api-security-testing/).

Deploy the vulnerable API environment
```
docker run -d --name vulnerable_vapi --network noname_default -p 127.0.0.1:5000:5000 stevenduckaert/api-security:vulnerable
```


### Automated Deployment
