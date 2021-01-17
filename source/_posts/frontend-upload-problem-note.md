title: H5图片上传踩坑记录
date: 2019-01-06
categories:
- 笔记本
tags:
- JavaScript
language: zh-CN
toc: true
cover: /gallery/covers/upload-error.jpg
thumbnail: /gallery/covers/upload-error.jpg
---

最近写的一个项目中图片上传这个功能，运营同学反馈用户上传的图片出现了照片内容变黑错乱、上传照片大小为0字节等奇怪问题，下面就记录下处理问题的过程。
<!-- more -->

## 前言

我们常用H5上传图片方式，有以下几种：

1. 利用 *input* 和 *form* 表单进行文件上传，但是无法对图片进行压缩、旋转等优化处理。
2. 利用RenderFile的`readAsDataURL`转成base64进行上传，但是base64编码通常比所对应的图片二进制文件体积要更大，同时不支持切片上传。
3. 利用RenderFile或者 URL.createObjectURL将File对象转换为Blob，再利用canvas进行图片压缩、剪裁。最后转换为blob插入到FormData中传递给后端服务。

实际项目中我选择的是最后一种来实现文件上传：

> type类型为file 的input标签可以选择设备上的文件进行一个或者多个进行上传操作，具体属性可阅读[MDN-input](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/Input/file)

```vue
<input
  type="file"
  ref="input"
  :name="fileKey"
  capture="camera"
  accept="image/*"
  @change="someHander"
/>
```


### 坑点一：选择相同文件不触发input 的onchange事件

因为两次选择的文件一致，input的value值并没有发生改变，所以不会触发change事件，所以得到文件后清空value来避免这个问题。

```javascript
function someHander(event) {
    const file = e.target.files[0];
    // some action
    event.target.value = '';
}
```



### 坑点二：对得到File对象进行处理

目前对FIle对象处理常用的API有

- [URL.createObjectURL](https://developer.mozilla.org/zh-CN/docs/Web/API/URL/createObjectURL)：参数传递file或者blob对象，返回一个新的URL对象
  - 执行方式：同步函数立即执行
  - 内存消耗：因为URL的生命周期和document相关，只有在窗口关闭、执行[URL.revokeObjectURL](https://developer.mozilla.org/zh-CN/docs/Web/API/URL/revokeObjectURL)时销毁，所以不适用后应该**主动**去销毁改URL 对象
  - 兼容性：IE10后的所有现代浏览器
- [FileReader. readAsDataURL](https://developer.mozilla.org/en-US/docs/Web/API/FileReader/readAsDataURL): 参数传递file或者blob对象，返回所对应Base64内容
  - 执行方式：FileRender利用自身的`load`、`error`进行异步执行，需要绑定对应的事件得到执行的结果。
  - 内存消耗：因为Base64包含的都是文本内容，相对于blob占据更多的内存空间，但是存储base64的变量在不使用之后会被JavaScript的垃圾回收机制，自动回收。
  - 兼容性：IE10后的所有现代浏览器

所以综上所述，这里选择URL.createObjectURL进行File对象的读取。

```javascript
const blob = URL.createObjectURL(file);
let imgEl = new Image();
imgEl.src = blob;
// some action....
URL.revokeObjectURL(blob); // 记得释放URL对象
```



### 坑点三：图片方向旋转问题

在IOS设备拍照中，会出现照片在canvas绘制出现是横的，和手持拍照时的方向并不一致。为了解决这个问题，我们可以获取exif中一个**Orientation**的参数，得到照片旋转方向。详见可以阅读[七牛云-关于图片 EXIF 信息中旋转参数 Orientation 的理解](https://developer.qiniu.com/dora/kb/1560/information-about-photo-exif-rotation-parameters-in-the-understanding-of-orientation)

可以利用开源的[exif-js](https://github.com/exif-js/exif-js)插件得到照片的exif信息，从而在canva绘制正确展示的图片。



### 坑点四：canvas最大尺寸渲染

利用canvas绘制图片，应该控制下控制绘制的尺寸大小，因为绘制也是占用内存的嘛。太大的尺寸的图片绘制效率不高不说，低端的手机可能会导致浏览器直接奔溃、卡顿等问题。

```javascript
resizeImg(img: HTMLImageElement, orientation) {
    const { width, height } = img;
    let ret = { width, height };
    if ("68".indexOf(orientation) > -1) {
      // 90、270度宽高互换
      ret = { width: height, height: width };
    }
    // 如果原图小于设定，采用原图
    if (
      ret.width < this.options.maxWidth ||
      ret.height < this.options.maxHeight
    ) {
      return ret;
    }

    const scale = ret.width / ret.height; // 基于宽高的比例进行scale

    if (width && height) {
      if (scale >= width / height) {
        if (ret.width > width) {
          ret.width = width;
          ret.height = Math.ceil(width / scale);
        }
      } else {
        if (ret.height > height) {
          ret.height = height;
          ret.width = Math.ceil(height * scale);
        }
      }
    } else if (width) {
      if (width < ret.width) {
        ret.width = width;
        ret.height = Math.ceil(width / scale);
      }
    } else if (height) {
      if (height < ret.height) {
        ret.width = Math.ceil(height * scale);
        ret.height = height;
      }
    }

    // 超过这个值base64无法生成，在IOS上
    while (ret.width >= 3264 || ret.height >= 2448) {
      ret.width *= 0.8;
      ret.height *= 0.8;
    }

    return ret;
  }

```



### 坑点五：某些webkit版本和微信X5内核 **toDataURL**解析问题

在上述环境中，照片压缩会导致图片变黑、错乱、重复排列问题，如图：

![](https://www.ifajiao.com/assets/2019-01-06/upload-error.jpg)

> Ps: 原因是因为在使用toDataURL(jpeg, 0.5)压缩时，内核使用了png解析算法，在这些机型下需要引用jpeg解析函数去自行解析。



### 坑点六：Formdata中 appendblob为空

这个问题这里就不展开说明了，可以参考[[Blob in FormData is null](https://stackoverflow.com/questions/41398692/blob-in-formdata-is-null)了解解决方案。



## 总结

以上为遇到几个主要的问题，可能还有没有触及到的其他问题，将会再遇到后继续编辑。