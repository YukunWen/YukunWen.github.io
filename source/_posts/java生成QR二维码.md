---
title: '"生成QR二维码"' 
categories: java 
tags: java 
date: 2018-10-27 11:44:29 
---

####引言
   随着二维码（QR code）的普及，越来越多的项目中会设计一些产生二维码的交互页面，以便更好地和用户互动，以及方便用户的使用，传播app等操作。那么今天就来探究一下如何在项目中快速简单的生成QR二维码。

##一.简易版本
###首先
我们用到谷歌开源的zxing项目包，使用maven的同学可以轻易的导入。
```
       <!--QRcode-->
        <dependency>
            <groupId>com.google.zxing</groupId>
            <artifactId>core</artifactId>
            <version>3.1.0</version>
        </dependency>
        <dependency>
            <groupId>com.google.zxing</groupId>
            <artifactId>javase</artifactId>
            <version>3.1.0</version>
        </dependency>
        <!--QRcode-->
```
<!--more-->
###第二步
我们来编写生成简单二维码的java代码。只要有一个content的输入，能把content的内容藏到二维码中即可。那么代码如下：
``` java
public class QRCodeUtil {

    private static final String CHARSET = "utf-8";

    // 二维码尺寸
    private static final int QRCODE_SIZE = 300;

    public static BufferedImage createImage(String content) {
        Hashtable<EncodeHintType, Object> hints = new Hashtable<EncodeHintType, Object>();
        hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
        hints.put(EncodeHintType.CHARACTER_SET, CHARSET);
        hints.put(EncodeHintType.MARGIN, 1);
        BitMatrix bitMatrix = null;
        try {
            bitMatrix = new MultiFormatWriter().encode(content, BarcodeFormat.QR_CODE, QRCODE_SIZE, QRCODE_SIZE,
                    hints);
        } catch (WriterException e) {
            e.printStackTrace();
        }
        int width = bitMatrix.getWidth();
        int height = bitMatrix.getHeight();
        BufferedImage image = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
        for (int x = 0; x < width; x++) {
            for (int y = 0; y < height; y++) {
                image.setRGB(x, y, bitMatrix.get(x, y) ? 0xFF000000 : 0xFFFFFFFF);
            }
        }
        return image;
    }
}
```
下面我们编写controller接口去进行测试
> **注意**：对于有些二维码是扫了即用，扫完即走那种的，这个时候就不应该产生本地的缓存暂用服务器资源。禁止缓存的方式详见代码。
```java
@Controller
@RequestMapping("/qr")
@Slf4j
public class QRController {

    @GetMapping(value = "/generate")
    @ResponseBody
    public void generateQR(@RequestParam("content")String content, HttpServletResponse response) {
        BufferedImage image;
        // 禁止图像缓存
        response.setHeader("Pragma", "No-cache");
        response.setHeader("Cache-Control", "no-cache");
        response.setDateHeader("Expires", 0);
        response.setContentType("image/jpeg");

        image = QRCodeUtil.createImage(content);
        // 创建二进制的输出流
        try(ServletOutputStream sos = response.getOutputStream()){
            // 将图像输出到Servlet输出流中。
            ImageIO.write(image, "jpeg", sos);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
###结果
最终效果如下图1所示：
![图1.简易版本QR码生成图](https://upload-images.jianshu.io/upload_images/14043408-d97d2b4718eeb260.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##二.拓展版本
在实际中，我们常常会看见一些二维码中间会放置上企业的logo，显得更加的美观专业。接下来我们就如何在二维码中间增加图案进行说明。
###首先
我们考虑增加两个变量，一个是插入图片的地址，一个压缩参数（主要用于过大logo情况下是否压缩）
```java
   // LOGO宽度
    private static final int LOGO_WIDTH = 80;
    // LOGO高度
    private static final int LOGO_HEIGHT = 80;
    /**
     * 插入LOGO
     *
     * @param source       二维码图片
     * @param logoPath     LOGO图片地址
     * @param needCompress 是否压缩
     * @throws Exception
     */
    public static void insertImage(BufferedImage source, InputStream logoPath, boolean needCompress)  {
        Image src = null;
        try {
            src = ImageIO.read(logoPath);
        } catch (IOException e) {
            e.printStackTrace();
        }
        int width = src.getWidth(null);
        int height = src.getHeight(null);
        if (needCompress) {
            // 压缩LOGO
            if (width > LOGO_WIDTH) {
                width = LOGO_WIDTH;
            }
            if (height > LOGO_HEIGHT) {
                height = LOGO_HEIGHT;
            }
            Image image = src.getScaledInstance(width, height, Image.SCALE_SMOOTH);
            BufferedImage tag = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
            Graphics g = tag.getGraphics();
            // 绘制缩小后的图
            g.drawImage(image, 0, 0, null);
            g.dispose();
            src = image;
        }
        // 插入LOGO
        Graphics2D graph = source.createGraphics();
        int x = (QRCODE_SIZE - width) / 2;
        int y = (QRCODE_SIZE - height) / 2;
        graph.drawImage(src, x, y, width, height, null);
        Shape shape = new RoundRectangle2D.Float(x, y, width, width, 12, 12);
        graph.setStroke(new BasicStroke(3f));
        graph.draw(shape);
        graph.dispose();
    }
```
我们只要在一中生成图片的基础上调用该方法，即可在里面插入一个Logo。
###接着
我们照样编写了controller去测试：
![图2.新增代码示意图](https://upload-images.jianshu.io/upload_images/14043408-a4bc18aa7a3e0d88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###结果
最终我们就可以得到带有logo的二维码了，如下图3所示：
![图3.拓展版本QR码生成图](https://upload-images.jianshu.io/upload_images/14043408-6c76d629f6fb91d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##三.总结
有了谷歌的帮助，我们生成二维码是非常简单的一件事情。最后，博主有几点想提醒读者：
1. 二维码的原理主要是依靠斜左上方的三个矩形框来进行定位，然后解析图片的黑白像素对应计算机编码的01操作。那么如果是二维码里面藏的东西过多的时候，二维码可能会很丑陋，几个定位符非常的小，里面的黑白点非常的密集。这个时候不妨尝试一下利用缓存，二维码里面只藏有简单的随机字符串，然后再根据扫描得到的字符串去请求缓存拿到真正的有用信息。(这种就是**代理**的思想)
2. 在第1点的基础上，有时候二维码就是为了做登陆的，要求极高的安全性。在二维码里面藏了一下检验，防串改的字串，比如jwtToken。那么这个二维码很有可能会非常丑，遇到了不懂技术的产品，可能要你改需求。低效的沟通还不如直接拿巨头的成本给他看，直接拿微信公众平台(https://mp.weixin.qq.com/)的登陆页面二维码给他看，让对方明白**为了安全有的时候不得不牺牲美观**。
![微信公众平台登陆页面二维码.png](https://upload-images.jianshu.io/upload_images/14043408-65d3375bfacb8b26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3.关于中间藏匿的logo，读者可能需要根据logo的大小和二维码的复杂程度去自行调节一下。觉得本文写得还行的读者可以扫描文中postman生成的二维码关注我的github。

本文内容代码均放置到了github上，需要的读者自行获取：
> https://github.com/YukunWen/QRdemo
