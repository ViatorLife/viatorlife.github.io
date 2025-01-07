[https://www.cnblogs.com/CKExp/p/17392384.html](https://www.cnblogs.com/CKExp/p/17392384.html)


作为开发人员，想要自己部署一个渠道访问或是想随时访问但是奈何魔法有限，又或是海外服务器太贵，不想耗费这个钱，本文借助 [Serverless](https://martinfowler.com/articles/serverless.html?spm=a2c6h.12873639.article-detail.4.76a13633bB375u) 来搭建一下私有 ChatGPT 服务，Serverless 按照使用量来计费，个人使用下(满足工作和生活)费用相当低。


本文过程较为繁琐，也有更为简便的其他方式：

[https://rptzik3toh.feishu.cn/docx/XtrdduHwXoSCGIxeFLlcEPsdn8b](https://rptzik3toh.feishu.cn/docx/XtrdduHwXoSCGIxeFLlcEPsdn8b)


### 前言

本次搭建过程需要满足以下几个条件，需要预先满足，已经注册或是从一些渠道获取到了 Api key，有自己的备案域名。

* 已有 Api key

* 需要开通阿里云 Serverless

* 需要开通阿里云容器镜像服务

* 已有备案域名![002101072_58e11ee6-cf25-4382-9c54-a72d5dedfc9e](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002101072_58e11ee6-cf25-4382-9c54-a72d5dedfc9e.png)


### 云函数

开通服务(略过,免费开通)， [https://www.aliyun.com/product/fc](https://www.aliyun.com/product/fc)

1. 顶部选择地域，如硅谷

2. 左侧选择服务及函数

3. 创建服务，填写服务描述信息即可

![002102573_d1d8364b-2db7-44a6-902e-b28ab79f3dda](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002102573_d1d8364b-2db7-44a6-902e-b28ab79f3dda.png)

### 容器镜像服务

开通服务(掠过，免费开通)， [https://www.aliyun.com/product/acr](https://www.aliyun.com/product/acr)

1. 顶部选择海外站点，如硅谷

![002103958_8a72f545-3c20-4b58-93d3-5044d62472b5](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002103958_8a72f545-3c20-4b58-93d3-5044d62472b5.png)
1. 进入个人实例。创建镜像仓库，下一步中选择本地仓库，保存。

![002105551_d1895c1c-2af1-4dd2-8035-f6a6ee39af22](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002105551_d1895c1c-2af1-4dd2-8035-f6a6ee39af22.png)
1. 点击仓库名称进入仓库内部，该部分命令稍后会用到，此处只需看到即可。

![002107167_2228557a-4a5d-453a-9cc8-5b18ef04d1b8](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002107167_2228557a-4a5d-453a-9cc8-5b18ef04d1b8.png)

### ChatGPTVNextWeb

[https://github.com/Yidadaa/ChatGPT-Next-Web](https://github.com/Yidadaa/ChatGPT-Next-Web)

该仓库实现了 ChatGPT 的交互 UI，提供了 Docker 镜像方便部署。可将该镜像上传到个人的容器镜像服务(供 Serverless 绑定部署使用)。

1. 拉取 VNextWeb 镜像到本地

```plain
docker pull yidadaa/chatgpt-next-web
```
1. 本地登录容器镜像服务(见容器镜像服务节末图)

```plain
docker login --username=用户名 registry.us-west-1.aliyuncs.com
```
1. 标记本地 VNextWeb 镜像版本

* 查看镜像 id

```plain
docker images
```
*eg：查看本地镜像，复制Id*
![002109120_9f1456c3-49fc-4c5b-a22f-1756377f1758](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002109120_9f1456c3-49fc-4c5b-a22f-1756377f1758.png)
* 设置镜像版本

```plain
docker tag [ImageId] registry.us-west-1.aliyuncs.com/partner/chatgptnextweb:[镜像版本号]
```
*eg：此处直接标记最新版本 latest*
```plain
docker tag a58372f00c78 registry.us-west-1.aliyuncs.com/partner/chatgptnextweb:latest
```
1. 上传镜像到容器镜像服务

```plain
docker push registry.us-west-1.aliyuncs.com/partner/chatgptnextweb:[镜像版本号]
```
*eg：推送镜像*
```plain
docker push registry.us-west-1.aliyuncs.com/partner/chatgptnextweb:latest
```


### 创建函数

回到 Serverless 管理页面中，开始创建函数

![002111018_379019a4-4d87-45a2-84f8-809a1483cad0](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002111018_379019a4-4d87-45a2-84f8-809a1483cad0.png)
#### 基础设置

使用镜像创建方式，填写描述信息，选择如下设置即可。

![002112809_1bc8c76c-ee30-418c-be45-aa5dde8b4eaa](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002112809_1bc8c76c-ee30-418c-be45-aa5dde8b4eaa.png)
#### 选择镜像与端口设置

选择容器镜像服务中个人实例下镜像仓库中的镜像，仓库需要和当前 Serverless 地域相同，设置**监听端口 3000**。

![002114085_0c0d91bf-1591-4faf-9d55-08508244fb8d](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002114085_0c0d91bf-1591-4faf-9d55-08508244fb8d.png)
#### 高级设置

选择最低配置即可，该配置足矣满足个人使用。

![002115643_6f8bfe64-dc80-417b-b0e5-2a3bf80c4000](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002115643_6f8bfe64-dc80-417b-b0e5-2a3bf80c4000.png)
#### 环境变量设置

* CODE, 设置访问密码，可设置多个(逗号隔开)以方便共享给其他人使用，不设置则任何人都可访问(谨慎)。

* OPENAI_API_KEY, ChatGPT 的 Api key。

![002116926_11acb8fb-9e24-4419-bf4c-4911026ba6e9](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002116926_11acb8fb-9e24-4419-bf4c-4911026ba6e9.png)
一切完毕点击创建即可。


### 访问函数

如上创建完毕后会跳转到详情页，点击测试函数，开始部署

![002118592_4433d3be-0461-4cdc-a173-920f2779947d](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002118592_4433d3be-0461-4cdc-a173-920f2779947d.png)
如执行完毕，得到返回页面则部署成功。

![002120720_2f703020-c752-4621-bf20-e9daa6b3e5a5](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002120720_2f703020-c752-4621-bf20-e9daa6b3e5a5.png)

### 私有域名

当部署完毕云服务商会提供一个生成好的域名**(点击触发器管理)**，但不能在浏览器中访问(直接访问会下载访问页面成文件形式)，需要配合自定义域名使用。

![002122209_09cf4dd1-e54a-4b1c-be43-3fb5791035f1](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002122209_09cf4dd1-e54a-4b1c-be43-3fb5791035f1.png)
#### 添加自定义域名

点击创建自定义域名跳转到新页面中，点击添加自定义域名跳转到添加页面。复制如下红箭头地址，进入到下一步中。

![002123739_7bc520a4-1e87-462e-ae64-874acd4fdb6b](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002123739_7bc520a4-1e87-462e-ae64-874acd4fdb6b.png)
此处需要已经有了一个备案好的域名，不管是在阿里云还是腾讯云。此处以腾讯云为例，进入 dns 解析页 [https://console.dnspod.cn/dns](https://console.dnspod.cn/dns)

![002125865_5668da1f-b749-4064-9f33-071f2bc3f82c](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002125865_5668da1f-b749-4064-9f33-071f2bc3f82c.png)
添加记录，设置想要的**域名前缀**，记录类型选择 **CNAME**，记录值中粘贴上一步的复制的**公网 CNAME 值**，确认即可。

回到添加自定义域名页面，设置域名地址(域名前缀.域名主体, 例如 cn.bing.com)，再选择服务名，函数名和版本，创建即可。

![002127639_d310e620-35d5-4082-a12d-6ee57012a0fe](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002127639_d310e620-35d5-4082-a12d-6ee57012a0fe.png)
通过域名地址访问，则可看到页面内容，点击设置填写预先设置好的访问密码。便可使用自己的 ChatGPT 服务了。

![002129204_df5b573b-cb83-44b7-85f8-fddad1432db6](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241128/002129204_df5b573b-cb83-44b7-85f8-fddad1432db6.png)

>2023-05-11,望技术有成后能回来看见自己的脚步

