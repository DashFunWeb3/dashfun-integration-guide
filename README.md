# DashFun 接入指引V0.1
* [简介](#简介)
* [接入文档](#接入文档)
* [测试方式](#测试方式)
* [服务器端API](#服务器端API)

# 简介
DashFun是一个h5游戏整合平台，运行在telegram的mini app中

# 接入文档
- DashFun的运行方式是在自身的app内包含一个iFrame，嵌入游戏，不需要在游戏方引入SDK，通过window之间发送消息进行通信
- **Unity接入参考** [**DashFun Unity 接入指引 V0.1**](https://github.com/DashFunWeb3/dashfun-integration/blob/main/integration-unity/README.md)
- **Cocos接入参考** [**DashFun Cocos 接入指引 V0.1**](https://github.com/DashFunWeb3/dashfun-integration/blob/main/integration-cocos/README.md)

## 必须要接入的消息
- [**loading**](#loading消息) 通知dashfun当前游戏的载入进度
- [**getUserProfile**](#getuserprofile消息) 获取当前登陆的tg用户信息，包括用户Id，语言等信息
- 必须接入dashfun的存档读档功能，不能存档在本地(仅限单机游戏)
- 付费UI上的付费货币必须改为钻石图标，图标地址[https://res.dashfun.games/icons/dashfun-diamond.png](https://res.dashfun.games/icons/dashfun-diamond.png) ，50钻石等于1美金
- 等级上报 (仅限网游)

## Payment相关消息
- [**requestPayment**](#requestpayment消息) 向dashfun请求支付，请求成功后会返回一个telegram invoice linke，拿到link后需要发送[**openInvoice**](#openinvoice消息)消息开启tg的支付界面
- [**openInvoice**](#openinvoice消息)开启telegram支付界面进行支付

# Telegram测试方式
- 在Telegram中搜索 DashFun [https://t.me/DashFunBot](https://t.me/DashFunBot)
- 向DashFunBot发送消息 /test 测试游戏的链接(可以是localhost链接)，bot会回复一条消息，点击即可开启测试游戏
- 测试模式下付费不会真正扣费
<!-- - 首先需要创建telegram的测试账号，[测试账号创建方法](https://docs.telegram-mini-apps.com/platform/test-environment)
- 登陆测试账号后，搜索 DashFunTest [https://t.me/DashFunTestBot](https://t.me/DashFunTestBot)
- 向DashFunTestBot发送消息 /test 测试游戏的链接，bot会回复一条消息，点击即可开启测试游戏
- **注: 测试游戏的连接必须是https协议地址** -->

# 网页测试方式
- 先运行游戏，获取运行地址，可以使用localhost
- 在浏览器中输入地址 `http://dashfun-test.nexgami.com/entry/test/game?`，并将本地运行地址加在 ? 后面
- 例如：
`http://dashfun-test.nexgami.com/entry/test/game?http://localhost:7456`
- 浏览器会出现如下界面，点击Play即可进行游戏测试
- **注：dashfun-test.nexgami.com支持http和https，测试时保证和测试游戏的链接协议匹配**

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
			desc: "购买200钻石",
			info: "200钻石",
			price: 2,
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

为了防止冒充，游戏方服务器可以通过[**验证支付结果API**](#验证支付结果api)对支付结果进行验证


---

# 服务器端API

服务器地址:
- 测试环境
  https://dashfun-server-test.nexgami.com
- 生产环境
  https://tma-server.dashfun.games
   
 
# 验证支付结果API

```http
[GET] /api/v1/payment/get
 
query:
game_id: 游戏id，向dashfun平台获取，测试模式下固定为 ForTest
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

# 上报用户等级API
游戏方的服务器端需要在用户 **登录时** 和 **升级后** 上报用户的等级

```http
[GET] /api/v1/game_report/player_level
 
query:
game_id: 游戏id，向dashfun平台获取，测试模式下固定为 ForTest
user_id: 用户的dashfun id
level:	 上报用户等级
```

此函数需要签名，详见[**签名算法**](#签名算法)


返回的结果
```json
{
	"code": 0, //0 | -1,   /0=success, -1=error
	"msg": "success" , //"success" or error message
}

```


# 签名算法
- 上报服务器的api调用均需要签名

### 签名方法
1. 为请求增加timestamp参数，值为unix timestamp，这个需要带到最终的url请求中
2. 增加secret参数，值向dashfun获取，**这个不要带到最终的url请求中**
3. 对所有参数，按照key的字母排序，从小到大
4. 将排序后的参数重新拼接成query string，注意拼接时对值进行urlencode
5. 计算成的query string的md5值
6. 将结算的结果加到最终的url请求中，参数名为sign

### 以上报用户等级api为例
原始请求为: /api/v1/game_report/player_level?game_id=a1b2c3d4&user_id=123456&level=10

增加timestam和secret，并排序
```json
{
	"game_id":"a1b2c3d4",
	"level":10,
	"secret":"ZGFzaGZ1xxxxxxx4ZWU5MjgxOTAwMA==",
	"timestamp":1728787740503,
	"user_id":"123456",
}
```

生成query string:

```http
game_id=9c4r4sdzb40&level=4&secret=ZGFzaGZ1xxxxxxx4ZWU5MjgxOTAwMA%3D%3D&timestamp=1728787740503&user_id=123456
```

对query string计算md5: 8c6dda5a5d7d75e10a00748207f1520d

最终请求为：
```http
/api/v1/game_report/player_level?game_id=a1b2c3d4&user_id=123456&level=10&timestamp=1728787740503&sign=8c6dda5a5d7d75e10a00748207f1520d
```
