# docker

### docker destop proxy

先试下可以自动配置clash的代理行不行，不用设置下面的配置
```
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "proxies": {
    "http-proxy": "http://host.docker.internal:3130",
    "https-proxy": "http://host.docker.internal:3130",
    "no-proxy": "*.test.example.com,.example.org, localhost, 127.0.0.1,host.docker.internal"
  }
}
```