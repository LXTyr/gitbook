# 在原本链项目服务端渲染化过程中的一些总结

### 配置代理

接口请求我们可以配置反向代理避免跨域，在nuxt.config.js中添加以下配置,其中cookieDomainRewrite是为了让转发的cookie的domain重写为空，有些接口的cookie会强制设置cookie的domain,会导致本地的接口请求不会带上cookie所以要加这项

```javascript
  axios: {
    baseURL: this.env.SERVER_ENDPOINT,
    credentials: true,
    proxy: {
      cookieDomainRewrite: ''
    }
  },
  proxy: {
    '/yuanbenlian/': {
      target: this.env.SERVER_ENDPOINT
    },
    '/anchor/': {
      target: this.env.SERVER_ENDPOINT
    }
  }
```

### 公共scss变量，混合

想要在每个页面导入公共的scss变量，我们可以在nuxt.config.js中配置styleResources，nuxt内部使用styleResourcesPlugin将公共scss文件和组件中的scss进行混合

```javascript
styleResources: {
  scss: [
    './styles/_functions.scss',
    './styles/_variables.scss',
    './styles/_mixins.scss'
  ]
}
```

### 环境区分

通过传入YUANBEN_ENV环境变量进行环境的区分，当前有staging和production两个环境。
然后根据传入的环境可以根据项目根目录下的下列文件制定环境变量

```
.env                # 在所有的环境中被载入
.env.local          # 在所有的环境中被载入，但会被 git 忽略
.env.[YUANBEN_ENV]         # 只在指定的模式中被载入
.env.[YUANBEN_ENV].local   # 只在指定的模式中被载入，但会被 git 忽略
```

一个环境文件只包含环境变量的“键=值”对：

```
SERVER_ENDPOINT=https://sapi.yuanbenlian.com
WEBSOCKET_ENDPOINT=wss://sapi.yuanbenlian.com/yuanbenlian/monitor
```


有的时候你可能有一些不应该提交到代码仓库中的变量，尤其是当你的项目托管在公共仓库时。这种情况下你应该使用一个 `.env.local` 文件取而代之。本地环境文件默认会被忽略，且出现在 .gitignore 中。

`.local` 也可以加在指定环境的文件上，比如 `.env.development.local` 将会在 development 模式下被载入，且被 git 忽略。

### 在客户端使用环境变量

.env*文件中的配置将会被混合到环境变量中，客户端处将会被`webpack.DefinePlugin`静态嵌入到客户端侧的包中。你可以在应用的代码中这样访问它们：

```
console.log(process.env.SERVER_ENDPOINT)
```

在构建过程中，`process.env.SERVER_ENDPOINT`将会被相应的值取代，在`SERVER_ENDPOINT=https://sapi.yuanbenlian.com`的情况下,它会被替换成`https://sapi.yuanbenlian.com`


### Plugins和middleware的区别

Plugin允许在vue根组件实例化之前为应用提供一些额外的功能，通常我们可以在Plugin中暴露一些公共方法在vue原型上，这很适合我们将第三方库中的功能扩展到vue中

假如想使用['vue-notifications'](https://github.com/se-panfilov/vue-notifications)来展示通知信息，我们需要在app启动之前设置插件

文件`plugins/vue-notifications.js`
```
import Vue from 'vue'
import VueNotifications from 'vue-notifications'

Vue.use(VueNotifications)
```

之后在`nuxt.config.js`中配置`plugins`字段
```
export default {
  plugins: ['~/plugins/vue-notifications']
}
```

middleware是指router中的middleware，middleware将在每次页面展示之前触发,还可以在页面中指定middleware，表示只有该页面在展示之前会调用。

plugins和middleware的共同点是都会在服务端调用一次，并且都接受[context](https://nuxtjs.org/api/context)作为第一个参数，因此一些服务端操作既可以放在plugins中也可以放在middleware中

不同的地方在plugins在首次页面加载的时候服务端和客户端都会调用一次，之后无论页面怎么切换都不会在客户端调用，middleware在页面首次加载的时候只会在服务端调用，并不会在客户端调用，然后之后在前端页面切换的时候每次都会调用，而且plugins可以在app中注入公共方法，初始化vue plugin，这在middleware中是做不到的，熟知这两者的区别才能正确的进行选择将代码放在哪出。

例如在yuanbenlian项目中根据cookie来判断语言

```javascript
export default function({ app, store, req }, inject) {
  if (process.server) {
    const browserLanguage = req.headers['accept-language']
      .split(',')[0]
      .split('-')[0]
    const defaultLang = app.cookie.get('yuanben_lang') || browserLanguage
    if (defaultLang) {
      store.commit('SET_LANG', defaultLang)
    }
  }
  sync(app.store, app.router)
  // sync(store, route)
  const messages = {
    en: require('../i18n/en.json'),
    zh: require('../i18n/zh.json')
  }
  const vueI18n = new VueI18n({
    locale: store.state.locale,
    messages
  })
  app.i18n = vueI18n
  inject('i18n', vueI18n)
}
```

这个方法用到了服务端的req，并且我们需要注入Vuei18n到vue跟实例中，因此我们只能在plugins中实现这段逻辑(在middleware中获取cookie然后在plugins中设置i18n会导致页面语言闪一下)

### websocket与服务端渲染的结合

纯websocket的项目一般无法进行服务端的渲染，本质上只有当前端准备好之后才能建立websocket连接，想要实现websocket的服务端渲染,必须首先在服务端建立websocket连接，然后将获取的数据放在context中，并在服务端开启一个websocket服务，在真实服务器推送数据过来的同时，将数据推送到客户端，详细代码可以参考原本链项目中的`/server/wsClient.js`和`/server/wsServer.js`