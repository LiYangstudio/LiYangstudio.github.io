
<h1>问题介绍</h1>
最近工作室非常忙，忙着招新，也忙着其他事情。（学习OkHttp）很多东西都没有积累下来。 所以打算还是要坚持写文章。实际上OkHttp最近理解又多了不少。 但是迫于时间不够，没有时间积累下来。
<!-- more -->
最近也要开始重构我们工作室的项目，里面的代码有点感人，所以很难下手。 所以还是需要谨慎开始，但最近在思考的是一个问题：

有一些接口是要登录之后才能获取数据的，不然会被返回500.(这里的接口设计严重有问题，但也没办法，也是遗留代码)。 所以Android需要在一段时间了进行轮询，不断向服务器重复请求某个接口，保持App用户在线，以便恢复的时候，还能正常请求数据。

这会导致什么后果？ 
1.Service造成不必要的内存开销，而且不能保证100%存活，而且增加耗电。
2.不断请求数据，会导致用户流量增加，而且也会增加耗电。
3.用户量增大，导致后台压力太大。
4......

简直可以无限吐槽，经过一番思考，我想出了以下方法。

<h1>可能的解决方案</h1>
<h2>1.修改后台代码</h2>
据了解，后台是有一个统一拦截器，用来判断这个用户是否登录。 所以我打算在请求头或者Cookie里面，带上我们的用户名和密码。（但是这样不安全，容易被拦截）

然后如果Session过期，则进行模拟登录，重新分配，然后再返回实际接口数据。

但和后台沟通了之后，发现好像他们后台没有直接操作Cookie的概念。（因为框架已经搞定了），所以这个我认为最可行的方案就不能了。

<h2>2.请求数据前，先登录</h2>
这个方法也不是特别好。 就是在请求数据之前，每次都先登录一次（可以从本地拿到账号密码先）。 这个方法，我觉得就算再不好。 也比原来项目的一直开启一个轮询计时器好。

发现这三种都不行。

<h2>3.结合RxJava操作符retryWhen</h2>
RxJava有一个操作符：retryWhen，就是专门用来处理这种情况的。 可以在流式处理中，当接口数据请求失败的时候，然后执行retryWhen里面的登录，然后再次请求接口数据。

如果会RxJava，这无疑是妥妥的方案。 但是，由于团队合作的小伙伴，并没有学习RxJava，不利于团队合作，所以这种方案又只能放弃了！

<h1>终于找到你：OkHttp拦截器Interceptor</h1>
OkHttp不愧是年度最佳请求库。 它有一个特点就是，拦截器Interceptor。

Interceptor是什么？这里我就不展开来讲，这个不是本文的重点。
有兴趣可以参考：
[学习OkHttp wiki--Interceptors](http://www.tuicool.com/articles/Uf6bAnz)

[OKHttp源码解析（一）](http://blog.csdn.net/chenzujie/article/details/47061095)

不过我简单放出一个图：
![官方Wiki图](http://img.blog.csdn.net/20160329235014319)

发现官方的Wiki里面有说到，Application Interceptor和NetWorkInterceptor

他们两者有什么区别，应该仔细看一下图就能理解了，如果不能理解还是看看刚刚上面的两篇文章~

而我现在就是在NetWorkInterceptor那里，当NetWork返回数据：Response的时候，我判断一下放回的状态的码。如果是500，我就先new一个登录的Request，登录成功之后，再继续进行原来Request的请求。

大致代码如下:
```
/**
 * Created by Hans on 2016/3/29.
 * 重新登陆的拦截器
 * <p/>
 * 主要用在当用户SESSIONID过期，防止掉线而造成数据获取失败
 */
public abstract class ReLoginIntercetor implements Interceptor {
 @Override
    public Response intercept(Chain chain) throws IOException {
        Request originalRequest = chain.request();//原始接口请求
        Response originalResponse = chain.proceed(originalRequest);//原始接口结果
        if (originalResponse.code() == 500) {//如果是500，说明Session过期
            originalResponse.body().close();//关闭 释放原有的资源
            Request loginRequest = getLoginRequest();//获取登陆的Request
            Response loginResponse = chain.proceed(loginRequest);//执行登陆，获取新的SessionId
            if (loginResponse.isSuccessful()) {//登陆成功
                loginResponse.body().close();//释放登陆成功的资源
                return chain.proceed(originalRequest);//重新执行
            }
        }
        return originalResponse;
    }

    public abstract Request getLoginRequest();
}

```
这里简单写了一下可能的代码，这个方案应该是本次项目最可行的！

<h1>不足</h1>
最后解决方案的不足是什么？

因为工作室还有一个项目，大体情况也是这样！ 但是他的登陆接口是需要验证码的！！！

要是登陆的时候需要验证码，那这个方法就行不通了！ 因为请求的参数需要带上验证码，还是需要用户手动登陆。

所以，没有办法完美解决所以问题。但是这个方法，在本次方案是可以实行的，而且效果杠杠的！

要是网上各位前辈还有什么解决方案，可以留言指导，一起交流一下！


