# 用户管理

本文对应实现例子为[tcb-demo-basic](https://github.com/TencentCloudBase/tcb-demo-basic)中的 `用户管理` 功能，可以通过该例子体验用户注册、登录、退出登录功能。

<p align="center">
    <img src="https://main.qcloudimg.com/raw/f36ab01f3fd9e0f899c879f71d11fdff.png" width="500px">
    <p align="center">扫码体验</p>
</p>

## 准备工作

用户管理，包括用户的信息（昵称、性别、头像等的获取）、注册、登录、鉴权等，本章将会分别从围绕这几方面去讲述基于云开发如何做用户的管理。

在开始 【小程序基础场景及开发教学】 整个章节之前，建议阅读一下以下的文档或者文章：
- 阅读 [微信登录能力优化](https://developers.weixin.qq.com/community/develop/doc/000e2aac1ac838e29aa6c4eaf56409)  和 [获取用户信息](https://developers.weixin.qq.com/community/develop/doc/000c2424654c40bd9c960e71e5b009) 两篇文章。
- 跟着[【文档导读】](../guide/readme.md)把云开发的文档读一下

## 体验 DEMO

本章的案例代码，是在 [tcb-demo-basic](https://github.com/TencentCloudBase/tcb-demo-basic)。

将代码下载下来之后，遵循以下步骤可以在微信开发者工具里体验：

1. 在云开发中创建好至少一个环境

2. 在 `app.js` 中，如果使用默认环境，则没有改动。如果需要使用非默认环境，则需要在 `wx.cloud.init` 中传入环境id，如下：

<p align="center">
    <img src="../assets/user1.png" width="800px">
    <p align="center">云开发获取环境ID</p>
</p>

```js
wx.cloud.init({
  env: 'xxxx', // 环境 id
  traceUser: true
});
```

如果使用非默认环境，在云函数的 `cloud.init` 处，也需要补充环境id。

3. 在云函数根目录 `cloud/functions` 中找到函数 `user-login-register`  和 `user-session`，在两者的 `config` 目录中新建 `index.js`，填入小程序的密钥 `AppSecret`，用 `npm` 安装两个云函数的依赖，并上传两个云函数。

<p align="center">
    <img src="../assets/user4.png" width="800px">
    <p align="center">小程序获取密钥</p>
</p>

4. 在云开发的数据库中，新建 `collection`，名为 `users`。


## 案例

参考了一些常用的小程序，比如知乎大学、百果园、摩拜等等，我们发现他们的登录方式都有相通之处：

<p align="center">
    <img src="../assets/user2.png" width="1000px">
    <p align="center">知乎大学小程序登录流程</p>
</p>

基本的流程就是：
1. 用户授权小程序可获取用户的开放数据
2. 选择登录方式（微信绑定的手机/用户的其它手机）
3. 如果选用了微信绑定的手机，直接信任，注册/登录成功，而如果选用其它手机，则还需要通过发送短信进行手机验证。

接下来的案例我们会仿照这个流程，但只会做微信绑定手机直接注册登录的这种方式。

## 用户登录、注册与信息

当我们阅读小程序文档[用户信息](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/login.html)的时候，总觉得千头万绪，不知道从哪里做起。整个用户的登录、注册、获取信息过程，涉及到以下的一些接口：

1. `wx.getSetting`，看看用户有没有授权小程序，可以获取昵称、头像、性别等的用户信息
2. `wx.getUserInfo` （旧版） / `button` （新版），授权后，可通过此接口/组件获取用户信息
3. `wx.checkSession`，获取 `session_key` 后，要检查 `session_key` 是否过期
4. 如果 `session_key` 过期，则通过 `wx.login` 获取 `code` 后，在云函数中调用 [`code2Session`](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/login/code2Session.html) 更新 `session_key`
5. 通过 `button` 组件，获取手机号码加密数据，并在云函数中通过之前取得的 `session_key` 进行解密获取真实手机号码

整个流程大概如下图：

<p align="center">
    <img src="../assets/user3.png" width="483px">
    <p align="center">设计登录流程</p>
</p>

### 用户授权

对于小程序来说，必须进行用户授权，才能在后续中获取用户的开放数据。因此，我们在 `onLoad` 生命周期里，做了授权的检测（用于旧版）。在模板文件中，则设置授权按钮（用于新版）。授权后，马上将用户数据存入临时的对象中：

```js
onLoad: async function (options) {
  this.db = wx.cloud.database();
  this.checkAuthSetting();
  this.checkUser();
},

// 检测权限，在旧版小程序若未授权会自己弹起授权
checkAuthSetting() {
  wx.getSetting({
    success: (res) => {
      if (res.authSetting['scope.userInfo']) {
        wx.getUserInfo({
          success: async (res) => {
            if (res.userInfo) {
              const userInfo = res.userInfo
              // 将用户数据放在临时对象中，用于后续写入数据库
              this.setUserTemp(userInfo)
            }

            const userInfo = this.data.userInfo || {}
            userInfo.isLoaded = true
            this.setData({
              userInfo,
              isAuthorized: true
            })
          }
        })
      } else {
        this.setData({
          userInfo: {
            isLoaded: true,
          }
        })
      }
    }
  })
},

// 设置临时数据，待 “真正登录” 时将用户数据写入 collection "users" 中
setUserTemp(userInfo = null, isAuthorized = true, cb = () => {}) {
  this.setData({
    userTemp: userInfo,
    isAuthorized,
  }, cb)
},

// 手动获取用户数据
async bindGetUserInfoNew(e) {
  const userInfo = e.detail.userInfo
  // 将用户数据放在临时对象中，用于后续写入数据库
  this.setUserTemp(userInfo)
},
```

```html
<button
  wx:if="{{userInfo.isLoaded && !isAuthorized && !userInfo.nickName}}"
  class="weui-btn"
  type="primary"
  open-type="getUserInfo"
  bindgetuserinfo="bindGetUserInfoNew"
>
  授权微信后登录
</button>
```

### 数据解密

由于有些数据的安全性问题，需要在后台服务进行解密，譬如手机号码，由于有了云开发，我们可以借助云开发的云函数来做这件事情。

首先通过 `checkUser` 方法，检测 `session_key` 是否已经过期，如果过期，则重新设置，并将用户写入数据库中，此逻辑通过 `updateSession` 方法和云函数 `user-session` 共同完成。

```js
// 检测小程序的 session 是否有效
async checkUser() {
  const Users = this.db.collection('users')
  const users = await Users.get()
  console.log(users)

  wx.checkSession({
    success: () => {
      // session_key 未过期，并且在本生命周期一直有效
      // 数据里有用户，则直接获取
      if (users.data.length && this.checkSession(users.data[0].expireTime || 0)) {
        this.setUserInfo(users.data[0])
      } else {
        this.setUserInfo()
        // 强制更新并新增了用户
        this.updateSession()
      }
    },
    fail: () => {
      // session_key 已经失效，需要重新执行登录流程
      this.updateSession()
    }
  })
},

// 更新 session_key
updateSession() {
  wx.login({
    success: async (res) => {
      console.log(res)
      try {
        await wx.cloud.callFunction({
          name: 'user-session',
          data: {
            code: res.code
          }
        })
      } catch (e) {
        console.log(e)
      }
    }
  })
},
```

以下是云函数 `user-session` 的源码，它的主要作为是，通过 `wXMINIUser.codeToSession` 方法获取最新 `session_key` 后，如果有数据，则只是更新 `session_key`，如果没数据则添加该用户并插入 `sesison_key`。此 `session_key` 与用户的数据关联，为后续进行数据解密打下了基础。

```js
// 云函数入口函数
exports.main = async (event) => {
  console.log(event)
  const db = cloud.database()

  const {
    OPENID,
    APPID
  } = cloud.getWXContext()

  const wXMINIUser = new WXMINIUser({
    appId: APPID,
    secret
  })

  const code = event.code // 从小程序端的 wx.login 接口传过来的 code 值
  const info = await wXMINIUser.codeToSession(code)

  try {
    // 查询有没用户数据
    const user = await db.collection('users').where({
      _openid: OPENID
    }).get()

    // 如果有数据，则只是更新 `session_key`，如果没数据则添加该用户并插入 `sesison_key`
    if (user.data.length) {
      await db.collection('users').where({
        _openid: OPENID
      }).update({
        data: {
          session_key: info.session_key
        }
      })
    } else {
      await db.collection('users').add({
        data: {
          session_key: info.session_key,
          _openid: OPENID
        }
      })
    }
  } catch (e) {
    return {
      message: e.message,
      code: 1,
    }
  }

  return {
    message: 'login success',
    code: 0
  }
}
```

当我们在数据中存入了用户的一条空数据以及它相关的 `session_key` 后，我们可以引导用户通过小程序获取微信绑定的手机号，实现快速登陆。在模板文件中，我们添加了一个 `button` 组件，并将 `open-type` 设置为 `getPhoneNumber`。

```html
<button
  wx:if="{{userInfo.isLoaded && isAuthorized && !userInfo.phoneNumber}}"
  class="weui-btn"
  type="primary"
  open-type="getPhoneNumber"
  bindgetphonenumber="bindGetPhoneNumber"
>
  微信快速登录
</button>
```

点击 “微信快速登陆” 后，便可马上调用 `bindGetPhoneNumber`，将存放于临时对象的用户开放数据，以及加密的微信手机数据，发送到 `user-login-register` 进行解密，并存入用户的数据中。

```js
// 获取用户手机号码
async bindGetPhoneNumber(e) {
  // console.log(e.detail);
  wx.showLoading({
    title: '正在获取',
  })

  try {
    const data = this.data.userTemp
    const result = await wx.cloud.callFunction({
      name: 'user-login-register',
      data: {
        encryptedData: e.detail.encryptedData,
        iv: e.detail.iv,
        user: {
          nickName: data.nickName,
          avatarUrl: data.avatarUrl,
          gender: data.gender
        }
      }
    })
    console.log(result)

    if (!result.result.code && result.result.data) {
      this.setUserInfo(result.result.data)
    }

    wx.hideLoading()
  } catch (err) {
    wx.hideLoading()
    wx.showToast({
      title: '获取手机号码失败，请重试',
      icon: 'none'
    })
  }
},
```

详细解密数据的原理，可以参见该[文档](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/signature.html)，对于那些不熟悉 `Node.js` 的朋友也不怕，因为文档里官方提供了示例代码，照着做就可以了。

```js
const WXBizDataCrypt = require('./WXBizDataCrypt');
const {
  appId,
  secret
} = require('./config');
const cloud = require('wx-server-sdk');

const duration = 24 * 3600 * 1000; // 开发侧控制登录态有效时间

cloud.init();

// 云函数入口函数
const cloud = require('wx-server-sdk')
const WXBizDataCrypt = require('./WXBizDataCrypt')

const duration = 24 * 3600 * 1000 // 开发侧控制登录态有效时间

cloud.init()

// 云函数入口函数
exports.main = async (event) => {
  const {
    OPENID,
    APPID
  } = cloud.getWXContext()

  const db = cloud.database()
  const users = await db.collection('users').where({
    _openid: OPENID
  }).get()

  if (!users.data.length) {
    return {
      message: 'user not found',
      code: 1
    }
  }

  // 进行数据解密
  const user = users.data[0]
  const wxBizDataCrypt = new WXBizDataCrypt(APPID, user.session_key)
  const data = wxBizDataCrypt.decryptData(event.encryptedData, event.iv)

  const expireTime = Date.now() + duration

  try {
    // 将用户数据和手机号码数据更新到该用户数据中
    const result = await db.collection('users').where({
      _openid: OPENID
    }).update({
      data: {
        ...event.user,
        phoneNumber: data.phoneNumber,
        expireTime
      }
    })

    if (!result.stats.updated) {
      return {
        message: 'update failure',
        code: 1
      }
    }
  } catch (e) {
    return {
      message: e.message,
      code: 1
    }
  }


  return {
    message: 'success',
    code: 0,
    data: {
      ...event.user,
      ...data
    },
  }
}
```

## 退出登陆

要做到使用户退出登陆，以上的步骤还不足够，因为我们无法去控制微信官方小程序 `session_key` 的过期时间。如果我们不需要用户退出登陆，单纯依赖 `wx.checkSession` 就可以作为用户登陆态失效的办法。但如果我需要允许用户主动退出呢？

我们可以在用户数据里加一个 `expireTime` 字段，用于记录用户登陆态失效的时间，在云函数 `user-login-register` 里就有 `expireTime` 的相关配置和写入逻辑。

```js
// 节选自 `user-login-register`
const duration = 24 * 3600 * 1000; // 开发侧控制登录态有效时间，此处表时24小时，即1天

// ...... 此处省略其它代码

// 将 expireTime 写入用户数据里
const result = await db.collection('users').where({
  _openid: OPENID
}).update({
  data: {
    ...event.user,
    phoneNumber: data.phoneNumber,
    expireTime
  }
})

```

在 `checkUser` 方法中，也有调用 `checkSession` 去检测用户数据中的 `expireTime` 是否过期，如果过期，则不会再展示用户数据，并更新一下 `session_key`。

```js
// 检查用户是否登录态还没过期
checkSession(expireTime = 0) {
  if (Date.now() > expireTime) {
    return false;
  }

  return true;
},
```

以下此方法，则是用户主动点击退出登陆按钮后，触发的方法，会将用户的 `expireTime` 设零过期。
```js
// 退出登录
async bindLogout() {
  const userInfo = this.data.userInfo

  await this.db.collection('users').doc(userInfo._id).update({
    data: {
      expireTime: 0
    }
  })

  this.setUserInfo()
}
```

就这样，基本完成了一个简单有效的用户注册、登陆页面。其实小程序的注册、登陆、获取用户信息的方案多种多样，这里只是模仿了一种，而且用户是以 `_openid` 作为主要的识别字段。而像知乎、摩拜等小程序，一般都是以手机号作为主要识别字段，`openid` 会作为参考。

另外你会发现，随时可以通过云函数依然可以获得用户的 `openid`，因此如果你的小程序不需要使用手机号码，整个流程会更简单一些，基至都不需要获取 `session_key`，进行数据解密，只需要首次用户授权获取开放数据后并存下来后，以后每次都可以把用户的数据调出来（不过你需要处理一下开放数据更新的问题）。