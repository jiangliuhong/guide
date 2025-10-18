---
title: java合并图片
categories: 
    - Java
date: 2020-03-25 20:26:00
tags:
    - tools
---

# 使用java合并图片

## 编写`Bimg`类

> 省略`get`、`set`方法

```java
import java.awt.image.BufferedImage;

public class Bimg {
    private BufferedImage image;
    private int w;
    private int y;
}

```

## 编写测试类

```java

import java.awt.*;
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.FileInputStream;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;

import javax.imageio.ImageIO;

public class Test {

    private static List<String> getImgs() {
        List<String> array = new ArrayList<>();
        array.add("D:\\temp\\signimgs\\img\\1.png");
        array.add("D:\\temp\\signimgs\\img\\2.png");
        array.add("D:\\temp\\signimgs\\img\\3.png");
        array.add("D:\\temp\\signimgs\\img\\4.png");
        return array;
    }

    public static void main(String[] args) throws Exception {
        // 横向图片数量
        int wsize = 3;
        List<String> imgs = getImgs();
        int allWidth = 0;
        int allHeight = 0;
        List<Bimg> bimgs = new ArrayList<>();
        // 目前横坐标
        int w = 0;
        // 目前竖坐标
        int y = 0;
        // 当前行宽度
        int currentRowWidth = 0;
        // 当前行高度
        int currentRowHeight = 0;
        for (int i = 1; i <= imgs.size(); i++) {
            String imgPath = imgs.get(i - 1);
            File imgFile = new File(imgPath);
            InputStream in = new FileInputStream(imgFile);
            BufferedImage imageBuffer = ImageIO.read(in);
            Bimg bimg = new Bimg();
            bimgs.add(bimg);
            bimg.setImage(imageBuffer);
            // 判断是否需要换行
            if (i % (wsize + 1) == 0) {
                // 需要换行
                w = 0;
                y = currentRowHeight;
                allWidth = Math.max(allWidth, currentRowWidth);
                allHeight = allHeight + currentRowHeight;
                currentRowWidth = 0;
                currentRowHeight = 0;
            }
            bimg.setW(w);
            bimg.setY(y);
            w = w + imageBuffer.getWidth();
            currentRowHeight = Math.max(currentRowHeight, imageBuffer.getHeight());
            // 维护当前行宽度
            currentRowWidth = currentRowWidth + imageBuffer.getWidth();
        }
        // 如果最终的高宽都是0，就进行一次设置，主要为仅一行的情况做处理
        if (currentRowWidth != 0 || currentRowHeight != 0) {
            allWidth = Math.max(allWidth, currentRowWidth);
            allHeight = allHeight + currentRowHeight;
        }

        // 解决透明区域变黑的问题
        BufferedImage combined = new BufferedImage(allWidth, allHeight, BufferedImage.TYPE_INT_RGB);
        Graphics2D g = combined.createGraphics();
        combined = g.getDeviceConfiguration().createCompatibleImage(combined.getWidth(), combined.getHeight(),
            Transparency.TRANSLUCENT);
        // 解决透明区域变黑的问题
        Graphics graphics = combined.getGraphics();
        for (Bimg bimg : bimgs) {
            graphics.drawImage(bimg.getImage(), bimg.getW(), bimg.getY(), null);
        }
        ImageIO.write(combined, "png", new File("D:\\temp\\signimgs\\img\\res.png"));
    }

}
```

其中需要注意的是，如果源图片存在透明区域，直接合并的话，透明区域会变为黑色，该情况使用下面的代码就能解决：

```java
BufferedImage combined = new BufferedImage(allWidth, allHeight, BufferedImage.TYPE_INT_RGB);
        Graphics2D g = combined.createGraphics();
        combined = g.getDeviceConfiguration().createCompatibleImage(combined.getWidth(), combined.getHeight(),
            Transparency.TRANSLUCENT);
```