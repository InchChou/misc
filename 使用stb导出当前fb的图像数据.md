## stb

stb是c/c++的一系列轻量级库，其特点是每个库只有一个头文件。

项目地址为：[nothings/stb: stb single-file public domain libraries for C/C++ (github.com)](https://github.com/nothings/stb)

## 导出当前fb图像

### `stb_image_write.h`

为了导出图像，需要使用stb的一个库：`stb_image_write.h`，它可以将图像写入磁盘，支持的格式有PNG、TGA、BMP、JPG。

### `glReadPixels`

`glReadPixels`可以从当前绑定的framebuffer中读取一块pixels，其API形式如下：

```c++
void glReadPixels(	GLint x,
 	GLint y,
 	GLsizei width,
 	GLsizei height,
 	GLenum format,
 	GLenum type,
 	void * data);
```

参数说明见：[glReadPixels - OpenGL ES 3.2 Reference Pages (khronos.org)](https://www.khronos.org/registry/OpenGL-Refpages/es3/html/glReadPixels.xhtml)

> 其中需要注意的是，`format`参数和`type`参数。`type`最好使用`GL_UNSIGNED_BYTE`。而在GLES中，`format`只支持`GL_RGBA`，所以如果要读取的数据不是`GL_RGBA`，需要将其blit到`GL_RGBA`的framebuffer中。

`glReadPixels`通常和`glReadBuffer`一起使用，后者可以指定读取当前绑定framebuffer的某个Attachment。

## 示例程序

```c++
static void DumpFB()
{
    static unsigned long loopCount = 0;
    if (loopCount < 90) {
        ++loopCount;
        return;
    } else {
        loopCount = 0;
    }

    // 获取需要读取的fb的大小
    GLsizei width, height;
    int color = 0;
    glGetFramebufferAttachmentParameteriv(GL_FRAMEBUFFER,
        GL_COLOR_ATTACHMENT0, GL_FRAMEBUFFER_ATTACHMENT_OBJECT_NAME, &color);
    if (color != 0) {
        GLint curTexture = 1;
        glGetIntegerv(GL_TEXTURE_BINDING_2D, &curTexture);
        glBindTexture(GL_TEXTURE_2D, color);
        glGetTexLevelParameteriv(GL_TEXTURE_2D, 0, GL_TEXTURE_WIDTH, &width);
        glGetTexLevelParameteriv(GL_TEXTURE_2D, 0, GL_TEXTURE_HEIGHT, &height);
        glBindTexture(GL_TEXTURE_2D, curTexture);
    }

    GLint srcFramebuffer;
    glGetIntegerv(GL_DRAW_FRAMEBUFFER_BINDING, &srcFramebuffer);

    // 生成附件格式为GL_RGBA的新fb
    static GLuint dstFramebuffer = 0;
    static GLuint newAttachment = 0;
    if (dstFramebuffer == 0) {
        glGenFramebuffers(1, &dstFramebuffer);
        glBindFramebuffer(GL_FRAMEBUFFER, dstFramebuffer);
        glGenTextures(1, &newAttachment);
        glBindTexture(GL_TEXTURE_2D, newAttachment);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_BASE_LEVEL, GL_ZERO);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAX_LEVEL, GL_ZERO);
        glTexStorage2D(GL_TEXTURE_2D, 1, GL_RGBA, width, height);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_SRGB_DECODE_EXT, GL_DECODE_EXT);
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, newAttachment, 0);
    }

    // 将源fb的附件数据blit到新fb中
    glBindFramebuffer(GL_READ_FRAMEBUFFER, srcFramebuffer);
    glBindFramebuffer(GL_DRAW_FRAMEBUFFER, dstFramebuffer);
    glBlitFramebuffer(0, 0, width, height, 0, 0, width, height, GL_COLOR_BUFFER_BIT, GL_LINEAR);

    // 设置读取的Attachment，读取pixels
    glReadBuffer(GL_COLOR_ATTACHMENT0);
    static GLubyte* pixels = NULL;
    if (pixels == NULL) {
        pixels = (GLubyte*)malloc(width * height * 4);
    }
    glReadPixels(0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, pixels);

    // 设置文件路径，使用stbi写入文件
    static const std::string path = "/sdcard/Android/data/yuanshenDump/";
    static unsigned long imgCount = 0;
    std::string fileName = std::to_string(imgCount);
    fileName = path + fileName + ".png";
    // int ret = stbi_write_bmp(fileName.c_str(), width, height, 4, pixels);
    // int ret = stbi_write_jpg(fileName.c_str(), width, height, 4, pixels, 100);
    int ret = stbi_write_png(fileName.c_str(), width, height, 4, pixels, 4 * width);
    if (ret == 0) {
        LOGE("dump failed");
    } else {
        LOGI("dump success");
    }
    imgCount++;
    // free(pixels);
}
```

