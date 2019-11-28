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
之前已经介绍了密文获取方式，密文主要由以下代码获取
```java
BCrypt.hashpw(rawPassword.toString(), salt);

public static String hashpw(String password, String salt) {
    byte passwordb[];

    passwordb = password.getBytes(StandardCharsets.UTF_8);

    return hashpw(passwordb, salt);
}

public static String hashpw(byte passwordb[], String salt) {
    BCrypt B;
    String real_salt;
    byte saltb[], hashed[];
    char minor = (char) 0;
    int rounds, off;
    StringBuilder rs = new StringBuilder();

    if (salt == null) {
        throw new IllegalArgumentException("salt cannot be null");
    }

    int saltLength = salt.length();

    if (saltLength < 28) {
        throw new IllegalArgumentException("Invalid salt");
    }

    /*
     * 计算偏移量
     * 若直接使用security提供的盐生成策略，则 off 必定为 4
     */
    if (salt.charAt(0) != '$' || salt.charAt(1) != '2')
        throw new IllegalArgumentException ("Invalid salt version");
    if (salt.charAt(2) == '$')
        off = 3;
    else {
        minor = salt.charAt(2);
        if ((minor != 'a' && minor != 'x' && minor != 'y' && minor != 'b')
                || salt.charAt(3) != '$')
            throw new IllegalArgumentException ("Invalid salt revision");
        off = 4;
    }

    // Extract number of rounds
    if (salt.charAt(off + 2) > '$')
        throw new IllegalArgumentException ("Missing salt rounds");

    if (off == 4 && saltLength < 29) {
        throw new IllegalArgumentException("Invalid salt");
    }
    rounds = Integer.parseInt(salt.substring(off, off + 2));

    // base64 解码，获取生成盐的 byte 数组
    real_salt = salt.substring(off + 3, off + 25);
    saltb = decode_base64(real_salt, BCRYPT_SALT_LEN);

    // 若直接使用security提供的盐生成策略, 则该逻辑必定为真
    if (minor >= 'a') // add null terminator
        passwordb = Arrays.copyOf(passwordb, passwordb.length + 1);

    B = new BCrypt();
    hashed = B.crypt_raw(passwordb, saltb, rounds, minor == 'x', minor == 'a' ? 0x10000 : 0);

    // 组装密文
    rs.append("$2");
    if (minor >= 'a')
        rs.append(minor);
    rs.append("$");
    if (rounds < 10)
        rs.append("0");
    rs.append(rounds);
    rs.append("$");
    encode_base64(saltb, saltb.length, rs); // 密文组装部分执行到此为 密码盐
    encode_base64(hashed, bf_crypt_ciphertext.length * 4 - 1, rs);
    return rs.toString();
}
```
上述代码若不追究 hashed 计算方式，只观察最后一段代码，前置部分与密码盐生成逻辑一样，若进行对比也可以发现密文首部与密码盐相同，只是由于盐由随机数生成，使得同一密码生成的密文不同。

密文尾部由 hashed 进行base64加密获得，其中 bf_crypt_ciphertext 是final修饰的byte 数组，长度为6，之前讨论密码盐生成时已经探讨该生成策略，可知最后 31 位字符为密码密文，因此可以通过使用相同密码尝试解析不同密文。

接下来开始研究 hashed 生成原理，其源码如下
```java
private byte[] crypt_raw(byte password[], byte salt[], int log_rounds,
                        boolean sign_ext_bug, int safety) {
    int rounds, i, j;
    int cdata[] =  bf_crypt_ciphertext.clone();
    int clen = cdata.length;
    byte ret[];

    if (log_rounds < 4 || log_rounds > 31)
        throw new IllegalArgumentException ("Bad number of rounds");
    rounds = 1 << log_rounds;
    if (salt.length != BCRYPT_SALT_LEN)
        throw new IllegalArgumentException ("Bad salt length");

    init_key();
    ekskey(salt, password, sign_ext_bug, safety);
    for (i = 0; i < rounds; i++) {
        key(password, sign_ext_bug, safety);
        key(salt, false, safety);
    }

    for (i = 0; i < 64; i++) {
        for (j = 0; j < (clen >> 1); j++)
            encipher(cdata, j << 1);
    }

    ret = new byte[clen * 4];
    for (i = 0, j = 0; i < clen; i++) {
        ret[j++] = (byte) ((cdata[i] >> 24) & 0xff);
        ret[j++] = (byte) ((cdata[i] >> 16) & 0xff);
        ret[j++] = (byte) ((cdata[i] >> 8) & 0xff);
        ret[j++] = (byte) (cdata[i] & 0xff);
    }
    return ret;
}
```

`init_key`部分，该部分并未执行特殊操作，准备运算数据。
```java
private int P[];
private int S[];

private void init_key() {
    P = P_orig.clone(); // P_orig 为长度 16 的数组
    S = S_orig.clone(); // S_orig 为长度 1024 的数组
}
```

`ekskey`部分，该部分功能较复杂，以下选取部分片段进行研究
```java
for (i = 0; i < plen; i++) {
    int words[] = streamtowords(key, koffp, signp);
    diff |= words[0] ^ words[1];
    P[i] = P[i] ^ words[sign_ext_bug ? 1 : 0];
}
```

```java
private static int[] streamtowords(byte data[], int offp[], int signp[]) {
    int i;
    int words[] = { 0, 0 };
    int off = offp[0];
    int sign = signp[0];

    for (i = 0; i < 4; i++) {
        words[0] = (words[0] << 8) | (data[off] & 0xff);
        words[1] = (words[1] << 8) | (int) data[off]; // sign extension bug
        if (i > 0) sign |= words[1] & 0x80;
        off = (off + 1) % data.length;
    }

    offp[0] = off;
    signp[0] = sign;
    return words;
}
```
`streamtowords`方法归纳结果如下：
1. words[0] 由 4 个 byte 组成 ；
2. words[1] 的值由最后一位负数及之后的byte组成，若不存在负数，则与 words[0] 相等；
3. off 递增数字，`streamtowords`执行一次，该值加 4，即 offp[0] 加4；
4. sign 若其值等于`0x80`，则 data 必定包含负数，不一定当前取值范围存在负数；

未完待续...