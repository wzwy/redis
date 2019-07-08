# redis


package com.wz.springboot.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.METHOD})//指定此注解可以放置的位置是方法上
@Retention(RetentionPolicy.RUNTIME)//指定此注解可以运行时期有效
public @interface TokenValidation {
}
package com.wz.springboot.cacheconstant;

public class CacheConstant {

    public static final String USER_TOKEN = "USER_TOKEN";

    public static final String USER_INFO = "USER_INFO";

}
package com.wz.springboot.commom;

import java.io.Serializable;

//公共用户token实体，如果想要实现，实体插入redis里必须实现序列化接口Serializable
public class BaseCommand implements Serializable {

    //用户token
    private String token;

    public String getToken() {
        return token;
    }

    public void setToken(String token) {
        this.token = token;
    }
}

package com.wz.springboot.commom;

import javax.validation.constraints.NotNull;

public class UserInfor extends BaseCommand {

    //校验用户名是否为空
    @NotNull(message = "用户名不能为空")
    private String name;

    //校验密码是否为空
    @NotNull(message = "密码不能为空")
    private String password;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}

package com.wz.springboot.configuration;

import org.springframework.cache.annotation.CacheConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.data.redis.core.RedisTemplate;

@CacheConfig
public class RedisConfiguration {

    @Bean
    public RedisTemplate redisTemplate(){
        return new RedisTemplate();
    }

}

package com.wz.springboot.configuration;

import com.alibaba.fastjson.JSONObject;
import com.wz.springboot.cacheconstant.CacheConstant;
import com.wz.springboot.commom.BaseCommand;
import com.wz.springboot.redis.RedisCache;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.context.annotation.Configuration;

import javax.annotation.Resource;

@Configuration//指定此类是配置文件
@Aspect//指定此类是aop切面类
public class TokenValidationInterceptor {

    @Resource(name = "redisCache")
    private RedisCache redisCache;

    //指定切入点，存在@TokenValidation注解的进行切入
    @Pointcut("@annotation(com.wz.springboot.annotation.TokenValidation)")
    public void tokenValidationAop(){

    }


    //指定切入点指定的逻辑
    @Around("tokenValidationAop()")
    public Object doAround(ProceedingJoinPoint pjp) throws Throwable {
        boolean vaildateFlg = Boolean.FALSE;
        JSONObject jsonObject = null;
        Object[] args = pjp.getArgs();
        if(args != null && args.length > 0){
            for (Object o : args){
                if(o instanceof BaseCommand){
                    BaseCommand baseCommand = (BaseCommand) redisCache.getHash(CacheConstant.USER_TOKEN + ((BaseCommand) o).getToken(),CacheConstant.USER_INFO);
                    if(baseCommand != null && !"".equals(baseCommand.getToken())){
//                        System.out.println("token:"+baseCommand.getToken());
                        vaildateFlg = Boolean.TRUE;
                        break;
                    }
                }
            }
        }
        if (!vaildateFlg) {
            jsonObject = new JSONObject();
            jsonObject.put("code","1001");
            jsonObject.put("msg","验证失败");
            return jsonObject;
        }
        return pjp.proceed(args);
    }

}

package com.wz.springboot.controller;

import com.alibaba.fastjson.JSONObject;
import com.wz.springboot.annotation.TokenValidation;
import com.wz.springboot.cacheconstant.CacheConstant;
import com.wz.springboot.commom.BaseCommand;
import com.wz.springboot.commom.UserInfor;
import com.wz.springboot.redis.RedisCache;
import org.springframework.validation.BindingResult;
import org.springframework.validation.ObjectError;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@RestController
public class UserController {

    @Resource(name = "redisCache")
    private RedisCache redisCache;

    @TokenValidation
    @PostMapping("/user")
    public JSONObject user(@RequestBody @Validated UserInfor userInfor , BindingResult bindingResult){
        JSONObject jsonObject = null;
        if(bindingResult.hasErrors()){
            jsonObject = new JSONObject();
            List<ObjectError> allErrors = bindingResult.getAllErrors();
            for (ObjectError objectError : allErrors) {
                // 输出错误信息
                System.out.println(objectError.getDefaultMessage());
                jsonObject.put("code","1001");
                jsonObject.put("msg","访问失败:"+objectError.getDefaultMessage());
            }
            return jsonObject;
        }
        jsonObject = new JSONObject();
        jsonObject.put("code","1000");
        jsonObject.put("msg","访问成功");
        return jsonObject;
    }

    @PostMapping("/insertToken")
    public JSONObject insertToken(){
        String token = this.saveUserInfo().getToken();
        BaseCommand baseCommand = new BaseCommand();
        baseCommand.setToken(token);
        redisCache.saveHash(CacheConstant.USER_TOKEN + token,CacheConstant.USER_INFO,baseCommand,180,TimeUnit.SECONDS);
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("code","1000");
        jsonObject.put("msg","插入Token成功");
        jsonObject.put("token",token);
        return jsonObject;
    }

    @GetMapping("/getToken")
    public JSONObject getToken(){
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("code","1000");
        jsonObject.put("msg","获取Token成功");
        jsonObject.put("token",this.saveUserInfo().getToken());
        return jsonObject;
    }

    public UserInfor saveUserInfo(){
        UserInfor userInfor = new UserInfor();
        userInfor.setToken(UUID.randomUUID().toString().replaceAll("-",""));
        return userInfor;
    }

}


package com.wz.springboot.redis;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.concurrent.TimeUnit;
//Redis配置文件类
@Component
public class RedisCache<K,V> {

    @Resource(name = "redisTemplate")
    private RedisTemplate redisTemplate;

    //添加hash类型数据存入Redis
    public void saveHash(K pkey,K key,V value,long liveTime,TimeUnit timeUnit){
//        BoundHashOperations boundHashOperations = redisTemplate.boundHashOps(pkey);
//        boundHashOperations.expire(liveTime,timeUnit);
//        boundHashOperations.put(key,value);
        redisTemplate.opsForHash().put(pkey,key,value);
        redisTemplate.expire(pkey,liveTime,timeUnit);

    }

    //获取存入Redis的hash类型数据
    public V getHash(K pkey,K key){
        return (V) redisTemplate.opsForHash().get(pkey,key);
    }

}


package com.wz.springboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication//springboot主启动类
public class SpringbootApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringbootApplication.class, args);
	}


}

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

	<!--springboot最终打的jar包-->
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.wz</groupId>
	<artifactId>springboot-redis-token</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>springboot-redis-token</name>
	<description>Spring Boot</description>

	<!--springboot父jar包-->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.3.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<!--jar版本-->
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<!--springmvc-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<!--测试-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

		<!--自动编译-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<optional>true</optional>
		</dependency>

		<!--redis-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>

		<!--fastjson-->
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>fastjson</artifactId>
			<version>1.2.56</version>
		</dependency>

		<!--aop-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-aop</artifactId>
		</dependency>


		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
		</dependency>


	</dependencies>

	<!--maven打包-->
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>


</project>

