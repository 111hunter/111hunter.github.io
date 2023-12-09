# RBAC0 权限设计实例


权限管理是所有后台系统的都会涉及的一个重要组成部分，主要目的是对不同的人访问资源进行权限的控制，避免因权限控制缺失或操作不当引发的风险问题，如操作错误，隐私数据泄露等问题。

迄今为止最为普及的权限设计模型是 RBAC 模型,基于角色的访问控制（Role-Based Access Control)，而 RBAC 模型又可以细分为 RBAC0，RBAC1，RBAC2，RBAC3。

本文介绍 RBAC0, 这是权限最基础也是最核心的模型，其他复杂模型也是建立在 RBAC0 之上的。关于 RBAC 还有很多重要理论，具体可参考知乎上的这篇 [权限系统设计](https://zhuanlan.zhihu.com/p/73414693) 和这篇 [RBAC 理念](https://zhuanlan.zhihu.com/p/63775995)。本文将带领读者体会 RBAC0 的实践运用，实现 RBAC0 的关键在建立 **用户-角色-权限** 之间的多对多关系。

![RBAC权限模型](/img/rbac-permission.jpg "RBAC权限模型")

## 实例

请注意，本文不涉及具体代码讲解。如需具体代码的讲解，请移步到 [后端代码讲解](https://bezkoder.com/node-js-jwt-authentication-mysql/) 和 [前端代码讲解](https://bezkoder.com/react-hooks-jwt-auth/)，在这两篇文章末尾附有源码地址。作者的讲解逻辑严密，注重细节，非常优秀，无需我再赘述。本文只演示实例程序，带领读者理解 RBAC0 权限设计模型。

实例程序将网站用户分为三个角色: Admin(管理员), Moderator(版主), User(普通用户)。

所有页面路由：home, rigister, login, profile, user, mod, admin

### 正常访问截图

对所有未登录用户开放的页面(访客页面): home, register, login

![访客权限展示的页面](/img/rbac-public.png "访客权限展示的页面")

对网站用户开放的页面：

- 对 User 开放的页面(用户页面)：访客页面, profile, user

![用户权限展示的页面](/img/rbac-profile.png "用户权限展示的页面")

- 对 Moderator 开放的页面：用户页面, mod(导航栏中增加 Moderator Board)

- 对 Admin 开放的页面：用户页面, admin(导航栏中增加 Admin Board)

正常访问其他页面的更多截图看 [这里](https://bezkoder.com/react-hooks-jwt-auth/)，或者自己运行前后端代码，修改用户角色需用 postman 向后端接口发送 http 请求或者直接修改数据表。

### 越权访问截图

未登录用户访问 user 页面：

![未登录用户访问user页面](/img/rbac-!user.png "未登录用户访问user页面")

User 访问 admin 页面：

![登录用户访问admin页面](/img/rbac-!admin.png "登录用户访问admin页面")

User, Admin 访问 mod 页面, Moderator 访问 admin 页面的显示结果同理。

当一个用户同时具有 User, Moderator, Admin 角色时，就有了所有页面的访问权限。

![访问所有页面的权限](/img/rbac-root.png "访问所有页面的权限")

根据用户角色来决定页面的数据，这样就实现了 RBAC0 的基本模型。

**参阅资料**

- [可能是史上最全的权限系统设计](https://zhuanlan.zhihu.com/p/73414693)
- [RBAC 理念](https://zhuanlan.zhihu.com/p/63775995)
- [实例程序讲解](https://bezkoder.com/react-express-authentication-jwt/)

<!-- <div align=center><img src="/img/node-rbac.png" width="60%"></div> -->

