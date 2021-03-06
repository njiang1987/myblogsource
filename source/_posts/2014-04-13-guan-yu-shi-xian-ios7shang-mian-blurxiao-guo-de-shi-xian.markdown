---
layout: post
title: "关于实现iOS7上面Blur效果的实现"
date: 2014-01-23 18:13:00 +0800
comments: true
categories:
- 技术
tags: [iOS]
---
前几个月iOS7发布，说实话界面扁平化，个人不是很喜欢，但是它的一些效果还是比较炫的，比如Blur效果，看下图 <br>
<img src="/uploads/2014-01-23/photo1_normal-169x300.png" alt="photo1_normal" width="169" height="300" class="alignnone size-medium wp-image-43" />
<img src="/uploads/2014-01-23/photo2_normal-169x300.png" alt="photo2_normal" width="169" height="300" class="alignnone size-medium wp-image-44" />

右边的图片，有个Blur效果，最近PM正想将这个效果添加到我们的产品中，查看网上说，苹果是通过直接从GPU里面取出图片信息，然后render出来的，也没有公用的API可以调用。不过有类似的方法可以模拟这个效果，比如Gaussian Blur，这个是通过CoreImage.framework里面的CIFilter实现的，具体实现如下：
{% codeblock lang:objc %}
CGImageRef imageRef = CGImageCreateWithImageInRect(image.CGImage, frame);
    
CIImage *imageToBlur = [CIImage imageWithCGImage:imageRef];
CIFilter *blurFilter = [CIFilter filterWithName:@"CIGaussianBlur"];
[blurFilter setValue:imageToBlur forKey:@"inputImage"];
[blurFilter setValue:[NSNumber numberWithFloat:0.1f] forKey:@"inputRadius"];
CIImage* result = [blurFilter valueForKey:@"outputImage"];
    
CIContext* context = [CIContext contextWithOptions:nil];
CGImageRef cgImage = [context createCGImage:result fromRect:[imageToBlur extent]];
    
UIImage* newImage = [UIImage imageWithCGImage:cgImage];
{% endcodeblock %}
这样就可以得到一个模糊的效果。但是这种模糊有2个缺点：
1.如果设置的半径比较大，效率会很低
2.模糊以后的效果不好

后来在网上查到可以用boxblur算法，这个会用到Accelerate.framework里面的一个函数vImageBoxConvolve_ARGB8888，具体实现如下：
{% codeblock lang:objc %}
- (UIImage*)imageWithBoxBlur:(CGFloat)blur withBoxSize:(int)iSize
{
    if (blur < 0.f || blur > 1.f) {
        blur = 0.5f;
    }
    int boxSize = (int)(blur * iSize);
    boxSize = boxSize - (boxSize % 2) + 1;

    CGImageRef img = self.CGImage;
    vImage_Buffer inBuffer, outBuffer;
    vImage_Error error;
    void *pixelBuffer;

    //create vImage_Buffer with data from CGImageRef
    CGDataProviderRef inProvider = CGImageGetDataProvider(img);
    CFDataRef inBitmapData = CGDataProviderCopyData(inProvider);

    inBuffer.width = CGImageGetWidth(img);
    inBuffer.height = CGImageGetHeight(img);
    inBuffer.rowBytes = CGImageGetBytesPerRow(img);

    inBuffer.data = (void*)CFDataGetBytePtr(inBitmapData);

    //create vImage_Buffer for output
    pixelBuffer = malloc(CGImageGetBytesPerRow(img) * CGImageGetHeight(img));

    if(pixelBuffer == NULL)
        NSLog(@"No pixelbuffer");

    outBuffer.data = pixelBuffer;
    outBuffer.width = CGImageGetWidth(img);
    outBuffer.height = CGImageGetHeight(img);
    outBuffer.rowBytes = CGImageGetBytesPerRow(img);

    //perform convolution
    error = vImageBoxConvolve_ARGB8888(&inBuffer, &outBuffer, NULL, 0, 0, boxSize, boxSize, NULL, kvImageEdgeExtend);

    if (error) {
        NSLog(@"error from convolution %ld", error);
    }

    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    CGContextRef ctx = CGBitmapContextCreate(outBuffer.data,
                                             outBuffer.width,
                                             outBuffer.height,
                                             8,
                                             outBuffer.rowBytes,
                                             colorSpace,
                                             (CGBitmapInfo)kCGImageAlphaNoneSkipLast);
    CGImageRef imageRef = CGBitmapContextCreateImage (ctx);
    UIImage *returnImage = [UIImage imageWithCGImage:imageRef];

    //clean up
    CGContextRelease(ctx);
    CGColorSpaceRelease(colorSpace);

    free(pixelBuffer);
    //free(pixelBuffer2);
    CFRelease(inBitmapData);

    CGColorSpaceRelease(colorSpace);
    CGImageRelease(imageRef);

    return returnImage;
}
{% endcodeblock %}
这个boxblur有个2个优点：<br>
1. 速度很快<br>
2. 模糊以后的效果很好<br>
正好弥补了Gaussian Blur的缺点，但是这个boxblur也有个致命的缺点就是，如果一张图片既有深色区域又有浅色区域，模糊以后，深色区域的颜色就不对了。过几天我再来补图。
后来无意中发现，如果先用Gaussian blur以后，再用boxblur几次，效果比较好。还是过几天来补图，呵呵，这几天比较忙，一直在做PM的Spec，虽然都是界面的东西，对我一个计算机半路出家的人来说还是优点难度的。以下是效果图，外加公司的产品。

<img src="/uploads/2014-01-23/iOS-Simulator-Screen-shot-Sep-4-2013-7.43.34-PM-300x225.png" alt="blur以后的效果图" width="300" height="225" class="size-medium wp-image-48" />