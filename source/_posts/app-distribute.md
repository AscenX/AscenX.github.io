---
title: APP分发服务平台搭建
date: 2024-04-27 01:12:35
tags:
- 工具
- APP分发
- 基础建设
- web
categories:
- 工具开发
---


刚开始做APP的时候用的是像蒲公英/Fir/第八区这种分发平台，这些平台有些缺点，一是不支持多包同时分发，如果同时进行多个需求，需要打包测试，就无法同时分发，即iOS同Bundle ID或者安卓同包名，只能分发一个包。第二就是这些平台对区块链APP并不友好，轻则下架APP重则ban账号。

<!-- more -->

### 1.0 分发

刚开始第一次弄分发服务比较简单，IPA/APK上传到腾讯的COS，后面使用CloudFlare的R2，获取到公开链接之后，编写一个静态HTML分发到Gitee / GitHub Pages，APK直接下载就行，IPA使用的是苹果的[itms-servers协议](https://support.apple.com/zh-cn/guide/deployment/depce7cefc4d/web)

即上传一个Plist文件到云存储/或者Git仓库上也都可以，Plist文件最重要的两个属性是 `Bundle ID` 和 下载URL，然后获取Plist的公开链接地址，如果上传到Git上就要使用git的raw地址，注意这个地址一定要是HTTPS链接

```html
<a href="itms-services://?action=download-manifest&url=https://betterbag.com/manifest.plist">安装 App</a>
```

如果在Mac上点击能出现弹窗弹出APP安装器就说明能成功获取到plist文件内容

![app-publish-1](https://github.com/AscenX/AscenX.github.io/blob/master/images/app-publish-1.jpg?raw=true)

而APK只要把下载链接放进去即可

```html
<div><a href="https://pubxxxx.apk">Android 点击下载APK</a></div>
```

这样一个基本的APP分发服务就搭建好了。
这样的服务缺点就是更新的时候比较麻烦，需要手动上传包到云存储，然后再更新GitHub资源，等待一两分钟自动部署完成即可


### 2.0 一键上传分发

后面就有想法至少把流程简化，只要已上传就自动分发好，再也不用手动更新Git仓库。
方案是全部基于CloudFlare的服务，CloudFlare对个人开发者真是太友好了，CloudFlare YYDS!

简单介绍一下需要用到的CloudFlare的服务

#### CloudFlare R2

基于AWS S3的云储存，个人免费空间有10GB！每个月有免费的百万次操作！能够获取公开链接下载，也能够使用S3的API进行操作。

#### CloudFlare Worker

能够使用脚本对CloudFlare的服务进行操作，例如对R2进行增删改查。能够公开调用API。

#### CloudFlare Pages

和GitHub Pages一样，部署静态资源。

以上3个服务组合在一起，便可达到一键使用Worker API进行上传到R2，再通过Pages进行分发


### 开始搭建

##### 1.开通R2
要开通R2云存储，个人账号需要绑定信用卡，有免费使用额度，只做分发服务的话不用担心会扣款。就算100M左右一个包，也能分发个八九十个，何况可以做更新，删除旧的包。

##### 2.开通R2的公开链接访问
在设置-R2.dev子域
![app-publish-2](https://github.com/AscenX/AscenX.github.io/blob/master/images/app-publish-2.jpg?raw=true)


##### 3.安装Wrangler

在这之前先安装Wrangler，是CloudFlare的快速部署工具

```shell
npm install wrangler -g
```

然后登录

```shell
wrangler login
```

登录成功可以使用命令看到当前登录的账号
```shell
wrangler whoami
```

然后开始创建项目


##### 4.创建Worker项目

```shell
npm create cloudflare@latest
```
输入项目名称，选择第一个 HelloWorld Worker，可选择是否使用TypeScript，我用的是TypeScript


根据需求写Worker API，有点像Node.js写API

```typescript
interface Env {
	appBucket: R2Bucket;
}

export default {

	async fetch(
		request: Request,
		env: Env,
		ctx: ExecutionContext
	): Promise<Response> {
		const bucket = env.appBucket;

		const url = new URL(request.url);
		const pathname = url.pathname
		if (pathname.length == 0) {
			return new Response('operation not found')
		}
		const key = pathname.slice(1);
		
		/// 获取所有上传的文件列表
		if (key == 'getObjectList') {
			if (request.method != "GET") {
				return new Response("Method not allow", { status: 405 });
			}
			const list = await bucket.list({
				include: ['customMetadata']
			})
			
			let resp = Response.json(list)
			resp.headers.set('Access-Control-Allow-Origin', '*')
			resp.headers.set('Access-Control-Allow-Methods', 'GET,HEAD,POST,OPTIONS')
			resp.headers.set('Access-Control-Max-Age', '86400')
			return resp
		}
		/// 上传文件，Free账户最大只支持100M，如果上传大于100M文件请使用S3 API
		if (key == 'upload') {
			if (request.method != "PUT" && request.method != "POST") {
				return new Response("Method not allow", { status: 405 });
			}
			const formData = await request.formData()
			const fileName = formData.get('fileName')
			if (fileName != null && typeof fileName == 'string') {
				const title = formData.get('title')
				const desc = formData.get('desc')
				if (typeof title == 'string' && typeof desc == 'string') {
					const obj = await bucket.put(fileName, formData.get('file'), {
						customMetadata: {
							title,
							desc,
						}
					});
					let resp = Response.json({
						code: 200,
						message : `Put ${fileName} successfully!`,
						data: obj
					})
					resp.headers.set('Access-Control-Allow-Origin', '*')
					resp.headers.set('Access-Control-Allow-Methods', 'GET,HEAD,POST,OPTIONS')
					resp.headers.set('Access-Control-Max-Age', '86400')
					return resp
				}
			}
			return new Response('upload failed')
		}
		/// 下载文件
		if (key == 'getObject') {
			if (request.method != "GET") {
				return new Response("Method not allow", { status: 405 })
			}
			const params = url.searchParams
			const name = params.get('name') ?? ''
			
			const object = await bucket.get(name);
			if (object === null) {
				return new Response("Object Not Found", { status: 404 })
			}
			const headers = new Headers()
			object.writeHttpMetadata(headers)
			headers.set("etag", object.httpEtag)
			let resp = new Response(object.body)
			resp.headers.set('Access-Control-Allow-Origin', '*')
			resp.headers.set('Access-Control-Allow-Methods', 'GET,HEAD,POST,OPTIONS')
			resp.headers.set('Access-Control-Max-Age', '86400')
			return resp
		}
		/// 获得文件信息
		if (key == 'getObjectInfo') {
			if (request.method != "GET") {
				return new Response("Method not allow", { status: 405 })
			}
			const params = url.searchParams
			const name = params.get('name') ?? ''
			
			const object = await bucket.get(name);
			if (object === null) {
				return new Response("Object Not Found", { status: 404 })
			}
			let resp = Response.json({
					code: 200,
				message: 'success',
				data: object
			})
			resp.headers.set('Access-Control-Allow-Origin', '*')
			resp.headers.set('Access-Control-Allow-Methods', 'GET,HEAD,POST,OPTIONS')
			resp.headers.set('Access-Control-Max-Age', '86400')
			return resp
		}
		/// 删除文件
		if (key == 'delete') {

			const formData = await request.formData()
			const fileName = formData.get('fileName') as string

			const object = await bucket.get(fileName);
			if (object === null) {
				return new Response("Object Not Found", { status: 404 })	
			}

			await bucket.delete(fileName)

			let resp = Response.json({
				code: 200,
				message : `Delete ${fileName} successfully!`,
			})
			resp.headers.set('Access-Control-Allow-Origin', '*')
			resp.headers.set('Access-Control-Allow-Methods', 'GET,HEAD,POST,OPTIONS')
			resp.headers.set('Access-Control-Max-Age', '86400')
			return resp
		}
		
		return new Response('operation not found')
	},
};
```


可以看到我在Env里定义了一个Bucket，需要在配置文件`wrangler.toml`里配置

```toml
[[r2_buckets]]
binding = "appBucket"
bucket_name = "app-publish"
```

配置好绑定的变量名和操作的R2 Bucket名称

##### 5.部署Worker

直接使用Wrangler就可以部署
```shell
wrangler deploy
```

需要在部署的Worker设置里绑定R2

![app-publish-4](https://github.com/AscenX/AscenX.github.io/blob/master/images/app-publish-4.jpg?raw=true)


部署好Worker之后就可以通过Worker的公开API进行调用了

![app-publish-3](https://github.com/AscenX/AscenX.github.io/blob/master/images/app-publish-3.jpg?raw=true)


**注意！第4,5步是部署一个Worker项目，其实把Worker放进Pages项目更好，这样就只用维护一个项目，但是经过测试，放进Pages的Worker获取静态资源极其不稳定，创建了两个项目测试亦是如此。具体可参照[CloudFlare Pages-高级模式](https://developers.cloudflare.com/pages/functions/advanced-mode/)**

如要切换这种模式,修改Pages项目里的`wrangler.toml`文件中的
```toml
pages_build_output_dir = ".vercel/output/static"
```
改成
```toml
pages_build_output_dir = "worker"
```
这样所有请求就会走worker文件里的`_worker.ts`, 和Worker项目里的不同的是，在其他URL的时候要返回静态资源

```typescript
return env.ASSETS.fetch(request);
```

**目前worker文件夹里的`_worker.ts`文件文件未生效**


##### 6.创建Pages项目

使用wrangler创建Website or web app
我选择的是Next.js项目

##### 7.解决Worker API上传最大100M的问题

R2是基于AWS S3的，直接使用AWS S3的api，安装S3 SDK

```shell
@aws-sdk/client-s3
```

```typescript
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const client = new S3Client({
    credentials: {
        accessKeyId: '',
        secretAccessKey: ''
    },
    endpoint: '',
    region: 'auto'
})

const tohex = (str: string) => {
    var hex, i;

    var result = "";
    for (i=0; i<str.length; i++) {
        hex = str.charCodeAt(i).toString(16);
        result += ("000"+hex).slice(-4);
    }

    return result
}

const s3UploadFile = async (file: File, title: string, desc: string, timestamp: number) => {

    let key = file.name

    if (timestamp > 0) {
        const type = key.slice(key.lastIndexOf('.'))
        key = key.slice(0, key.lastIndexOf('.'))
    
        key = `${key}_${timestamp}${type}`
    }


    const titleStr = tohex(title)
    const descStr = tohex(desc)

    const uploadCmd = new PutObjectCommand({
        "Bucket" : 'app-publish',
        "Key" : key,
        "Body" : file,
        "Metadata" : {
            "title" : titleStr,
            "desc" : descStr
        }
    })

    try {
        const resp = await client.send(uploadCmd)
        return resp
    } catch (err) {
        return null
    }
}

export {
    s3UploadFile
}
```

endpoint就是使用R2设置中的S3 API
accessKeyId和secretAccessKey需要在R2中创建一个API Token,创建完填入即可

![app-publish-5](https://github.com/AscenX/AscenX.github.io/blob/master/images/app-publish-5.jpg?raw=true)


**如果不用上传100M以上的文件可以不用第7步**

##### 8.部署

```shell
npm run deploy
```

这样项目就部署完成啦


项目源码上传到了[我的github](https://github.com/AscenX/app-publish)