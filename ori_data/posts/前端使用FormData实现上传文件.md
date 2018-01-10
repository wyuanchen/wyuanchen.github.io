---
title: 前端使用FormData实现上传文件
date: 2017-01-27 22:36:06
categories: 前端
tags:
- js
- 前端
---
>场景: 用户通过点击图片弹出上传文件的框框，然后选择将要替换的图片，选择后实时预览，点击确定后通过ajax上传到服务器.


**前端html**
```html
<div id="img_div">
    <input type="file" id="img_upload">
    <img id="picture" src="$picturePath$" alt="头像" class="img-rounded">
</div>
```

**将上述html中的上传input元素的透明度设置为0，并且设置宽度和高度，用它来遮住a标签,注意设置外部div的position**   

```css 
#img_div{
    position: relative;
}
#img_upload{
    opacity: 0;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    z-index: 9;
    height: 100%;
}
```

<!-- more -->

**JS代码**

```js
//为头像上传设置实时预览监听器
function setOnchangeListener() {
    $("body").on("change","#img_upload",previewFile);
}

//实时预览上传图片
function previewFile() {
    var preview=$("#picture");
    var file=$("#img_upload")[0].files[0];
    var reader=new FileReader();
    reader.addEventListener("load",function () {
       preview.prop("src",reader.result);
    },false);
    if(file){
        reader.readAsDataURL(file);
    }
}

//上传图片
function setPicture() {
    var file=$("#img_upload")[0].files[0];
    if(file==null)
        return;
    var formData=new FormData();
    formData.append('file',file);
    var url=serverUrl+"uploadPic";
    $.ajax({
        url:url,
        type:'POST',
        cache:false,
        data:formData,
        processData:false,
        contentType:false,
        xhrFields: {
            withCredentials: true
        },
        crossDomain: true,
    });
}
```
**其中**

```js
processData:false,
contentType:false,
```
**的设置需要注意。**

**这里后台我用的是java web（SpringMVC）就贴一下控制器接收上传文件的代码**

```java
@RequestMapping("/uploadPic")
    public @ResponseBody Object  handleUploadPic(@RequestParam("file")MultipartFile file, HttpServletRequest request){
        String userName=cookieService.getUserName(request);
        String picUrl=fileService.saveImg(file);
        String oldPicUrl=manageService.getPicUrl(userName);
        fileService.deleteFile(oldPicUrl);
        manageService.setPicUrl(userName,picUrl);
        Map<String,Object> result=new HashMap<String, Object>();
        result.put(keyStatus,valueStatusOk);
        return result;

    }
```
**其中file就是接收到的上传文件**
