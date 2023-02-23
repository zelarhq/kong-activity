# Kong Activities

## Prerequisites
- Ubuntu
- Docker&Docker compose
- HTTPie

## Getting the app up and fixing the issues to be able to signin
Run the containers
  ```
  docker-compose up -d
  ```
Type `docker logs <container-id>` to see the errors
## Fixes
1. Need to change `KONG_DATABASE: off` to `KONG_DATABASE: postgres` as we want to use the postgres db
2. Add the following environment variable as this is required to enable basic auth for kong admin
    ```
    KONG_ADMIN_GUI_SESSION_CONF: '{"cookie_name":"kookie","secret":"changeme","cookie_secure":false}' 
    ``` 
3. Change `KONG_PG_HOST: db` from `KONG_PG_HOST: postgres` as the postgres docker container is named `db`
  > Note: Also changed the kong image version from `2.8.0.0-alpine` to `3.1.1.3` as instructed

## Activity 1

1. In Kong Manager, create an Upstream with two targets (use the name `httpbin-upstream` for the Upstream) that have the following hosts; 
    - httpbin.org:80 
    - localhost:80 

    ```
    http post http://localhost:8001/upstreams name=httpbin-upstream Kong-Admin-Token:password
    http post http://localhost:8001/upstreams/httpbin-upstream/targets target=httpbin.org:80 Kong-Admin-Token:password
    http post http://localhost:8001/upstreams/httpbin-upstream/targets target=localhost:80 Kong-Admin-Token:password
    ```

2. Create a Service pointing to the Upstream (use `httpbin-upstream` for the Host in the Service) and `/anything` for the path 
    ```
    http post http://localhost:8001/services name=httpbin-service host=httpbin-upstream path=/anything Kong-Admin-Token:password
    ```

3. Create a Route pointing to the Service with `/echo` as the path
    ```
    http -f post http://localhost:8001/services/httpbin-service/routes name=httpbin-route paths=/echo Kong-Admin-Token:password

    ```

4. Using curl, send requests to the API hosted in Kong which should echo back the headers/body etc.
    ``` 
    curl http://localhost:8000/echo
    ``` 

5. Using the Admin API, mark one of the Targets unhealthy 
    - First enable health check in kong admin manager, set:
      `Healthchecks.Active.Unhealthy.Timeouts = 1` and  `Healthchecks.Active.Unhealthy.Interval = 5`
    - Mark the target unhealthy
      ```
      http put http://localhost:8001/upstreams/httpbin-upstream/targets/8fe13fd5-d853-446e-b536-de91481aff18/unhealthy Kong-Admin-Token:password
      ```

6. Send more requests to the API and note which target in now used
    ``` 
    curl http://localhost:8000/echo
    ``` 

    > The other target which is still healthy is being used

7. Using the Admin API, mark the other Target unhealthy
    ```
    http put http://localhost:8001/upstreams/httpbin-upstream/targets/5f0b83ab-9c9c-4ad3-8a3e-a5c91bc8a3cd/unhealthy Kong-Admin-Token:password
    ```

8. Send more requests to the API, and observe what happens. 
    > {"message":"failure to get a peer from the ring-balancer"}


9. Using the Admin API, mark the original Target healthy 

    ```
    http put http://localhost:8001/upstreams/httpbin-upstream/targets/5f0b83ab-9c9c-4ad3-8a3e-a5c91bc8a3cd/healthy Kong-Admin-Token:password
    ```


## Activity 2

1. Add key authentication to the `Echo` service.
    ```
    http post http://localhost:8001/services/httpbin-service/plugins name=key-auth Kong-Admin-Token:password
    ```

2. Create two consumers and add a different rate limit for each consumer
    ```
    http post http://localhost:8001/consumers username=consumer1 Kong-Admin-Token:password
    http post http://localhost:8001/consumers username=consumer2 Kong-Admin-Token:password
    ```
    ```
    http post http://localhost:8001/consumers/consumer1/key-auth key=consumer1key Kong-Admin-Token:password
    http post http://localhost:8001/consumers/consumer2/key-auth key=consumer2key Kong-Admin-Token:password
    ```
    ```
    http -f post http://localhost:8001/consumers/consumer1/plugins name=rate-limiting config.minute=5 Kong-Admin-Token:password
    http -f post http://localhost:8001/consumers/consumer2/plugins name=rate-limiting config.minute=10 Kong-Admin-Token:password
    ```
    - Make requests to test the rate limits
    > http get http://localhost:8000/echo apikey:consumer1key 

    > http get http://localhost:8000/echo apikey:consumer2key

    - After sending more than 5 requests, encountered the rate limit error for the consumer1

      ```
      http get http://localhost:8000/echo apikey:consumer1key
      HTTP/1.1 429 Too Many Requests
      Connection: keep-alive
      Content-Length: 41
      Content-Type: application/json; charset=utf-8
      Date: Tue, 21 Feb 2023 07:18:17 GMT
      RateLimit-Limit: 5
      RateLimit-Remaining: 0
      RateLimit-Reset: 43
      Retry-After: 43
      Server: kong/2.8.0.0-enterprise-edition
      X-Kong-Response-Latency: 1
      X-RateLimit-Limit-Minute: 5
      X-RateLimit-Remaining-Minute: 0

      {
          "message": "API rate limit exceeded"
      }
      ```

    - After sending more than 10 requests, encountered the rate limit error for the consumer2
     
      ```
      http get http://localhost:8000/echo apikey:consumer2key
      HTTP/1.1 429 Too Many Requests
      Connection: keep-alive
      Content-Length: 41
      Content-Type: application/json; charset=utf-8
      Date: Tue, 21 Feb 2023 07:21:21 GMT
      RateLimit-Limit: 10
      RateLimit-Remaining: 0
      RateLimit-Reset: 39
      Retry-After: 39
      Server: kong/2.8.0.0-enterprise-edition
      X-Kong-Response-Latency: 2
      X-RateLimit-Limit-Minute: 10
      X-RateLimit-Remaining-Minute: 0

      {
          "message": "API rate limit exceeded"
      }
      ```

## Activity 3

* Install k3s
  ```
  curl -sfL https://get.k3s.io | sh -
  export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
  sudo chmod 644 $KUBECONFIG
  ```
* Create `kong` namespace
  ```
  kubectl create namespace kong
  ```

* Creae secret for kong license
  ```
  kubectl create secret generic kong-enterprise-license --from-file=./license -n kong
  ```

* Install kong
  ```
  kubectl apply -f https://bit.ly/k4k8s-enterprise-install
  ```

* Install kong ingress
  ```
  kubectl apply -f https://bit.ly/kong-ingress-dbless
  ```

* Check install status
  ```
  kubectl get pods -n kong
  ```

* Create echo deployment and service
  ```
  kubectl apply -f k8s/echo-deploy-service.yaml
  ```

* Deploy Ingress
  ```
  kubectl apply -f k8s/ingress.yaml
  ```

* Check ingress rule status
  ```
  curl -i localhost:8000/echo
  ```

* Create key-auth plugin for echo service
  ```
  kubectl apply -f k8s/key-auth-plugin.yaml
  ```

* Create API key for consumer1
  ```
  kubectl create secret generic consumer1-apikey  \
    --from-literal=kongCredType=key-auth  \
    --from-literal=key=consumer1key
  ```

* Create API key for consumer2
  ```
  kubectl create secret generic consumer2-apikey  \
    --from-literal=kongCredType=key-auth  \
    --from-literal=key=consumer2key
  ```

* Create consumer1
  ```
  kubectl apply -f k8s/consumer1.yaml
  ```

* Create consumer2
  ```
  kubectl apply -f k8s/consumer2.yaml
  ```

* Create rate limit plugin for consumer1
  ```
  kubectl apply -f k8s/rate-limit-consumer1.yaml
  ```

* Create rate limit plugin for consumer2
  ```
  kubectl apply -f k8s/rate-limit-consumer2.yaml
  ```

  - After hitting more than 5 request for the consumer1. Rate limit reached:
    ```
    venkat@venkat-latitude-3420:~/workspace/kong-assignment$ http get http://localhost:8000/echo apikey:consumer1key
    HTTP/1.1 429 Too Many Requests
    Connection: keep-alive
    Content-Length: 41
    Content-Type: application/json; charset=utf-8
    Date: Wed, 22 Feb 2023 13:44:49 GMT
    RateLimit-Limit: 5
    RateLimit-Remaining: 0
    RateLimit-Reset: 11
    Retry-After: 11
    Server: kong/3.1.1
    X-Kong-Response-Latency: 1
    X-RateLimit-Limit-Hour: 100
    X-RateLimit-Limit-Minute: 5
    X-RateLimit-Remaining-Hour: 74
    X-RateLimit-Remaining-Minute: 0

    {
        "message": "API rate limit exceeded"
    }
    ```

  - After hitting more than 10 requests for the consumer2. Rate limit reached:
    ```
    venkat@venkat-latitude-3420:~/workspace/kong-assignment$ http get http://localhost:8000/echo apikey:consumer2key
    HTTP/1.1 429 Too Many Requests
    Connection: keep-alive
    Content-Length: 41
    Content-Type: application/json; charset=utf-8
    Date: Wed, 22 Feb 2023 13:46:37 GMT
    RateLimit-Limit: 10
    RateLimit-Remaining: 0
    RateLimit-Reset: 23
    Retry-After: 23
    Server: kong/3.1.1
    X-Kong-Response-Latency: 1
    X-RateLimit-Limit-Hour: 100
    X-RateLimit-Limit-Minute: 10
    X-RateLimit-Remaining-Hour: 65
    X-RateLimit-Remaining-Minute: 0

    {
        "message": "API rate limit exceeded"
    }
    ```

## Reference Material
  You will likely need to refer to the documentation for help. Most of the answers are in Kong
  Documentation below :
  - https://docs.konghq.com/enterprise/
  - https://docs.konghq.com/hub/
  - https://docs.konghq.com/gateway/2.8.x/reference/configuration/
  - https://docs.konghq.com/gateway/latest/reference/configuration/
  - https://docs.konghq.com/kubernetes-ingress-controller/