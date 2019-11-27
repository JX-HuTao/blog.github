---
title: spring-security加解密源码分析之BCrypt(一)
categories: [spring]
tags: [源码, 加解密]
date: 2019-11-27 19:59:29
---
# 环境
```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-core</artifactId>
    <version>5.2.1.RELEASE</version>
    <scope>compile</scope>
</dependency>
```
# 概述
PasswordEncoder 是spring-security-core 提供的接口，该接口提供了三个方法：
```java
String encode(CharSequence rawPassword);

boolean matches(CharSequence rawPassword, String encodedPassword);

default boolean upgradeEncoding(String encodedPassword) {
    return false;
}
```
分别用于生成密码、校验密码、判断密码是否需要再次加密以获得更安全的密码，其中第三个方法较4.x.x 版本为新增方法，不考虑。以下开始讨论其中一个实现类 BCryptPasswordEncoder。
# BCryptPasswordEncoder 分析
BCryptPasswordEncoder 提供了 7 个构造方法，较4.x 版本主要增加参数 BCryptVersion，该参数是个枚举类，参与密码盐的生成。

该类的加密方法如下：
```java
public String encode(CharSequence rawPassword) {
    String salt;
    if (random != null) {
        salt = BCrypt.gensalt(version.getVersion(), strength, random);
    } else {
        salt = BCrypt.gensalt(version.getVersion(), strength);
    }
    return BCrypt.hashpw(rawPassword.toString(), salt);
}
```
上述可知，该方法首先生成密码盐，然后通过盐值生成密码。其中生成盐时若未指定 SecureRandom，则会调用无参构造实例化一个 SecureRandom。以下开始 BCrypt。
# BCrypt
```java
public static String gensalt(String prefix, int log_rounds)
        throws IllegalArgumentException {
    return gensalt(prefix, log_rounds, new SecureRandom());
}

private static final int BCRYPT_SALT_LEN = 16;

public static String gensalt(String prefix, int log_rounds, SecureRandom random)
        throws IllegalArgumentException {
    StringBuilder rs = new StringBuilder();
    byte rnd[] = new byte[BCRYPT_SALT_LEN];

    if (!prefix.startsWith("$2") ||
            (prefix.charAt(2) != 'a' && prefix.charAt(2) != 'y' &&
                    prefix.charAt(2) != 'b')) {
        throw new IllegalArgumentException ("Invalid prefix");
    }
    if (log_rounds < 4 || log_rounds > 31) {
        throw new IllegalArgumentException ("Invalid log_rounds");
    }

    random.nextBytes(rnd);

    rs.append("$2");
    rs.append(prefix.charAt(2));
    rs.append("$");
    if (log_rounds < 10)
        rs.append("0");
    rs.append(log_rounds);
    rs.append("$");
    encode_base64(rnd, rnd.length, rs);
    return rs.toString();
}
```
该方法较简单，前置判断校验参数正确性，encode_base64 之前为拼接盐前缀，大致格式为`$2a$10$`，这些参数通过 `$` 进行分割，分别表示version、strength；
```java
static void encode_base64(byte d[], int len, StringBuilder rs)
        throws IllegalArgumentException {
    int off = 0;
    int c1, c2;

    if (len <= 0 || len > d.length) {
        throw new IllegalArgumentException("Invalid len");
    }

    while (off < len) {
        c1 = d[off++] & 0xff;
        rs.append(base64_code[(c1 >> 2) & 0x3f]);
        c1 = (c1 & 0x03) << 4;
        if (off >= len) {
            rs.append(base64_code[c1 & 0x3f]);
            break;
        }
        c2 = d[off++] & 0xff;
        c1 |= (c2 >> 4) & 0x0f;
        rs.append(base64_code[c1 & 0x3f]);
        c1 = (c2 & 0x0f) << 2;
        if (off >= len) {
            rs.append(base64_code[c1 & 0x3f]);
            break;
        }
        c2 = d[off++] & 0xff;
        c1 |= (c2 >> 6) & 0x03;
        rs.append(base64_code[c1 & 0x3f]);
        rs.append(base64_code[c2 & 0x3f]);
    }
}
```
encode_base64 主要处理功能位于 while 方法，该方法内对 off 参数进行了三次自增操作，因此只需要考虑三种条件：len 分别为 1、2、3 时生成的结果。
```text
len = 1，生成两个字符，分别为当前 byte 的高6 bit、低 6 bit;

len = 2，生成三个字符，分别为 b0 的高6 bit、b0 的低2 bit及b1 的高4 bit、b1 的低6 bit;

len = 3，生成四个字符，三个byte 刚好24 bit，每个字符占6 bit；
```
base64_code 是一个用final修饰长度为64 的char[], 由生成的盐可以反推 SecureRandom 生成的byte数组，反之亦可；
# 密码生成
待补充