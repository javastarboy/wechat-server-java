# 微信公众号扫码关注登录后端服务
> 用以实现 web 网站上，通过微信扫码关注后进行登录的后端实现。
> 
> 针对 [one-api](https://github.com/javastarboy/one-api) 、[one-hub](https://github.com/javastarboy/one-hub) 进行开发，其他项目也可以使用。
> 
> 主要是让熟悉Java开发的伙伴 把本项目集成到已经部署的接收微信服务器项目中，也可以直接进行使用。
> 
> 本项目目前没有前端页面，靠配置文件进行配置。

## 1、功能
+ [x] Access Token 自动刷新 & 提供外部访问接口
+ [x] 登录验证
+ [x] 自定义回复

## 2、展示
[领航AGI 聚合平台](https://javastarboy.com) 的微信登录就是基于本项目构建的, 欢迎前去体验。

## 3、部署
### 3.1 基于Docker进行部署
执行:
```shell
docker run -d --name wechat-server -p 3080:8080 \
-v /www/projects/wechat-server-java/logs:/fan/logs \
-v /www/projects/wechat-server-java/src/main/resources/application.yaml:/fan/application.yaml \
--restart always fanstars1020/wechat-server:v1.1-alpha
```
其中，参数大家可以按自己实际情况自行调整：
- `-p 3080:8080`：3080 为宿主机暴漏端口，可以自行更改，记得放开防火墙以及安全组
- `-v /home/tomcat/wechat-server-java-main/logs`：宿主机日志记录位置
- `-v /home/tomcat/wechat-server-java-main/src/main/resources/application.yaml`：宿主机配置文件存放位置，你需要在配置文件中调整您的微信配置信息
在 `./application.yaml` 中进行你的配置

### 3.2 基于Docker-compose进行部署
同样的要在 ./application.yaml 中进行你的配置
```yaml
version: '3'

services:
  wechat-server:
    image: fanstars1020/wechat-server:v1.1-alpha
    container_name: wechat-server
    ports:
      - 3080:8080
    volumes:
      - ./home/tomcat/wechat-server-java-main/logs:/fan/logs
      - ./home/tomcat/wechat-server-java-main/application.yaml:/fan/application.yaml
    restart: always
    networks:
      - default

# Networks
networks:
  default:
    driver: bridge
    name: fan
```

## 4、配置
1. application.yaml
```yaml
server:
   port: 8080 # 端口

wx:
   mp:
      appId: xxxxxxxxxx  # 公众号 AppID
      secret: xxxxxxxxxx # 公众号 AppSecret
      token: xxxxxxxxxx  # 公众号 消息token
      aesKey: xxxxxxxxxx # 公众号 EncodingAESKey
      api-host: https://api.weixin.qq.com
      open-host: https://open.weixin.qq.com
      mp-host: https://mp.weixin.qq.com
api:
   token: xxxxxxxxxx        # 访问凭证 token, one-api/one-hub 中 Wechat Server 访问凭证要设置一样
   send-code-keyword: 验证码 # 发送code关键字，公众号对话框输入这三个字将获取到验证码
   code-template: "您正在登录领航AGI大模型聚合平台, 请将验证码录入到页面中点击提交，您当前的验证码是: ${code}" # 发送验证码的模板
   code-length: 6           # 验证码长度
   code-expire-time: 300000 # 验证码过期时间，以毫秒为单位
   width: 300  # 宽
   height: 300 # 高
   margin: 1   # 边距
   qrcode-scene-id: 1008   # 二维码场景ID
   qrcode-expire-time: 600 # 二维码过期时间，以秒为单位
   custom-reply-map:       # 自定义回复键值对
      lol: "英雄联盟"
      cf: "窜越火线"
      dnf: "地下城与勇士"
```
2. 前往[微信公众号配置页面 -> 设置与开发 -> 基本配置](https://mp.weixin.qq.com/)填写以下配置：
   - `URL` 填：`https://<your.domain>/mp/portal/<your.appid>`
   - `Token` 首先在我们的配置页面随便填写一个 Token，然后在微信公众号的配置页面填入同一个 Token 即可。
   - `EncodingAESKey` 点随机生成，然后在我们的配置页面填入该值。
   - 消息加解密方式都可以选择。
   - 需企业订阅号或者企业服务号才有接口权限 
   - 域名需要备案（微信官方限制），且80端口鉴权，也可以用nginx做代理准发

3. Nginx 配置，解放 80 端口
```nginx
    # wechat-server 微信登录
    location /mp/portal/你的公众号AppID {
        proxy_http_version 1.1;
        proxy_pass http://localhost:3080/mp/portal/你的公众号AppID;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header Accept-Encoding gzip;
        proxy_read_timeout 300s;
    }
    
    # wechat-server 微信登录
    location /api/wechat/qrcode {
        proxy_http_version 1.1;
        proxy_pass http://localhost:3080/api/wechat/qrcode;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header Accept-Encoding gzip;
        proxy_read_timeout 300s;
    }
```

## 5、API
### 获取 Access Token
1. 请求方法：`GET`
2. URL：`/api/wechat/access_token`
3. 无参数，但是需要设置 HTTP 头部：`Authorization: <token>`

### 通过验证码查询用户 ID
1. 请求方法：`GET`
2. URL：`/api/wechat/user?code=<code>`
3. 需要设置 HTTP 头部：`Authorization: <token>`

### 动态获取二维码
1. 请求方法：`GET`
2. URL：`/api/wechat/qrcode`
3. 无参数

### 注意
需要将 `<token>` 和 `<code>` 替换为实际的内容。

## 联系我们（@AGI舰长，备注：new-hub）

<table>
  <tr>
    <td align="center">
      <picture>
        <img height="300" src="https://oss.javastarboy.com/agi/%E4%B8%AA%E4%BA%BA%E4%BC%81%E5%BE%AE%E4%BA%8C%E7%BB%B4%E7%A0%81.png">
      </picture>
      <br/>
      （微信号）
    </td>
    <td align="center">
      <picture>
        <img height="300" src="https://lobechat2vercel.oss-cn-beijing.aliyuncs.com/prod/%E5%BE%AE%E4%BF%A1%E4%BA%A4%E6%B5%81%E7%BE%A4.png">
      </picture>
      <br/>
      （AI全栈交流群）
    </td>
  </tr>
</table>


## 请我喝咖啡

如果您觉得对您有所帮助，欢迎您的赞助。您的支持将使我有更多的动力继续维护和改进这个项目。
您可以通过扫描下面的二维码来请我喝杯咖啡：


<table>
  <tr>
    <td align="center">
      <picture>
        <img height="400" src="https://lobechat2vercel.oss-cn-beijing.aliyuncs.com/prod/%E5%BE%AE%E4%BF%A1%E6%94%B6%E6%AC%BE%E7%A0%81.jpeg">
      </picture>
    </td>
    <td align="center">
      <picture>
        <img height="400" src="https://lobechat2vercel.oss-cn-beijing.aliyuncs.com/prod/%E6%94%AF%E4%BB%98%E5%AE%9D%E6%94%B6%E6%AC%BE%E7%A0%81.jpeg">
      </picture>


    </td>

  </tr>
</table>