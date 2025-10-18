---
title: nodejs连接redis
date: 2020-04-08 22:41:43
categories:
  - 随笔
tags:
  - nodejs
cover: https://static.jiangliuhong.top/blogimg/other/nodejs-logo.png
---

## 引入依赖包

```
npm install redis --save
```

## 工具类模式redisUtil

```javascript
import redis from 'redis';
const redisUtil = {
    config: {
        url: 'localhost',
        port: '6379',
        password: '123456'
    },
    client: null,
    createClient(conf) {
        this.client = redis.createClient(conf.port, conf.url, {});
        this.client.auth(conf.password, function (res) {
            console.log(res);
        });
        const _this = this;
        this.client.on('connect', function () {
            _this.client.set('author', 'Wilson', redis.print);
            _this.client.get('author', redis.print);
            console.log('connect');
        });
        this.client.on('ready', function () {
            console.log('ready');
        });
    },
    getClient(){
        if(this.client == null){
            this.createClient(this.config);
        }
        return this.client;
    },
    setKey(key, value) {
        const client = this.getClient();
        return new Promise((resolve, reject) => {
            client.set(key, value, (err, replay) => {
                if (err) {
                    reject(err);
                } else {
                    resolve(replay);
                }
            })
        })
    },
    getKey(key) {
        const client = this.getClient();
        return new Promise((resolve, reject) => {
            client.get(key, (err, replay) => {
                if (err) {
                    reject(err);
                } else {
                    resolve(replay);
                }
            })
        })
    }
}
export default redisUtil;
```

## 对象模式

> es6语法

### 定义一个RedisClient对象

```javascript
import redis from 'redis';
class RedisClient {
    /**
     * 构造函数
     * @param {Object} config 
     */
    constructor(config) {
        this.config = config;
        this.client = null;
    }

    /**
     * 创建连接
     * @param {Object} conf 配置信息
     */
    createClient(conf) {
        this.client = redis.createClient(conf.port, conf.url, {});
        this.client.auth(conf.password, function (res) {
            console.log(res);
        });
        const _this = this;
        this.client.on('connect', function () {
            _this.client.set('author', 'Wilson', redis.print);
            _this.client.get('author', redis.print);
            console.log('connect');
        });
        this.client.on('ready', function () {
            console.log('ready');
        });
    }

    /**
     * 获取链接
     */
    getClient() {
        if (this.client == null) {
            this.createClient(this.config);
        }
        return this.client;
    }

    /**
     * 设置值
     * @param {String} key 
     * @param {Object} value 
     */
    setKey(key, value) {
        const client = this.getClient();
        return new Promise((resolve, reject) => {
            client.set(key, value, (err, replay) => {
                if (err) {
                    reject(err);
                } else {
                    resolve(replay);
                }
            })
        })
    }

    /**
     * 获取值
     * @param {String}} key 
     */
    getKey(key) {
        const client = this.getClient();
        return new Promise((resolve, reject) => {
            client.get(key, (err, replay) => {
                if (err) {
                    reject(err);
                } else {
                    resolve(replay);
                }
            })
        })
    }

    /**
     * 查询key
     * @param {String}} pattern 
     */
    keys(pattern) {
        const client = this.getClient();
        return new Promise((resolve, reject) => {
            client.keys(pattern, (err, replay) => {
                if (err) {
                    reject(err);
                } else {
                    resolve(replay);
                }
            });
        });
    }
}
export default RedisClient;
```

### 对象的使用方法

```javascript
import RedisClient from "./js/app/RedisClient";
const config = {
    url: "localhost",
    port: "6379",
    password: "123456"
};
const rc = new RedisClient(config);
rc.getKey("test2").then(function(res) {
    console.log(res);
});
```
