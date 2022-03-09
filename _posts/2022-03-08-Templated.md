---
layout: post
title:  "Templated"
date:   2022-03-09 6:17:40 -0500
categories: jekyll update
---
## Introduction 
In this application, we will take advantage of a Server-Side template injection in order to gain RCE in the flask server. 

### Step 1
Surfing the application we get 
![Image-1 Broken](assets/templated/templated-1.png)

Obviously, _Proudly powered by Flask/Jinja2_ is an indication for SSTI vulnerability against the Jinja2 Engine. 

Fuzzing more through the application I thought of fuzzing for hidden API endpoints?, but I guess this is way too hard for an easy challenge, let's just test it manually first. 

### Finding the injection point 
Sending a request to */test* non-existing endpoint we get 
![Image-2 Broken](assets/templated/templated-2.png) 

Page Source:
![Image-3 Broken](assets/templated/templated-3.png) 
The endpoint name is being reflected in an HTML _<str>_ tag, which is interesting. 

Let’s try something simple ... 
![Image-4 Broken](assets/templated/templated-4.png) 

Boom! template rendering applied.

Let's jump in to try out RCE payloads. 

```python3 
GET /{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```
![Image-5 Broken](assets/templated/templated-5.png)

_*TIP*_: We can use the Internal Field Separator bash variable _${IFS}_ in order to inject bash spaces since endpoint path should not contain whitespaces. 

