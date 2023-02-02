---
layout:     post
title:      SpringBoot AOP 记录操作日志、异常日志
subtitle:   SpringBoot AOP 记录操作日志、异常日志
date:       2022-02-02
author:     BY
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- Springboot
---
平时我们在做项目时经常需要对一些重要功能操作记录日志，方便以后跟踪是谁在操作此功能；我们在操作某些功能时也有可能会发生异常，但是每次发生异常要定位原因我们都要到服务器去查询日志才能找到，而且也不能对发生的异常进行统计，从而改进我们的项目，要是能做个功能专门来记录操作日志和异常日志那就好了。<br />当然我们肯定有方法来做这件事情，而且也不会很难，我们可以在需要的方法中增加记录日志的代码，和在每个方法中增加记录异常的代码，最终把记录的日志存到数据库中。听起来好像很容易，但是我们做起来会发现，做这项工作很繁琐，而且都是在做一些重复性工作，还增加大量冗余代码，这种方式记录日志肯定是不可行的。<br />我们以前学过Spring 三大特性，IOC（控制反转），DI（依赖注入），AOP（面向切面），那其中AOP的主要功能就是将日志记录，性能统计，安全控制，事务处理，异常处理等代码从业务逻辑代码中划分出来。今天我们就来用springBoot Aop 来做日志记录，好了，废话说了一大堆还是上货吧。
<a name="tgNCs"></a>
### 一、创建日志记录表、异常日志表，表结构如下：
操作日志表<br />![](https://cdn.nlark.com/yuque/0/2022/png/958674/1665370402396-8e00c1a1-683b-49a2-b01a-f2c8c96a53c9.png#clientId=u72558335-f86f-4&from=paste&id=u062eaa9b&originHeight=355&originWidth=880&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u38876c13-2da4-4671-b1f3-5a680dab0dc&title=)<br />异常日志表<br />![](https://cdn.nlark.com/yuque/0/2022/png/958674/1665370402370-c5e35c9e-ff69-42b4-8cec-ae5e28aa111d.png#clientId=u72558335-f86f-4&from=paste&id=ucd430f32&originHeight=298&originWidth=878&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=ua8369aa9-3de1-4169-a3ea-a474b85fe3f&title=)
<a name="eUll3"></a>
### 二、添加Maven依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
<a name="DunTP"></a>
### 三、创建操作日志注解类OperLog.java
```
package com.hyd.zcar.cms.common.utils.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 自定义操作日志注解
 * @author wu
 */
@Target(ElementType.METHOD) //注解放置的目标位置,METHOD是可注解在方法级别上
@Retention(RetentionPolicy.RUNTIME) //注解在哪个阶段执行
@Documented
public @interface OperLog {
    String operModul() default ""; // 操作模块
    String operType() default "";  // 操作类型
    String operDesc() default "";  // 操作说明
}
```
<a name="Xam2u"></a>
### 四、创建切面类记录操作日志
```
package com.hyd.zcar.cms.common.utils.aop;

import java.lang.reflect.Method;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestAttributes;
import org.springframework.web.context.request.RequestContextHolder;

import com.gexin.fastjson.JSON;
import com.hyd.zcar.cms.common.utils.IPUtil;
import com.hyd.zcar.cms.common.utils.annotation.OperLog;
import com.hyd.zcar.cms.common.utils.base.UuidUtil;
import com.hyd.zcar.cms.common.utils.security.UserShiroUtil;
import com.hyd.zcar.cms.entity.system.log.ExceptionLog;
import com.hyd.zcar.cms.entity.system.log.OperationLog;
import com.hyd.zcar.cms.service.system.log.ExceptionLogService;
import com.hyd.zcar.cms.service.system.log.OperationLogService;

/**
 * 切面处理类，操作日志异常日志记录处理
 *
 * @author wu
 * @date 2019/03/21
 */
@Aspect
@Component
public class OperLogAspect {

    /**
     * 操作版本号
     * <p>
     * 项目启动时从命令行传入，例如：java -jar xxx.war --version=201902
     * </p>
     */
    @Value("${version}")
    private String operVer;

    @Autowired
    private OperationLogService operationLogService;

    @Autowired
    private ExceptionLogService exceptionLogService;

    /**
     * 设置操作日志切入点 记录操作日志 在注解的位置切入代码
     */
    @Pointcut("@annotation(com.hyd.zcar.cms.common.utils.annotation.OperLog)")
    public void operLogPoinCut() {
    }

    /**
     * 设置操作异常切入点记录异常日志 扫描所有controller包下操作
     */
    @Pointcut("execution(* com.hyd.zcar.cms.controller..*.*(..))")
    public void operExceptionLogPoinCut() {
    }

    /**
     * 正常返回通知，拦截用户操作日志，连接点正常执行完成后执行， 如果连接点抛出异常，则不会执行
     *
     * @param joinPoint 切入点
     * @param keys      返回结果
     */
    @AfterReturning(value = "operLogPoinCut()", returning = "keys")
    public void saveOperLog(JoinPoint joinPoint, Object keys) {
        // 获取RequestAttributes
        RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
        // 从获取RequestAttributes中获取HttpServletRequest的信息
        HttpServletRequest request = (HttpServletRequest) requestAttributes
                .resolveReference(RequestAttributes.REFERENCE_REQUEST);

        OperationLog operlog = new OperationLog();
        try {
            operlog.setOperId(UuidUtil.get32UUID()); // 主键ID

            // 从切面织入点处通过反射机制获取织入点处的方法
            MethodSignature signature = (MethodSignature) joinPoint.getSignature();
            // 获取切入点所在的方法
            Method method = signature.getMethod();
            // 获取操作
            OperLog opLog = method.getAnnotation(OperLog.class);
            if (opLog != null) {
                String operModul = opLog.operModul();
                String operType = opLog.operType();
                String operDesc = opLog.operDesc();
                operlog.setOperModul(operModul); // 操作模块
                operlog.setOperType(operType); // 操作类型
                operlog.setOperDesc(operDesc); // 操作描述
            }
            // 获取请求的类名
            String className = joinPoint.getTarget().getClass().getName();
            // 获取请求的方法名
            String methodName = method.getName();
            methodName = className + "." + methodName;

            operlog.setOperMethod(methodName); // 请求方法

            // 请求的参数
            Map<String, String> rtnMap = converMap(request.getParameterMap());
            // 将参数所在的数组转换成json
            String params = JSON.toJSONString(rtnMap);

            operlog.setOperRequParam(params); // 请求参数
            operlog.setOperRespParam(JSON.toJSONString(keys)); // 返回结果
            operlog.setOperUserId(UserShiroUtil.getCurrentUserLoginName()); // 请求用户ID
            operlog.setOperUserName(UserShiroUtil.getCurrentUserName()); // 请求用户名称
            operlog.setOperIp(IPUtil.getRemortIP(request)); // 请求IP
            operlog.setOperUri(request.getRequestURI()); // 请求URI
            operlog.setOperCreateTime(new Date()); // 创建时间
            operlog.setOperVer(operVer); // 操作版本
            operationLogService.insert(operlog);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 异常返回通知，用于拦截异常日志信息 连接点抛出异常后执行
     *
     * @param joinPoint 切入点
     * @param e         异常信息
     */
    @AfterThrowing(pointcut = "operExceptionLogPoinCut()", throwing = "e")
    public void saveExceptionLog(JoinPoint joinPoint, Throwable e) {
        // 获取RequestAttributes
        RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
        // 从获取RequestAttributes中获取HttpServletRequest的信息
        HttpServletRequest request = (HttpServletRequest) requestAttributes
                .resolveReference(RequestAttributes.REFERENCE_REQUEST);

        ExceptionLog excepLog = new ExceptionLog();
        try {
            // 从切面织入点处通过反射机制获取织入点处的方法
            MethodSignature signature = (MethodSignature) joinPoint.getSignature();
            // 获取切入点所在的方法
            Method method = signature.getMethod();
            excepLog.setExcId(UuidUtil.get32UUID());
            // 获取请求的类名
            String className = joinPoint.getTarget().getClass().getName();
            // 获取请求的方法名
            String methodName = method.getName();
            methodName = className + "." + methodName;
            // 请求的参数
            Map<String, String> rtnMap = converMap(request.getParameterMap());
            // 将参数所在的数组转换成json
            String params = JSON.toJSONString(rtnMap);
            excepLog.setExcRequParam(params); // 请求参数
            excepLog.setOperMethod(methodName); // 请求方法名
            excepLog.setExcName(e.getClass().getName()); // 异常名称
            excepLog.setExcMessage(stackTraceToString(e.getClass().getName(), e.getMessage(), e.getStackTrace())); // 异常信息
            excepLog.setOperUserId(UserShiroUtil.getCurrentUserLoginName()); // 操作员ID
            excepLog.setOperUserName(UserShiroUtil.getCurrentUserName()); // 操作员名称
            excepLog.setOperUri(request.getRequestURI()); // 操作URI
            excepLog.setOperIp(IPUtil.getRemortIP(request)); // 操作员IP
            excepLog.setOperVer(operVer); // 操作版本号
            excepLog.setOperCreateTime(new Date()); // 发生异常时间

            exceptionLogService.insert(excepLog);

        } catch (Exception e2) {
            e2.printStackTrace();
        }

    }

    /**
     * 转换request 请求参数
     *
     * @param paramMap request获取的参数数组
     */
    public Map<String, String> converMap(Map<String, String[]> paramMap) {
        Map<String, String> rtnMap = new HashMap<String, String>();
        for (String key : paramMap.keySet()) {
            rtnMap.put(key, paramMap.get(key)[0]);
        }
        return rtnMap;
    }

    /**
     * 转换异常信息为字符串
     *
     * @param exceptionName    异常名称
     * @param exceptionMessage 异常信息
     * @param elements         堆栈信息
     */
    public String stackTraceToString(String exceptionName, String exceptionMessage, StackTraceElement[] elements) {
        StringBuffer strbuff = new StringBuffer();
        for (StackTraceElement stet : elements) {
            strbuff.append(stet + "\n");
        }
        String message = exceptionName + ":" + exceptionMessage + "\n\t" + strbuff.toString();
        return message;
    }
}
```
<a name="rFlsV"></a>
### 五、在Controller层方法添加@OperLog注解
![](https://cdn.nlark.com/yuque/0/2022/png/958674/1665370402426-14868a7f-f667-49af-ba19-c4873e142f31.png#clientId=u72558335-f86f-4&from=paste&id=ua6eb6e54&originHeight=450&originWidth=1080&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=ud6777b86-0ae3-4a5d-b1ed-a70861f08d2&title=)
<a name="hfImP"></a>
### 六、操作日志、异常日志查询功能
![](https://cdn.nlark.com/yuque/0/2022/png/958674/1665370402428-5065b2b8-f1c0-45e4-b4ea-bedee653fc90.png#clientId=u72558335-f86f-4&from=paste&id=u1dbb7283&originHeight=539&originWidth=1080&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u4648c434-6049-4a36-bd76-d44b9535d81&title=)<br />![](https://cdn.nlark.com/yuque/0/2022/png/958674/1665370402389-5947762a-942b-4421-a3e0-b65303d920f8.png#clientId=u72558335-f86f-4&from=paste&id=ue45b9f65&originHeight=645&originWidth=1080&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u75fa6ef2-1548-4121-9c51-0a17c03bfce&title=)<br />![](https://cdn.nlark.com/yuque/0/2022/png/958674/1665370403616-36736ec8-8f1b-4e2f-adfe-d8abd1be18c6.png#clientId=u72558335-f86f-4&from=paste&id=u4bb1e0de&originHeight=540&originWidth=1080&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u7738716a-aba5-41ee-91b7-148cee4690e&title=)<br />![](https://cdn.nlark.com/yuque/0/2022/png/958674/1665370403107-e082ad97-529d-479e-9c55-4fbf31036949.png#clientId=u72558335-f86f-4&from=paste&id=ua7db9ddf&originHeight=525&originWidth=1080&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u7f81401e-d805-4b9a-8966-f3a660a76a0&title=)<br />![](https://cdn.nlark.com/yuque/0/2022/png/958674/1665370403029-565b6306-2ad8-444f-8747-5a29be1a3bb9.png#clientId=u72558335-f86f-4&from=paste&id=u5747de9c&originHeight=480&originWidth=1080&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=ub89d08bd-cb04-4fd1-84b5-7196541e975&title=)
