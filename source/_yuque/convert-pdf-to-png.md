---
title: 065-使用Java将pdf转换并拼接成一张图片
urlname: convert-pdf-to-png
date: 2021-09-09 19:35:21 +0800
tags: [java,pdf,png,tools,utils]
categories: [java]
---

> 这是坚持技术写作计划（含翻译）的第 65 篇，定个小目标 999，每周最少 2 篇。

本文 主要讲解通过 java 将 pdf 转换成一张图片(基于[apache pdfbox](https://pdfbox.apache.org/)),需要注意本方法不适用于页数特别多的情况(容易 OOM),如果页码多的情况，还是建议生成多张图片

<!-- more -->

​

依赖库 参考 [https://pdfbox.apache.org/3.0/migration.html#dependency-updates](https://pdfbox.apache.org/3.0/migration.html#dependency-updates)

```xml

        <dependency>
            <groupId>org.apache.pdfbox</groupId>
            <artifactId>pdfbox</artifactId>
            <version>3.0.0-RC1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.pdfbox</groupId>
            <artifactId>pdfbox-tools</artifactId>
            <version>3.0.0-RC1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.pdfbox</groupId>
            <artifactId>fontbox</artifactId>
            <version>3.0.0-RC1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.pdfbox</groupId>
            <artifactId>jbig2-imageio</artifactId>
            <version>3.0.3</version>
        </dependency>
        <dependency>
            <groupId>com.github.jai-imageio</groupId>
            <artifactId>jai-imageio-core</artifactId>
            <version>1.4.0</version>
        </dependency>
        <dependency>
            <groupId>com.github.jai-imageio</groupId>
            <artifactId>jai-imageio-jpeg2000</artifactId>
            <version>1.4.0</version>
        </dependency>
```

```java
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.FileUtils;
import org.apache.pdfbox.Loader;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.rendering.ImageType;
import org.apache.pdfbox.rendering.PDFRenderer;
import org.apache.pdfbox.tools.imageio.ImageIOUtil;
import org.junit.Test;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.IOException;
import java.net.URL;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

/**
 * pdf 工具类
 *
 * @author AnJia
 * @since 2021-09-09 16:05
 */
@Slf4j
public class PdfUtils {

    @Test
    public void convertPdf2ImageTest() throws IOException {
        // pdf转换成单张图片
        convertPdf2Image("https://file-examples-com.github.io/uploads/2017/10/file-sample_150kB.pdf", System.getProperty("java.io.tmpdir") + "anjia", "sample.png");
        // pdf转换成多张
        convertPdf2Images("https://file-examples-com.github.io/uploads/2017/10/file-sample_150kB.pdf", System.getProperty("java.io.tmpdir") + "anjia", "sample.png");
    }

    /**
     * 将pdf转成一张图片,如果要输出的文件已经存在，则不会进行转换
     *
     * @param url      pdf url
     * @param fileName 文件名带png后缀，例如 xxxx.png
     * @param dir      存放文件夹，如果不存在会自动创建
     * @return 文件，如果报错，则会返回null
     * @throws IOException 文件下载失败
     */
    public File convertPdf2Image(String url, String dir, String fileName) throws IOException {
        File pdfFile = new File(new File(dir).getAbsolutePath() + File.separator + fileName + ".pdf");
        pdfFile.getParentFile().mkdirs();
        FileUtils.copyURLToFile(new URL(url), pdfFile);
        return convertPdf2Image(pdfFile, dir, fileName);
    }

    /**
     * 将pdf转成一张图片,如果要输出的文件已经存在，则不会进行转换
     *
     * @param pdfFile  pdf 文件
     * @param fileName 文件名带png后缀，例如 xxxx.png
     * @param dir      存放文件夹，如果不存在会自动创建
     * @return 文件，如果报错，则会返回null
     */
    public File convertPdf2Image(File pdfFile, String dir, String fileName) {
        File pngFile = new File(new File(dir).getAbsolutePath() + File.separator + fileName);
        if (pngFile.exists()) {
            return pngFile;
        }
        try (final PDDocument document = Loader.loadPDF(pdfFile)) {
            PDFRenderer pdfRenderer = new PDFRenderer(document);
            // 不知道图片的宽和高，所以先定义个null
            BufferedImage pdfImage = null;
            // pdf有多少页
            int pageSize = document.getNumberOfPages();
            int y = 0;
            for (int i = 0; i < pageSize; ++i) {
                // 每页pdf内容
                BufferedImage bim = pdfRenderer.renderImageWithDPI(i, 300, ImageType.RGB);
                // 如果是第一页需要初始化 BufferedImage
                if (Objects.isNull(pdfImage)) {
                    // 假设每页一样宽，一样高，高度就是每页高度*总页数
                    pdfImage = new BufferedImage(bim.getWidth(),
                        bim.getHeight() * pageSize, BufferedImage.TYPE_INT_ARGB);
                }
                // 将每页pdf画到总的pdfImage上,x坐标=0，y坐标=之前所有页的高度和
                pdfImage.getGraphics().drawImage(bim, 0, y, null);
                y += bim.getHeight();
            }

            assert pdfImage != null;
            ImageIO.write(pdfImage, "png", pngFile);
            return pngFile;
        } catch (Exception ex) {
            log.error("pdf转换png失败", ex);
        }
        return null;
    }

    /**
     * 将pdf转换成多张图片(1页pdf转换成1张图片),如果要输出的图片已经存在，则不会进行转换
     *
     * @param url          pdf url
     * @param dir          存放目录
     * @param baseFileName pdf文件名
     * @return 如果转换成功会返回 图片文件list，如果失败会返回null
     * @throws IOException 下载文件失败
     */
    public List<File> convertPdf2Images(String url, String dir, String baseFileName) throws IOException {
        File pdfFile = new File(dir + File.separator + baseFileName + ".pdf");
        pdfFile.getParentFile().mkdirs();
        FileUtils.copyURLToFile(new URL(url), pdfFile);
        return convertPdf2Images(pdfFile, dir, baseFileName);
    }

    /**
     * 将pdf转换成多张图片(1页pdf转换成1张图片),如果要输出的图片已经存在，则不会进行转换
     *
     * @param pdfFile      pdf 文件
     * @param dir          存放目录
     * @param baseFileName pdf文件名
     * @return 如果转换成功会返回 图片文件list，如果失败会返回null
     */
    public List<File> convertPdf2Images(File pdfFile, String dir, String baseFileName) {
        List<File> images = null;
        File directory = new File(dir);
        directory.mkdirs();
        try (final PDDocument document = Loader.loadPDF(pdfFile)) {
            PDFRenderer pdfRenderer = new PDFRenderer(document);
            // pdf有多少页
            int pageSize = document.getNumberOfPages();
            images = new ArrayList<>(pageSize);
            File pageFile;
            for (int i = 0; i < pageSize; ++i) {
                pageFile = new File(String.format("%s%s%s-%s.png", directory.getAbsolutePath(), File.separator, baseFileName, i));
                if (pageFile.exists()) {
                    continue;
                }
                BufferedImage bim = pdfRenderer.renderImageWithDPI(i, 300, ImageType.RGB);
                ImageIOUtil.writeImage(bim, pageFile.getAbsolutePath(), 300);
                images.add(pageFile);
            }
        } catch (Exception ex) {
            log.error("pdf转换png失败", ex);
        }
        return images;

    }
}

```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/09/09/convert-pdf-to-png/)
- [我的掘金](https://juejin.cn/post/7006119643953758221/)
- [Convert PDF Files to PNG, JPEG, BMP, and TIFF Images using Java](https://blog.aspose.com/2021/02/12/convert-pdf-to-png-jpeg-bmp-tiff-using-java/)
- [pdfbox document >> Migration to PDFBox 2.0.0#pdf-rendering](https://pdfbox.apache.org/2.0/migration.html#pdf-rendering)
