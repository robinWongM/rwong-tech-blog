---
title: "在 2023 年，如何防止 CSRF 攻击"
description: "It's simple"
date: 2023-11-21T00:00:00+08:00
---

## TL;DR

如果你的前端应用：

- 主要使用 XHR / Fetch API 与后端交互
- 不使用 `<form action="">` 这种浏览器内置的提交表单的方式
- 不需要跨域

那么：

1. 前端请求时带上一个自定义的请求头，例如 `X-CSRF-PROTECTION: 1`。_头的值不重要。_

```ts
fetch('/your/sweet/api', {
  method: 'POST',
  body: '{"genshin":"impact"}',
  headers: {
    'X-CSRF-PROTECTION': '1',
  },
});
```

2. 服务端收到请求时，检查是否存在这个自定义请求头。如果存在则放行，否则不放行。_头的值不重要。_

```ts
// 如果你在使用 Koa

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

That's it! 

## Reference

- https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- https://security.stackexchange.com/questions/265447/can-we-set-a-custom-csrf-token-inside-header-requests-to-make-a-csrf-poc