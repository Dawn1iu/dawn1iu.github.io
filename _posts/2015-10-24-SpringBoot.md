---
    author: Dawn1iu
    date: 2015-10-24
    layout: post
    title: SpringBoot相关整理
    tags:
		- spring
---
## SpringBoot相关整理
### 1、SpringApplication 执行过程
#### 1）静态run方法
![1508987987757.jpg](http://upload-images.jianshu.io/upload_images/7557064-df03191e3a049863.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1.创建 SpringApplication 实例  
2.根据classPath里是否存在```{ "javax.servlet.Servlet","org.springframework.web.context.ConfigurableWebApplicationContext" }```来判断是否创建web应用或者一个Standalone的ApplicationContext  
3.使用SpringFactoriesLoader 在应用 classpath 中查找并加载所有