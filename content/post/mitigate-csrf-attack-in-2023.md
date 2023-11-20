---
title: "在 2023 年，如何防止 CSRF 攻击"
description: "It's damn simple"
date: 2023-11-21T00:00:00+08:00
---

## TL;DR

如果：

- 你的前端应用只使用 XHR / Fetch API 请求后端接口
- 完全不使用 `<form action="">` 这种浏览器内置的提交表单的方式
- 完全不需要跨域

那么：

1. 前端请求时带上一个**自定义的请求头**，例如 `X-CSRF-PROTECTION: 1`。_头的值不重要。_

```ts
fetch('/your/sweet/api', {
  method: 'POST',
  body: '{"genshin":"impact"}',
  headers: {
    'X-CSRF-PROTECTION': '1',
  },
});
```

2. 服务端收到请求时，检查是否存在这个**自定义的请求头**。如果存在则放行，否则不放行。_头的值不重要。_

```ts
// 比如说你用 Koa 的话
app.use(async (ctx, next) => {
  if (!ctx.get('x-csrf-protection')) {
    ctx.status = 400;
    ctx.body = {
      message: 'You may be a victim of CSRF.',
    };

    return;
  }

  await next();
});
```

完事！

## 如果我需要跨域怎么办

你需要这样配置 CORS 头：

- `Access-Control-Allow-Origin`: 你的前端页面的 origin，例如 `https://homesweethome.com`。**不可以设置为 `*`。**
- `Access-Control-Allow-Headers`: 你的自定义请求头 key，例如 `X-CSRF-PROTECTION`。**不可以设置为 `*`。**

目的在于，不可以让来自其他网站的请求可以带上你的自定义请求头。所以必须限制只有来自你的前端页面的请求可以带上自定义请求头。

---

最近被安全同事拿着 Burp Suite 说我的服务有 CSRF 漏洞。

我心想嘿这怎么可能，这种业界已经一大把通用方案的东西，我们直接一个 `npm install csurf` 还能出什么差错？

结果进 [csurf 的 GitHub 仓库]一看，居然 deprecated 了：

> This npm module is currently deprecated due to **the large influx of security vulunerability reports received**, most of which are simply exploiting the underlying limitations of CSRF itself. The Express.js project does not have the resources to put into this module, which is largely unnecessary for modern SPA-based applications.

所以连这个周下载量最高峰时接近 60w 的 CSRF 中间件都有问题？

这时突然意识到，自己对于 CSRF 攻击如何利用以及防御其实也只有浅浅的了解——大概也许或许就是别人构造一个隐藏表单 POST 过来就可以在用户不知情的情况下发起转账，而要防的话好像似乎直接 cookies 跟 input/header 搭配一下就完事儿了。那，CSRF 具体是怎么一回事儿呢？2023 年了，是更容易攻击还是更容易防御了？听说 Chrome 早就已经 SameSite=Lax by default 了，对 CSRF 防御有啥影响不？

于是狠狠地花了一晚上的时间研究了一下。

## 如何利用

> Cross-Site Request Forgery (CSRF) is a type of attack that occurs when a malicious web site, email, blog, instant message, or program **causes a user's web browser** to perform an unwanted action on a trusted site when the user is authenticated.

反正就是通过各种手段，在用户不知情的情况下，让用户的浏览器往咱们的网站发了个请求，并且这个请求（往往）带有用户的登录态。

这会造成什么情况呢？比如经典例子，如果银行转账的 API 接口是

```
GET https://www.teyvatbank.com/elemental-transactions/paimonSpecial?amount=2568Mora&currency=GenesisCrystals&payee=AdventurersGuild
```

那我只需要构造一个网页，里边带上这个链接：

```html
<a href="https://www.teyvatbank.com/elemental-transactions/paimonSpecial?amount=2568Mora&currency=GenesisCrystals&payee=AdventurersGuild">点击查看 OpenAI 员工联名信</a>
```

然后把这个网页放到随便一个我可以控制的网站（成本相当低廉，不需要黑掉 www.teyvatbank.com 喔），通过社工手段诱导用户点击并在浏览器内打开——嚯，用户的 2568 Mora 就被转走了！

之所以如此，是因为浏览器（往往）在发送请求时会带上对应站点的 Cookies，且 Cookies 内带有用户的登录态，于是网站就收到了一个看起来很正常的请求，并执行了对应的操作。就算这个请求实际上是从别的网站发起的。

这只是一个简单的例子。接下来我们尝试站在攻击者的角度，看看如何能够发动此类攻击。_暂时忽略 SameSite 的存在，哈哈。_

### GET

如果目标接口是一个 GET 接口，那可以利用的方式数不胜数。

```html
<!-- 经典 a 标签 -->
<a id="adventurers" href="https://www.teyvatbank.com/elemental-transactions/paimonSpecial?amount=2568Mora&currency=GenesisCrystals&payee=AdventurersGuild">关注星瞳_Official</a>
<!-- 甚至我可以自动点击？ -->
<script>document.getElementById('adventurers').click()</script>

<!-- img 图片静默加载 -->
<img src="https://www.teyvatbank.com/elemental-transactions/paimonSpecial?amount=2568Mora&currency=GenesisCrystals&payee=AdventurersGuild">

<!-- 让用户提交一个 form 表单 -->
<form action="https://www.teyvatbank.com/elemental-transactions/paimonSpecial?amount=2568Mora&currency=GenesisCrystals&payee=AdventurersGuild">
  <input type="submit" value="送溜溜梅" />
</form>

<!-- 不演了，直接 JavaScript 发请求吧 -->
<script>
const endpoint = 'https://www.teyvatbank.com/elemental-transactions/paimonSpecial?amount=2568Mora&currency=GenesisCrystals&payee=AdventurersGuild';
fetch(endpoint, {
  method: 'GET',
  mode: 'no-cors',
  // 由于这是一个 simple request，所以不会触发 preflight
  // 无论 Access-Controls-Allow-Credentials 如何设置都会直接请求喔
  // @see https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple_requests
  credentials: 'include',
});
</script>

<!-- 等你来补充。 -->
```

所以，对于**会改变状态**的接口，千万不要用 GET。这也不符合 GET 的语义。

### POST w/ `x-www-form-urlencoded`

假设我们的接口吸取教训，变成了人见人爱的 POST 接口：

```
POST https://www.teyvatbank.com/elemental-transactions/paimonSpecial
Content-Type: application/x-www-form-urlencoded

amount=2568Mora&currency=GenesisCrystals&payee=AdventurersGuild
```

那可以利用的方式就少一些了。像 `<a>` 标签、`<img>` 标签等都无法直接发起 POST 请求。如果没有其他漏洞，通常可供利用的方式就只有 `<form>` 和 XHR / Fetch API 请求。

依葫芦画瓢，我们可以这样构造一个 `<form>` 表单：

```html
<form method="post" action="https://www.teyvatbank.com/elemental-transactions/paimonSpecial">
  <input type="hidden" name="amount" value="2568Mora" />
  <input type="hidden" name="currency" value="GenesisCrystals" />
  <input type="hidden" name="payee" value="AdventurersGuild" />
  <input type="submit" value="大的要来了" />
</form>
<!-- 我也可以自动点击？ -->
<script>document.querySelector('form').click()</script>
```

通过 XHR / Fetch API 也是一样的：

```ts
const endpoint = 'https://www.teyvatbank.com/elemental-transactions/paimonSpecial';
fetch(endpoint, {
  method: 'POST',
  mode: 'no-cors',
  body: 'amount=2568Mora&currency=GenesisCrystals&payee=AdventurersGuild',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  // 即使是 POST，即使设置了 Headers，这仍然是一个 simple request 捏
  // 复习一下：https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple_requests
  credentials: 'include',
});
```

## Reference

- https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- https://security.stackexchange.com/questions/265447/can-we-set-a-custom-csrf-token-inside-header-requests-to-make-a-csrf-poc
- https://web.dev/articles/samesite-cookies-explained
- https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS