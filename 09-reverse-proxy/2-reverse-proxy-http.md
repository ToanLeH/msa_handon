# Test Reverse Proxy with REST Client

## Add Extension **REST Client** to visual code
- From Extension management, add **REST Client** to visual code

## Create folder **REST Client**
- Create folder **REST Client**
- Create file **reverse-proxy.http** with below content
``` http
@host_name=localhost
@port=8080
@host={{host_name}}:{{port}}
@client_id=m2m.client
@scope=productapi.read
@client_secret=511536EF-F270-4058-80CA-1C89C192F69A
@grant_type=client_credentials
@token=
###
POST https://{{host}}/identity-api/connect/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Cache-Control: no-cache

client_id={{client_id}}
&scope={{scope}}
&client_secret={{client_secret}}
&grant_type={{grant_type}}

###
GET https://{{host}}/product-api/v1/product HTTP/1.1
Authorization: bearer {{token}}
Accept: */*
```

## Test access to product api with JWT token
- Open 3 terminal, start MSA.IdentityServer, MSA.ProductService and MSA.ReverseProxy project
- Hit the identity endpoint to get access token
- past the access token via param **@token**
- Hit the product endpoint and ensure you can access a list of products