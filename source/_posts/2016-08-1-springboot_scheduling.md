---
layout: post
title: "Spring Boot配置定时任务以及注册启动类"
description: ""
tag: "SpringBoot"
date: "2016-08-14 00:00:00 +0800"
categories: "java"
---

#### 配置Spring Boot定时任务

在Spring Boot启动方法上加上`@EnableScheduling`开启定时任务

<!--more--> 


```
@SpringBootApplication
@EnableScheduling
@EnableAutoConfiguration
@ComponentScan(basePackages = "com.smtcl.machinetool")
public class Main extends SpringBootServletInitializer{

	private static Class<Main> applicationClass = Main.class;

	/*build spring boot*/
	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder builder){

		return builder.sources(applicationClass);
	}

	/*spring boot 启动主方法*/
	public static void main(String[] args){

		SpringApplication.run(applicationClass, args);
	}
}
```

#### 开启定时任务 
```
	@Scheduled(fixedRate = 10000)
	public void updateOemTool(){

		//		List<Name> list = idao.executeQuery("from Name");
		//		for (Name name : list){
		//
		//			System.out.println(name.getName());
		//		}

	}
```

fixedRate=10000表示每十秒执行一次该方法，如果需要更精确的定时可以用  

`cron`表达式  

```
"0 0 12 * * ?"    每天中午十二点触发 
"0 15 10 ? * *"    每天早上10：15触发 
"0 15 10 * * ?"    每天早上10：15触发 
"0 15 10 * * ? *"    每天早上10：15触发 
"0 15 10 * * ? 2005"    2005年的每天早上10：15触发 
"0 * 14 * * ?"    每天从下午2点开始到2点59分每分钟一次触发 
"0 0/5 14 * * ?"    每天从下午2点开始到2：55分结束每5分钟一次触发 
"0 0/5 14,18 * * ?"    每天的下午2点至2：55和6点至6点55分两个时间段内每5分钟一次触发 
"0 0-5 14 * * ?"    每天14:00至14:05每分钟一次触发 
"0 10,44 14 ? 3 WED"    三月的每周三的14：10和14：44触发 
"0 15 10 ? * MON-FRI"    每个周一、周二、周三、周四、周五的10：15触发 
```

#### 配置随项目启动启动

在`application.propertites`中添加如下配置  
```
context.listener.classes=你要注册的类的路径
```

你要注册的类只要继承`ApplicationListener<ContextRefreshedEvent>`并且实现`onApplicationEvent`方法就可以了，这时项目启动的时候就会自动运行`onApplicationEvent`中的服务。  

```
public class RFIDTCPService implements ApplicationListener<ContextRefreshedEvent>{

	public void onApplicationEvent(ContextRefreshedEvent event){

		/**
		 * 获取服务端端口
		 */
		Integer port = (Integer) event.getApplicationContext().getBean("SERVER_PORT");
		new Thread(new waitConnection(port)).start();
	}

}
```