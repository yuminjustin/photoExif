  最近遇到一个棘手的问题——手机上拍照并上传图片，因为拍摄角度问题，图片的显示问题，实际情况如下图所示。<br />  
![img](http://yuminjustin.cn/upload/20150913/20150423110458646.jpg "img") <br />
  当客户拍摄时选择的角度问题，照片的显示角度也会有问题，看图软件会自动帮我们矫正，但是在项目中我们就必须要自己去矫正了，之前找到一个关于这方面信息的文章，是腾讯的工程师写的：[查看](http://tgideas.qq.com/webplat/info/news_version3/804/808/811/m579/201409/278736.shtml) <br /> 
  方法有点复杂，首先要将拍照获取的图片文件转成64位编码，然后再将64位编码转换成二进制，这样做的原因是为了去获取图片的EXIF信息，EXIF中包含了照片的相关信息，其中就有一个Orientation的属性，顾名思义，翻译成中文就是设备方向呗。<br />
  首先，将拍照的图片获取。代码如下：
###1、HTML input标签的写法:
        <input type="file" accept="image/*;capture=camera" id="captureFile" capture="camera"/>
###2、绑定input的change事件，并在客户端创建临时的图片对象
        xxx.onchange = function(event){
            var files = event.target.files || event.dataTransfer.files; //获取所选取的文件集合
            var URL = window.URL || window.webkitURL; //URL对象
            var blob =  URL.createObjectURL(files[0]);// 创建blob对象 HTML5的新增类型
            //这里是只选取了一个文件，多个文件需要遍历
            var img = new Image();
            img.src = blob;
            img.file = files[0];
            //临时的图片创建完毕
          ...
        }
###3、利用canvas 得到64位编码（接着上面的省略号）
        img.onload = {
           var canvas = document.createElement("canvas");
           canvas.width = img.width;
           canvas.height = img.height;
           var ctx = canvas.getContext("2d");
           ctx.drawImage(img, 0, 0);
        }
  边看文章边学着做，我的天啊，不是报错就是不运行，搞js的人都知道，js是不可以操作文件的，将图片信息转成64位倒还好办，转成二进制真的难倒我了，那篇文章上也讲得很笼统，太难搞了，几乎快要放弃。<br />
  于是乎就到度娘那儿搜索看看有没有获取图片EXIF的查件，找到了一个jQuery.Exif的插件，例子：[查看](http://developer.51cto.com/art/201207/346157.htm) <br />
  这个例子当中罗列的照片的一系列信息，拍照的设备、型号、白平衡等等等，于是就下载下来，测试了一番。
### ①正常竖拍 Orientation值为6<br />  
![img2](http://yuminjustin.cn/upload/20150913/20150423113934816.jpg "img2")  
### ②正常横拍 Orientation值为1<br />  
![img3](http://yuminjustin.cn/upload/20150913/20150423114302848.jpg "img3")  
### ③逆向竖拍 Orientation值为8<br />  
![img4](http://yuminjustin.cn/upload/20150913/20150423114316513.jpg "img4")  
### ④逆向横拍 Orientation值为3<br />  
![img5](http://yuminjustin.cn/upload/20150913/20150423114328333.jpg "img5") <br />
  有了jQuery.Exif的帮助，那种兜圈子还要搞二进制的方法我就不考虑了。php那边也有接收64位编码并生成图片的代码，也为我的功能实现铺好了路。那么接下来要改写上面的第三点了，因为要提前把图片翻转好，才能生成64位编码。
###新的3、利用jQuery.Exif将拍摄情况分类并创建一个容器canvas（接着上面的省略号）
       var w = img.width, //图片的 宽度
           h = img.height, //图片的高度
           canvas = document.createElement('canvas'),
           ctx,
           exifs = $(img).exifLoad(function () {//刚才生成的临时图片
              var or = exifs.exif("Orientation")[0]; //获取当前的拍摄角度
              //这个值是个数组，也就意味着它会有斜着拍摄的参数，在这里我是以最上面示例图的四个方向作参考，有兴趣的筒子可以深入研究下
              if(or==1||or==3) { //横拍
                ..横拍..
              }
              else if(or==6||or==8) { //竖拍
                ..竖拍..
             }
          }
###4.1、利用canvas 将图片翻转（接着上面的省略号..横拍..）
       canvas.width = w;
       canvas.height = h;
       ctx = canvas.getContext('2d');
       if (or == 3) {
          ctx.translate(canvas.width, canvas.height);
          ctx.scale(-1, -1);
          ctx.drawImage(img, canvas.width - w, canvas.height - h);
          //逆向横拍 利用canvas将图片翻转倒立
       } else {
           ctx.drawImage(img, 0, 0);
          //正常横拍 直接加入到canvas中
       }
###4.2、利用canvas 将图片翻转（接着上面的省略号..竖拍..）
      canvas.width = h;
      canvas.height = w;
      ctx = canvas.getContext('2d');
      if (or == 6) {
         ctx.rotate(90 * Math.PI / 180);  
         ctx.translate(0, -h);
         //正常竖拍，将图片顺时针翻转90度
      } else {
         ctx.rotate(-90 * Math.PI / 180);
         ctx.translate(-w, 0);
         //逆向竖拍，将图片逆时针翻转90度
      }
      ctx.drawImage(that, 0, 0, that.width, that.height, 0, 0, w, h);
###5、获取64位编码，并修改input change事件所选取的对象，大功告成！
      var base64 = canvas.toDataURL('image/jpeg', quality);
      img.file.base64Resize = base64; //将新的64位编码，赋给临时的图片对象
      //因为img.file = files[0]; 你所选择的文件也就被修改了
      //再使用XMLHttpRequest 去上传你的新图片了
###6、android系统（我的是4.4.4）获取图像64位编码失败的解决如下：
     使用JPEGEncoder的插件获取
          var encoder = new JPEGEncoder();
          if (or == 1||or == 3) {
                base64 = encoder.encode(ctx.getImageData(0, 0, w, h), quality * 100);
          } else {
                base64 = encoder.encode(ctx.getImageData(0, 0, h, w), quality * 100);
          }
### 更多详细内容请查看：[点击这里](http://dwz.cn/1JZiE6) 
  利用插件：jQuery.Exif
