> Web安全/Spring/Spring Security

> 在工作中，前端请求后端接口一直报403，排查了半天发现是因为Spring Security的CSRF保护逻辑导致的。由于之前在开发工作中对这块知识点的没有很关注，这里专门写一篇文章来总结一下。

# 什么是CSRF攻击

**跨站请求伪造**（Cross-site request forgery），也被称为**one-click attack**或者**session riding**，通常缩写为**CSRF**或者**XSRF**，是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。

这里我们分两种场景来介绍CSRF攻击的具体案例：

* Get请求

    假设一家银行的转账请求的请求地址是`GET http://bank.com/transfer?accountNo=1234&amount=100`，其中`acoountNo`是收款方账号，`amount`是转账金额。假设攻击者想将受害者的钱转到自己的账户，那么他就可以将`accountNo`改成5678自己的账户，比如`GET http://bank.com/transfer?accountNo=5678&amount=1000`。多种方式可以实现这种方案：

    * HTML链接

        ```html
        <a href="http://bank.com/transfer?accountNo=5678&amount=1000">
        打开链接
        </a>
        ```

        受害者点击该链接就会执行转账逻辑，从而中招，向攻击者账号转账。

    * HTML 图像

        ```html
        <img src="http://bank.com/transfer?accountNo=5678&amount=1000"/>
        ```

        攻击者可以使用 `<img/>` 标签将目标 URL 作为图片源。换句话说，甚至不需要点击。页面加载时会自动执行请求。

* Post请求

    我们将上面转账的请求变为一个POST请求：

    ```Java
    POST http://bank.com/transfer
    accountNo=1234&amount=100
    ```

# CSRF攻击的使用场景

# CSRF攻击的解决方案

# 总结

# 参考文章

* [【维基百科】跨站请求伪造](https://zh.wikipedia.org/wiki/%E8%B7%A8%E7%AB%99%E8%AF%B7%E6%B1%82%E4%BC%AA%E9%80%A0)
* [A Guide to CSRF Protection in Spring Security](https://www.baeldung.com/spring-security-csrf)
* [How to Solve 403 Error in Spring Boot POST Request](https://www.baeldung.com/java-spring-fix-403-error)[CSRF With Stateless REST API](https://www.baeldung.com/csrf-stateless-rest-api)
* [什么是 cookie 的 httponly 属性](https://cloud.tencent.com/developer/article/2297159)