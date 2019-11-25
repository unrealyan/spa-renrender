# Prerender 项目介绍
### SEO优化原理
#### 利用预渲染让搜索引擎来爬的时候渲染出来页面



### Be applicable SPA , React Vue  ...


![1574665884197.jpg](https://cdn.steemitimages.com/DQmPoXxusf6aUxc3eKhACHc1wp77kRSFYSFpKRkAPNx3V7h/1574665884197.jpg)


#### nginx.conf 

```
location @prerender {
           set $prerender 0;
           if ($http_user_agent ~* "googlebot|bingbot|yandex|baiduspider|twitterbot|facebookexternalhit|rogerbot|linkedinbot|embedly|quora link preview|showyoubot|outbrain|pinterest\/0\.|pinterestbot|slackbot|vkShare|W3C_Validator|whatsapp") {
               set $prerender 1;
           }
           if ($args ~ "_escaped_fragment_") {
               set $prerender 1;
           }
           if ($http_user_agent ~ "Prerender") {
               set $prerender 0;
           }
           if ($uri ~* "\.(js|css|xml|less|png|jpg|jpeg|gif|pdf|doc|txt|ico|rss|zip|mp3|rar|exe|wmv|doc|avi|ppt|mpg|mpeg|tif|wav|mov|psd|ai|xls|mp4|m4a|swf|dat|dmg|iso|flv|m4v|torrent|ttf|woff|svg|eot)") {
               set $prerender 0;
           }
           if ($prerender = 1) {
               rewrite .* /render?url=$scheme://$host$request_uri? break;
                  proxy_pass http://prerender-svc:3000;
           }
           if ($prerender = 0) {
               rewrite .* /index.html break;
           }
   }
```

#### dockerfile

``` 
FROM node:10.16.0
MAINTAINER unrealyan@gmail.com

EXPOSE 3000

# Install dumb-init to rape any Chrome zombies
RUN wget https://github.com/Yelp/dumb-init/releases/download/v1.2.0/dumb-init_1.2.0_amd64.deb
RUN dpkg -i dumb-init_*.deb

# Install Chromium.
RUN \
  wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - && \
  echo "deb http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google.list && \
  apt-get update && \
  apt-get install -y google-chrome-stable && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*
RUN google-chrome-stable --no-sandbox --version > /opt/chromeVersion

RUN mkdir -p /usr/src/app
RUN groupadd -r prerender && useradd -r -g prerender -d /usr/src/app prerender
RUN chown prerender:prerender /usr/src/app

USER prerender
WORKDIR /usr/src/app

COPY yarn.lock /usr/src/app/
COPY package.json /usr/src/app/
RUN yarn --pure-lockfile install
COPY . /usr/src/app

CMD [ "dumb-init", "yarn", "prod" ]
```



#### 
server.js

```
const prerender = require('prerender');

const forwardHeaders = require('./plugins/forwardHeaders');
const stripHtml = require('./plugins/stripHtml');
const healthcheck = require('./plugins/healthcheck');
const removePrefetchTags = require('./plugins/removePrefetchTags');
const log = require('./plugins/log');
const consoleDebugger = require('./plugins/consoleDebugger');

const options = {
	pageDoneCheckInterval: process.env.PAGE_DONE_CHECK_INTERVAL || 500,
	pageLoadTimeout: process.env.PAGE_LOAD_TIMEOUT || 20000,
	waitAfterLastRequest: process.env.WAIT_AFTER_LAST_REQUEST || 250,
	chromeFlags: [ '--no-sandbox', '--headless', '--disable-gpu', '--remote-debugging-port=9222', '--hide-scrollbars' ],
};
console.log('Starting with options:', options);

const server = prerender(options);

server.use(log);
server.use(healthcheck('_health'));
server.use(forwardHeaders);
server.use(prerender.blockResources());
server.use(prerender.removeScriptTags());
server.use(removePrefetchTags);
server.use(prerender.httpHeaders());
if (process.env.DEBUG_PAGES) {
	server.use(consoleDebugger);
}
server.use(stripHtml);

server.start();

```
