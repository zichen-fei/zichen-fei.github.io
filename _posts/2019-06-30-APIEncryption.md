---
layout: post
title: API数据加密
date: 2019-06-30
tags: 微服务
---

#### **使用AES和RSA混合加密**

两种加密算法特点：  

AES（对称加密）  
1、加密方和解密方使用同一个密钥  
2、加密解密的速度比较快，适合数据比较长时的使用  
3、密钥传输的过程不安全，且容易被破解，密钥管理也比较麻烦  

RSA（非对称加密）  
1、算法强度复杂，其安全性依赖于算法与密钥  
2、加密解密的速度远远低于对称加密算法，因此不适用于数据量较大的情况  
3、非对称加密算法有两种密钥，其中一个是公开的，所以在密钥传输上不存在安全性问题，使得其在传输加密数据的安全性上又高于对称加密算法  

**AES加密快，RSA安全性高，所以将两种算法混合使用，用AES加密数据，再用RSA加密AES使用的KEY**

API加密前后端交互的主要流程如下：  

1、后端生成密钥对，用户登录后，获取后端的公钥，并保存在当前session  
2、浏览器生成密钥对，随机生成用于AES加密的明文KEY，用AES算法加密请求参数，并用后端的公钥加密KEY，将加密后的请求参数、加密后的KEY、前端公钥传给后端
（1和2主要在前后端交换公钥，生成方式可随意，需要保证在交换之后保持不变）  
3、后端使用私钥解密出明文KEY（由前端使用后端公钥加密），再用KEY解密请求参数，返回数据之前，后端再随机生成用于AES加密的明文KEY，加密结果数据，用前端公钥加密KEY，将加密后的结果数据、加密后的KEY返回给前端  
4、前端得到数据后，用自己的私钥解密出明文KEY，再通过KEY解密数据  

示例：
```
package cn.bit.api.aspect;

import cn.bit.api.annotation.Decrypt;
import cn.bit.api.annotation.Encrypt;
import cn.bit.api.util.AesUtil;
import cn.bit.api.util.Constant;
import cn.bit.api.util.RsaUtil;
import cn.bit.bdp.common.response.ResponseBuilder;
import com.alibaba.fastjson.JSON;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.commons.codec.binary.Base64;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.text.SimpleDateFormat;
import java.util.HashMap;
import java.util.Map;

/**
 * AES + RSA 加解密AOP处理
 */
@Aspect
@Component
public class SafetyAspect {

    public static final Map<String, String> KEYS = new HashMap<>();

    @Pointcut(value = "execution(public * cn.bit.api.controller.*.*(..))")
    public void safetyAspect() {
    }

    /**
     * 环绕通知
     */
    @Around(value = "safetyAspect()")
    public Object around(ProceedingJoinPoint pjp) {
        try {
            ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            assert attributes != null;
            HttpServletRequest request = attributes.getRequest();

            String httpMethod = request.getMethod().toLowerCase();

            Method method = ((MethodSignature) pjp.getSignature()).getMethod();

            Annotation[] annotations = method.getAnnotations();

            Object[] args = pjp.getArgs();

            boolean hasDecrypt = false;
            boolean hasEncrypt = false;
            for (Annotation annotation : annotations) {
                if (annotation.annotationType() == Decrypt.class) {
                    hasDecrypt = true;
                }
                if (annotation.annotationType() == Encrypt.class) {
                    hasEncrypt = true;
                }
            }

            String frontPublicKey = "";

            ObjectMapper mapper = new ObjectMapper();
            mapper.setDateFormat( new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

            if ("post".equals(httpMethod) && hasDecrypt) {
                String backendPrivateKey = KEYS.get(Constant.PRIVATE_KEY);
                String data = request.getParameter(Constant.DATA);
                String aesKey = request.getParameter(Constant.AES_KEY);
                frontPublicKey = request.getParameter(Constant.PUBLIC_KEY);

                System.out.println("前端公钥：" + frontPublicKey);

                //后端私钥解密的到AES的key
                byte[] plaintext = RsaUtil.decryptByPrivateKey(Base64.decodeBase64(aesKey), backendPrivateKey);
                aesKey = new String(plaintext);
                System.out.println("解密出来的AES的key：" + aesKey);

                //RSA解密出来字符串多一对双引号
                aesKey = aesKey.substring(1, aesKey.length() - 1);

                //AES解密得到明文data数据
                String decrypt = AesUtil.decrypt(data, aesKey);
                System.out.println("解密出来的data数据：" + decrypt);

                //设置到方法的形参中，目前只能设置只有一个参数的情况
                mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

                args[0] = mapper.readValue(JSON.toJSONString(decrypt), args[0].getClass());
            }

            Object o = pjp.proceed(args);

            //返回结果之前加密
            if (hasEncrypt) {
                mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
                //每次响应之前随机获取AES的key，加密data数据
                String key = AesUtil.getKey();
                System.out.println("AES的key：" + key);
                String dataString = mapper.writeValueAsString(o);
                System.out.println("需要加密的data数据：" + dataString);
                String data = AesUtil.encrypt(dataString, key);

                //用前端的公钥来加密AES的key，并转成Base64
                String aesKey = Base64.encodeBase64String(RsaUtil.encryptByPublicKey(key.getBytes(), frontPublicKey));

                Map<String, String> map = new HashMap<>();
                map.put(Constant.DATA, data);
                map.put(Constant.AES_KEY, aesKey);
                System.out.println(JSON.toJSONString(map));
                return ResponseBuilder.ok(map);
            }

            return o;

        } catch (Throwable e) {
            System.err.println(pjp.getSignature());
            e.printStackTrace();
            return ResponseBuilder.error("加解密异常");
        }
    }
}

```
![](/images/posts/api-encryption/110.png)
<br>
![](/images/posts/api-encryption/111.png)
<br>
![](/images/posts/api-encryption/113.png)

