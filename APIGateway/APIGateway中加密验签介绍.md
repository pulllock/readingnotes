需要提供给接口调用方一个用来加密的key，调用方根据key、一些其他参数以及业务参数进行加密，还需要对报文进行签名，使用加密的参数请求接口。

APIGateway接收到请求后进行验签，再进行解密，得到参数后进行处理。

处理完成需要按照跟请求接口一样的方式将结果进行加密加签，然后将结果返回给调用方，调用方按照同样的方式进行验签解密拿到结果。

方案：摘要签名、对称加密

1. kv参数排序，按照key自然排序，ASCII升序，并将参数值URLEncode一下。
2. 在参数中加入两个额外参数timestamp和nonce。timestamp表示请求发送时间，APIGateway可以对比当前时间戳，来判断请求是否正常；nonce表示一个随机数，也就是常说的盐值，APIGateway可以通过一定时间段内判断nonce是否重复来判断请求是否正常。
3. 使用给定的key，将参数进行加密，使用AES或者DES等。
4. 再将所有参数进行签名，MD5或者SHA1等

示例代码：

```java
public class CodecClient {

    public static final String HMAC_SHA1 = "HmacSHA1";

    public static final String AES = "AES";

    public static final String AES_CBC = "AES/CBC/PKCS5Padding";


    public static String encrypt(String key, String content) {
        try {
            SecretKeySpec secretKeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), AES);
            Cipher cipher = Cipher.getInstance(AES_CBC);
            IvParameterSpec iv = new IvParameterSpec(new byte[16]);
            cipher.init(Cipher.ENCRYPT_MODE, secretKeySpec, iv);
            return bytesToHexString(cipher.doFinal(content.getBytes(StandardCharsets.UTF_8)));
        } catch (NoSuchPaddingException | NoSuchAlgorithmException | InvalidKeyException | BadPaddingException | IllegalBlockSizeException | InvalidAlgorithmParameterException e) {
            throw new RuntimeException("Encrypt data error!");
        }
    }

    public static String decrypt(String key, String content) {
        try {
            SecretKeySpec secretKeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), AES);
            Cipher cipher = Cipher.getInstance(AES_CBC);
            IvParameterSpec iv = new IvParameterSpec(new byte[16]);
            cipher.init(Cipher.DECRYPT_MODE, secretKeySpec, iv);
            return new String(cipher.doFinal(hexStringToBytes(content)));
        } catch (NoSuchAlgorithmException | InvalidKeyException | InvalidAlgorithmParameterException | NoSuchPaddingException | BadPaddingException | IllegalBlockSizeException e) {
            e.printStackTrace();
            throw new RuntimeException("Decrypt data error!");
        }
    }

    public static String sign(String key, String data, String nonce) {
        try {
            SecretKeySpec secretKeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), HMAC_SHA1);
            Mac mac = Mac.getInstance(HMAC_SHA1);
            mac.init(secretKeySpec);
            mac.update(data.getBytes(StandardCharsets.UTF_8));
            return bytesToHexString(mac.doFinal(nonce.getBytes(StandardCharsets.UTF_8)));
        } catch (NoSuchAlgorithmException | InvalidKeyException e) {
            throw new RuntimeException("Sign error!");
        }
    }

    private static String bytesToHexString(byte[] bytes) {
        StringBuilder builder = new StringBuilder(bytes.length * 2);
        for (byte data : bytes) {
            builder.append(String.format("%02x", data & 0xff));
        }
        return builder.toString();
    }

    private static byte[] hexStringToBytes(String str) {
        byte[] bytes = new byte[str.length() / 2];
        for(int i = 0; i < str.length() / 2; i++) {
            String subStr = str.substring(i * 2, i * 2 + 2);
            bytes[i] = (byte) Integer.parseInt(subStr, 16);
        }

        return bytes;
    }

    public static void main(String[] args) throws UnsupportedEncodingException {
        String key = "xxxhhhsshkjkkddd";
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("id", 123);
        jsonObject.put("name", "测试姓名");
        jsonObject.put("alias", "tom");

        String param = URLEncoder.encode(jsonObject.toJSONString(), "utf-8");
        System.out.println("param: " + param);

        int nonce = new SecureRandom().nextInt();
        String data = String.format("apiCode=%s&data=%s&nonce=%d", "createUser#XDFFDDD", param, nonce);
        System.out.println("data: " + data);

        data = encrypt(key, data);
        System.out.println("encrypt data: " + data);

        String sign = sign(key, data, nonce + "");
        System.out.println("sign: " + sign);

        System.out.println("key: " + key);
        String decryptData = decrypt(key, data);
        System.out.println("DecryptData: " + decryptData);
    }
}
```

