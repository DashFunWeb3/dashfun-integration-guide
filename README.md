# DashFun 接入指引V0.1
* [简介](#简介)
* [接入文档](#接入文档)
* [测试方式](#测试方式)

# 简介
DashFun是一个h5游戏整合平台，运行在telegram的mini app中

# 接入文档
DashFun的运行方式是在自身的app内包含一个iFrame，嵌入游戏，不需要在游戏方引入SDK，通过window之间发送消息进行通信

## 必须要接入的消息
- [**loading**](#loading消息) 通知dashfun当前游戏的载入进度
- [**getUserProfile**](#getuserprofile消息) 获取当前登陆的tg用户信息，包括用户Id，语言等信息

## Payment相关消息
- [**requestPayment**](#requestpayment消息) 向dashfun请求支付，请求成功后会返回一个telegram invoice linke，拿到link后需要发送[**openInvoice**](#openinvoice消息)消息开启tg的支付界面
- [**openInvoice**](#openinvoice消息)开启telegram支付界面进行支付

# 测试方式
- 首先需要创建telegram的测试账号，[测试账号创建方法](https://docs.telegram-mini-apps.com/platform/test-environment)
- 登陆测试账号后，搜索 DashFunTest [https://t.me/DashFunTestBot](https://t.me/DashFunTestBot)
- 向DashFunTestBot发送消息 /test 测试游戏的链接，bot会回复一条消息，点击即可开启测试游戏
-  <font color="#ff9900">注: 测试游戏的连接必须是https协议地址</font> 

---
# API说明
- 向DashFun发送的消息格式
```typescript
{
	dashfun:{
		method:string, //方法名称
		payload:any, //方法携带的数据
	}
}
```
- DashFun回应的消息格式
```typescript
{
	dashfun:{
		method:string, //回应的方法名称，
		result:{ //回应的结果数据
			state: "success"|"error",
			data: any|string, //state=success时为写到的结果数据，error时为错误消息
		}, 
	}
}
```


# loading消息
- loading方法通知DashFun当前游戏的加载进度，不发送这个消息，dashfun的play按钮不会变亮。
- 携带数据中 ```payload.value``` 取值范围0-100，收到loading消息后，dashfun的play按钮会变成载入状态，直到收到payload.value == 100 时，变为点击状态。
- 此方法没有回应消息

- 发送示例
```typescript
const {parent} = window;
const msg = {
	dashfun:{
		method:"loading",
		payload:{
			value:1, //value取值范围0-100
		}
	}
}
parent.postMessage(msg, "*")
```

# getUserProfile消息
- 获取当前用户的profile
- profile中的id为dashfun的用户id，游戏方可以用这个id直接让用户登录游戏，不需要注册过程

```typescript
type UserProfile = {
	id: string				//dashfun userId
	channelId: string		//渠道方id，此处为TG的用户Id
	displayName: string 	//显示名称
	userName: string 		//用户名
	avatarUrl: string		//avatar地址
	from: number			//用户来源
	createData: number		//创建时间
	loginTime: number		//登录时间
	logoffTime: number		//登出时间
	language: string		//language code
}

const { parent } = window;

const msg = {
	dashfun:{
		method:"getUserProfile"
	}
}

//发送消息
parent.postMessage(msg, "*")

//接收结果
window.addEventListener("message", ({data}) => {
	const dashfun = data.dashfun;
	if(!dashfun) return;

	//回应消息的名字=发送消息的名字+Result
	if(dashfun.method == "getUserProfileResult"){
		console.log(dashfun.result.data) //UserProfile
	}
})
```

# requestPayment消息
- 向dashfun平台请求消费，平台会返回生成的telegram平台的消费链接(invoiceLink)
- 仅返回链接，不会开启支付窗口
```typescript
type PaymentRequest = {
	title:string, //付费项目名称，将显示在telegram的支付界面上
	desc:string, //项目描述
	info:string, //支付携带信息
	price:number, //付费项目价格，单位为telegram stars
}

type PaymentRequestResult = {
	invoiceLink: string, //tg invokce 链接
	paymentId: string, //dashfun平台paymentId
}

const { parent } = window;

const msg = {
	dashfun:{
		method: "requestPayment",
		payload:{ //PaymentRequest
			title:"200钻石",
			desc: "购买200钻石"
			info: "200钻石",
			price: 2
		}
	}
}
//发送消息
parent.postMessage(msg, "*")

//接收结果
window.addEventListener("message", ({data})=>{
	const dashfun = data.dashfun;
	if(!dashfun) return;

	//回应消息的名字=发送消息的名字+Result
	if(dashfun.method == "requestPaymentResult"){
		console.log(dashfun.result.data) //PaymentRequestResult
		const {invoiceLink, paymentId} = dashfun.result.data;
	}
})

```

# openInvoice消息
- 开启telegram invokce link对应的支付页面，并等待用户支付
```typescript
type OpenInvoiceRequest = {
	invoiceLink: string, //tg invokce 链接
	paymentId: string, //dashfun平台paymentId
}

type OpenInvoiceResult = {
	paymentId: string, //dashfun平台paymentId
	status: "paid"|"canceled"|"failed"
}


const { parent } = window;

const msg = {
	dashfun:{
		method: "openInvoice",
		payload:{ //OpenInvoiceRequest
			invoiceLink: "https:/t.me/$xxxxxxx",
			paymentId: "12345"
		}
	}
}
//发送消息
parent.postMessage(msg, "*")


//接收结果
window.addEventListener("message", ({data})=>{
	const dashfun = data.dashfun;
	if(!dashfun) return;

	//回应消息的名字=发送消息的名字+Result
	if(dashfun.method == "openInvoiceResult"){
		console.log(dashfun.result.data) //OpenInvoiceResult
		const {paymentId, status} = dashfun.result.data;
	}
})

```

为了防止冒充，游戏方服务器可以通过以下api对支付结果进行验证

```http
[GET] http://dashfun-server-test.nexgami.com/api/v1/payment/get

query:
game_id: 游戏id，向dashfun平台获取
payment_id: 要查询的payment_id
user_id: 要查询支付用户的id
```

返回的结果
```json
{
    "code": 0,
    "msg": "success",
    "data": {
        "id": "6xxj917w2kg",			//支付id
        "userId": "6a1cq67o5c0",		//用户id
        "game_id": "ForTest",			//游戏id
        "payment_id": "0Xju1IRO2EVyAAAAUqN3_ztksHI", //telegram的invoice id
        "title": "100 Coin",
        "description": "100 Coin",
        "payload": "200钻石",  			//requestPayment时传入的info参数
        "currency": "XTR",
        "from": 1,
        "price": 2,
        "extraData": "https://t.me/$0Xju1IRO2EVyAAAAUqN3_ztksHI",
        "message": "stxfEpyYwZef2AzOiN2H8l3uhAhfrfJ2KmEdCZlYwAA3yhEeUmN2B-4KDqpDByBNoE5CL70lXXA2V7YJMEM-MTDwPqmQnw-RMp3gwr7tcN5JnLc1gAxk3c_N--ekCG1vOAS",
        "created_at": 1723580812293,
        "pay_at": 1723580814169,
        "status": 3			//订单状态 2=pending,3=paid,4=canceled,5=failed
    }
}
```