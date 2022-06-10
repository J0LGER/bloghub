---
layout: default
title:  "Templated"
permalink: /bloghub/templated/
date:   2022-03-09 6:17:40 -0500
categories: jekyll update
---

## Introduction 
In this application, we will take advantage of a Server-Side template injection in order to gain RCE in the flask server. 

### Step 1
Surfing the application we get 

![image](https://user-images.githubusercontent.com/54769522/172023200-340c7a1b-191e-4e23-a8aa-187391b1a440.png)

Obviously, _Proudly powered by Flask/Jinja2_ is an indication for SSTI vulnerability against the Jinja2 Engine. 

Fuzzing more through the application I thought of fuzzing for hidden API endpoints?, but I guess this is way too hard for an easy challenge, let's just test it manually first. 

### Finding the injection point 
Sending a request to */test* non-existing endpoint we get 

![image](https://user-images.githubusercontent.com/54769522/172023219-9a050ce5-7f16-44a5-981f-f123c6e9682a.png)

Page Source:

![image](https://user-images.githubusercontent.com/54769522/172023229-d54bd9f4-f727-4ac2-9b49-b51a258466a5.png) 
The endpoint name is being reflected in an HTML _<str>_ tag, which is interesting. 

Let’s try something simple ... 

![image](https://user-images.githubusercontent.com/54769522/172023248-1b915200-5084-4fd5-afed-d77bf72f68a4.png) 

Boom! template rendering applied.

Let's jump in to try out RCE payloads. 

```plaintext
GET /{{request.application.__globals__.__builtins__.__import__('os').popen('cat${IFS}/flag.txt').read()}}
```
  
We get the flag.txt 
```
HTB{t3mpl4t3s_4r3_m0r3_p0w3rfu1_th4n_u_th1nk!}
``` 
_*TIP*_: We can use the Internal Field Separator bash variable _${IFS}_ in order to inject bash spaces since endpoint path should not contain whitespaces. 
