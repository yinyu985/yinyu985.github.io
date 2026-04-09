---
title: Flask+Vue 前后端分离记录
slug: flask+vue-front-and-rear-separation-records
tags: [Python, Flask]
date: 2023-07-12T20:13:14+08:00
---

如题，记录告警平台从原来的 layui 升级到 Vue，并实现 Flask+Vue 前后端分离，记录前后端三种解决跨域的方式（有点像茴香豆的四种写法？没事，技多不压身）<!--more-->

### Flask 解决跨域

不需要 `render_template` 了，只需要提供 API 给前端提供数据，需要解决跨域的问题，或者通过前端代理解决，或者通过 Nginx 代理解决（一共三种）。

```python
from flask import Flask, render_template
from flask_cors import *

"""flask-cors解决跨域问题的主角"""
from models import model_to_dict
from models import Alarm, AlarmGroup, db
from fake_data_generator import generate_fake_data

app = Flask(__name__)
CORS(app, resources={r'/*': {'origins': 'http://localhost:5173'}})
# CORS(app, resources={r'/*': {'origins': '*'}})
"""将导入的CORS应用，指定来源为任意，或者本地的5173"""

app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://postgres:123456@localhost:5432/yinyu'
db.init_app(app)

@app.route('/')
def hello_world():
    ## 测试数据库连接成功
    alarm = Alarm.query.first()
    print(alarm.name)
    return alarm.name
"""为了避免有时候容器没起，数据库没连上，这里做个测试，连上了，页面就会显示数据库中的一个名字"""

@app.route('/api/alarm', methods=['GET'])
def get_alarms():
    alarms = Alarm.query.all()
    alarms_list = [model_to_dict(a) for a in alarms]
    """query查到的是一个ORM对象，在model.py里面有model_to_dict的方案，在头部导入了"""
    return alarms_list

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
        generate_fake_data()
    """携带上下文，清空数据库，重新生成数据"""
    app.run()
```

### Vue 操作

通过 axios 模块，安装 `npm install axios`，在 `src` 路径下创建 `axios.js`。

```javascript
import axios from 'axios'

const instance = axios.create({
  baseURL: 'http://localhost:5000'
  // 指定请求的基本地址，也就是后面的 api 会接在这个的后面的
})

export default instance
// 还有一些额外的配置可以添加，本次不深究
```

`app.vue` 是 Vue 的程序入口文件，配置 axios 请求后端。

```vue
<template>
  <div>
    <h1>{{ message }}</h1>
  </div>
</template>

<script>
import axios from './axios'
// 注意，这里需要引入上一步的axios，同级目录

export default {
  data() {
    return {
      message: ''
    }
  },
  mounted() {
    // 使用axios请求/api/alarm，也就是请求baseURL+/api/alarm
    axios.get('/api/alarm', {})
      .then(response => {
        console.log(response.data) // 打印响应数据
        this.message = response.data // 将响应数据存储在 message 中
      })
      .catch(error => {
        console.log(error) // 打印错误信息
      })
  }
}
</script>

<style lang="scss" scoped>
.logo {
  width: 120px;
  height: 120px;
}

nav {
  padding: 10px;

  a {
    padding: 10px;
  }
}
</style>
```

### Vue 代理解决跨域

`vite.config.js` 是 vite 的配置文件，可以在这里配置一个代理，从而解决跨域问题。

```js
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

// https://vitejs.dev/config/
export default defineConfig({
  server: {
    // proxy: {
    //   '^/api': {
    //     target: 'http://localhost:5000',
    //     changeOrigin: true,
    //     rewrite: (path) => path.replace(/^\/api/, ''),
    //   },
    // },
    // 以上被注释掉的部分就是有效内容，为了避免冲突，使用 Flask 即可。
  },
  plugins: [
    vue(),
  ],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  }
})
```

在经历过 404（链接错了）、500（请求不到后端，本质还是链接错了）、ERR_SSL_PROTOCOL_ERROR（AI 补全补了个 https，没注意到），和跨域错误后：

![image-20230712210636499](https://s2.loli.net/2023/07/12/CpnLcmVQdf45yYq.png)

完事，收工！

### Nginx 解决跨域

```lua
server {
  listen 80;
  server_name api.example.com;

  location / {
    proxy_pass http://localhost:5000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    # 上面的set_header就是解决跨域的配置
  }
}

server {
  listen 80;
  server_name app.example.com;

  location /api {
    rewrite ^/api/(.*) /$1 break;
    proxy_pass http://api.example.com;
  }

  location / {
    root /var/www/app;
    try_files $uri $uri/ /index.html;
  }
}
```