---
layout: post
title: "Fixed CORS Issue when Enable AWS Lambda Proxy in AWS API Gateway"
categories: AWS
excerpt: To fix CORS issue when enable AWS Lambda Proxy in AWS API Gateway. You need to add header response in your lambda. It's different if you are not enable Lambda Proxy, you dont need to add the header response. This the example how to add header response of AWS Lamda in JAVA
---
To fix CORS issue when enable AWS Lambda Proxy in AWS API Gateway. You need to add header response in your lambda. It's different if you are not enable Lambda Proxy, you dont need to add the header response. This the example how to add header response of AWS Lamda in JAVA :
 ```
...
response.setBase64Encoded(false);
response.setStatusCode(200);
Gson gson = new Gson();
response.setBody(gson.toJson(map));

Map<String, String> headerMap = new HashMap<>();
headerMap.put("Access-Control-Allow-Origin","*");
headerMap.put("Access-Control-Allow-Methods","OPTIONS, POST");
headerMap.put("Allow", "OPTIONS, POST");
response.setHeaders(headerMap);
...
 ```
 