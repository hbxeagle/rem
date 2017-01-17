# 前端：『REM』手机屏幕高清适配方案

> 『学而不思则罔，思而不学则殆。』前一篇『[了解真实的『REM』手机屏幕适配](./README.md)』中，只考虑了屏幕宽高和font size对手机屏幕适配的影响。没有处理DPR做『高清适配』。原来采用的1px方案，在开发中还是感觉不太直接。所以在学习了淘宝的『flexible』后，结合它对DPR的处理，得到了现在的高清适配方案。

**注** 本方案需要动态修改 viewport ，如果不习惯这种方式，可以考虑前一篇中给出的方案——『[Retina屏的移动设备如何实现真正1px的线？](http://jinlong.github.io/2015/05/24/css-retina-hairlines/)』。再结合给html标签上设置 data-dpr 属性，达到图片高清适配的目的。

REM适配主要解决在不同屏幕宽度下，布局和元素尺寸保持一致，即屏宽大的对应尺寸也大。而单独使用这个方案在要求更高的『高清适配』中，就有些力不从心了。

先来简单了解一下什么是『DPR』，以及 DPR 带来的影响。

`DPR：当前显示设备上的一个物理像素的尺寸与一个设备独立像素的尺寸(dips)的比率。`

直白一点，对前端开发来说，*如果不进行特殊设置*，就是在 `dpr = 1` 的设备上，css中设置1px，对应显示在屏幕上也是1像素（物理像素）。 `dpr = 2` 的设备上（如iPhone6），css中设置1px，对应显示在屏幕上会是2像素（物理像素）。同理，在 `dpr = 3` 的设备上如（iPhone6 Plus），css中设置1px，对应显示在屏幕上会是3像素（物理像素）。至于详细的关系，这里就不多说了，网上有很多关于 dpr 的说明。

现在来看『高清适配』要解决两个问题：
* 1px问题：即在不同高清屏幕下，设置为1px，希望显示也为1像素（物理像素）。尤其是在边框上此问题比较突出。
* 高清图片适配：将一张 `100px * 100px` 的图片显示在 `100px * 100px` 的区域时，由于在 `dpr = 2` 的设备上，对应的区域其实有 `200 * 200` 个像素（物理像素）点，所以在此设备上，这张图片相当于被拉伸了一倍，显示效果下降。因此要做到高清适配，我们需要在这样的设备上使用一张 `200px * 200px` 的图片，才能保证显示效果。反过来，如果统一使用 `200px * 200px` 的图片，则会造成流量浪费。

问题清楚了，下面直接放出解决方案：

```js
(function(designWidth, rem2px) {
    var win = window;
    var doc = win.document;
    var docEl = doc.documentElement;
    var metaEl = doc.querySelector('meta[name="viewport"]');
    var dpr = 0;
    var scale = 0;
    var tid;

    if (!dpr && !scale) {
      var devicePixelRatio = win.devicePixelRatio;
      if (win.navigator.appVersion.match(/iphone/gi)) {
        if (devicePixelRatio >= 3 && (!dpr || dpr >= 3)) {
          dpr = 3;
        } else if (devicePixelRatio >= 2 && (!dpr || dpr >= 2)){
          dpr = 2;
        } else {
          dpr = 1;
        }
      } else {
        dpr = 1;
      }
      scale = 1 / dpr;
    }

    docEl.setAttribute('data-dpr', dpr);
    if (!metaEl) {
      metaEl = doc.createElement('meta');
      metaEl.setAttribute('name', 'viewport');
      metaEl.setAttribute('content', 'width=device-width,initial-scale=' + scale + ', maximum-scale=' + scale + ', minimum-scale=' + scale + ', user-scalable=no');

      if (docEl.firstElementChild) {
        docEl.firstElementChild.appendChild(metaEl);
      } else {
        var wrap = doc.createElement('div');
        wrap.appendChild(metaEl);
        doc.write(wrap.innerHTML);
      }
    } else {
      metaEl.setAttribute('name', 'viewport');
      metaEl.setAttribute('content', 'width=device-width,initial-scale=' + scale + ', maximum-scale=' + scale + ', minimum-scale=' + scale + ', user-scalable=no');
    }
    // 以上代码是对 dpr 和 viewport 的处理，代码来自 lib-flexible。

    // 一下代码是处理 rem，来自上篇文章。不同的是获取屏幕宽度使用的是 
    // document.documentElement.getBoundingClientRect
    // 也是来自 lib-flexible ，tb的技术还是很强啊。
    function refreshRem(_designWidth, _rem2px){
      // 修改viewport后，对网页宽度的影响，会立刻反应到 
      // document.documentElement.getBoundingClientRect().width
      // 而这个改变反应到 window.innerWidth ，需要等较长的时间
      // 相应的对高度的反应，
      // document.documentElement.getBoundingClientRect().height 
      // 要稍微慢点，没有准确的数据，应该会受到机器的影响。
      var width = docEl.getBoundingClientRect().width;
      var d = window.document.createElement('div');
      d.style.width = '1rem';
      d.style.display = "none";
      docEl.firstElementChild.appendChild(d);
      var defaultFontSize = parseFloat(window.getComputedStyle(d, null).getPropertyValue('width'));
      // d.remove();
      var portrait = "@media screen and (width: "+ width +"px) {html{font-size:"+ ((width/(_designWidth/_rem2px)/defaultFontSize)*100) +"%;}}";
      var dpStyleEl = doc.getElementById('dpAdapt');
      if(!dpStyleEl) {
        dpStyleEl = document.createElement('style');
        dpStyleEl.id = 'dpAdapt';
        dpStyleEl.innerHTML = portrait;
        docEl.firstElementChild.appendChild(dpStyleEl);
      } else {
        dpStyleEl.innerHTML = portrait;
      }
      // 由于 height 的响应速度比较慢，所以在加个延时处理横屏的情况。
      setTimeout(function(){
        var height = docEl.getBoundingClientRect().height;
        var landscape = "@media screen and (width: "+ height +"px) {html{font-size:"+ ((height/(_designWidth/_rem2px)/defaultFontSize)*100) +"%;}}"
        var dlStyleEl = doc.getElementById('dlAdapt');
        if(!dlStyleEl) {
          dlStyleEl = document.createElement('style');
          dlStyleEl.id = 'dlAdapt'
          dlStyleEl.innerHTML = landscape;
          docEl.firstElementChild.appendChild(dlStyleEl);
        } else {
          dlStyleEl.innerHTML = landscape;
        }
      },500);
    }

    // 延时，让浏览器处理完viewport造成的影响，然后再计算root font-size。
    setTimeout(function(){
      refreshRem(designWidth, rem2px);
    }, 1);

  })(640, 100);
```

此方案分为上下两部分。

第一部分为通过设置 viewport ，将物理像素和css中的像素对应起来。iOS下，对于2和3的屏，正常按dpr实际值处理，其余的用1倍方案。

1. 在 dpr = 1 的设备上设置：

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no" />
```
如 Galaxy S5 中，设置前获取屏宽 `window.innerWidth` 为 `360`，设置后还是 `360`。

2. 在 dpr = 2 的设备上：

```html
<meta name="viewport" content="width=device-width, initial-scale=0.5, minimum-scale=0.5, maximum-scale=0.5, user-scalable=no" />
```
如 iPhone6 中，设置前获取屏宽 `window.innerWidth` 为 `375`，设置后是 `750`（与物理像素匹配）。

3. 在 dpr = 3 的设备上：

```html
<meta name="viewport" content="width=device-width, initial-scale=0.3333333333333333, maximum-scale=0.3333333333333333, minimum-scale=0.3333333333333333, user-scalable=no" />
```
如 iPhone6 Plus 中，设置前获取屏宽 `window.innerWidth` 为 `414`，设置后是 `1242`（与物理像素匹配）。


同时在html标签上设置 「data-dpr」属性，标注当前设备的dpr值，为后面做图片高清适配做准备。

第二部分为设置了 viewport 后，获取新的屏幕高宽，然后按照等比缩放原则，考虑系统字体大小的影响（前一篇中解决的问题），计算出我们需要的 root font size。

**注意** 一定是要在设置的 viewport 生效后，再进行计算，不然获取到的屏幕高宽不准确。所以在这里使用了两个延时来保证这一点。

第一个延时，让浏览器处理完修改 viewport 造成的影响，然后再计算root font-size。

```js
setTimeout(function(){
  refreshRem(designWidth, rem2px);
}, 1);
```

第二个延时，由于 height 对 viewport 的更改响应速度比较慢，所以在加个延时处理横屏的情况。

```js
setTimeout(function(){
  var height = docEl.getBoundingClientRect().height;
  ...
});
```

另外，最好将上面的代码用内联的方式，直接放置到页面的 head 里面，由于js处理会阻塞页面渲染（因为宽度对 viewport 的更改相应很快，所以不必担心上面的代码占用太长的时间），利用这一点，可以保证后面的页面渲染的时候不会闪烁。

这样在css中你就可以放心的使用 1px 来设置 1像素的边框了。而高清图片的适配，就需要使用下面的代码

```css
.pic {
  width:1rem;
  height:1rem;
  background: url(../images/100_100.png);
  background-repeat: no-repeat;
  background-size: cover;
}
[data-dpr="2"] .pic {
  background-image: url("../images/100_100@2x.png");
}
[data-dpr="3"] .pic {
  background-image: url("../images/100_100@3x.png");
}
```

这时候使用 less 或 sass 就很方便了
> 只给出了替换 .png 的示例。

less:

```less
.dpr(@selector, @img) {
  [data-dpr="2"] {
    @_img : replace("@{img}", '\.png$' ,'@2x.png');
    @{selector} {
      background-image: url("@{_img}");
    }
  }
  [data-dpr="3"] {
    @_img : replace("@{img}", '\.png$' ,'@3x.png');
    @{selector} {
      background-image: url("@{_img}");
    }
  }
}
.pic {
  width:1rem;
  height:1rem;
  background: url(../images/100_100.png);
  background-repeat: no-repeat;
  background-size: cover;
}
// 这里要加使用 ~".pic"， 
// 不然生成的 selector 会是带有引号的 [data-dpr="2"] ".pic"
// 而不是 [data-dpr="2"] .pic
.dpr(~".pic", "../images/100_100.png");
```

sass:

```sass
@function str-replace($string, $search, $replace: '') {
  $index: str-index($string, $search);

  @if $index {
    @return str-slice($string, 1, $index - 1) + $replace + str-replace(str-slice($string, $index + str-length($search)), $search, $replace);
  }

  @return $string;
}

@mixin dpr($selector, $img) {
  [data-dpr="2"] {
    $_img : str-replace("#{$img}", '\.png' ,'@2x.png');
    #{$selector} {
      background-image: url("#{$_img}");
    }
  }
  [data-dpr="3"] {
    $_img : str-replace("#{$img}", '\.png' ,'@3x.png');
    #{$selector} {
      background-image: url("#{$_img}");
    }
  }
}
.pic {
  width:1rem;
  height:1rem;
  background: url(../images/100_100.png);
  background-repeat: no-repeat;
  background-size: cover;
}

@include dpr($selector:".pic", $img:"../images/100_100.png")
```

另外附赠一个webp特性检查，也可以一起放到上面的立即执行函数中，同时更新一下 less 或 sass ，就可以很方便使用webp了。

js

```js
// 监测webp支持情况，如果支持为html标签添加属性：data-webp=1
(function() {
  var webp = new Image();
  webp.onload = webp.onerror = function() {
    webp.height === 2 && document.documentElement.setAttribute('data-webp', 1);
    webp.onload = webp.onerror = null;
    webp = null;
  };
  webp.src = 'data:image/webp;base64,UklGRjoAAABXRUJQVlA4IC4AAACyAgCdASoCAAIALmk0mk0iIiIiIgBoSygABc6WWgAA/veff/0PP8bA//LwYAAA';
})();
```

less

```less
.dpr-webp(@selector, @img) {
  [data-dpr="2"] {
    @_img : replace("@{img}", '\.png$' ,'@2x.png');
    @{selector} {
      background-image: url("@{_img}");
    }
  }
  [data-dpr="3"] {
    @_img : replace("@{img}", '\.png$' ,'@3x.png');
    @{selector} {
      background-image: url("@{_img}");
    }
  }
  [data-webp="1"] {
    @_img : replace("@{img}", '\.png$' ,'.webp');
    @{selector} {
      background-image: url("@{_img}");
    }
  }

  [data-webp="1"][data-dpr="2"] {
    @_img : replace("@{img}", '\.png$' ,'@2x.webp');
    @{selector} {
      background-image: url("@{_img}");
    }
  }

  [data-webp="1"][data-dpr="3"] {
    @_img : replace("@{img}", '\.png$' ,'@3x.webp');
    @{selector} {
      background-image: url("@{_img}");
    }
  }
}
```

sass
```sass
@function str-replace($string, $search, $replace: '') {
  $index: str-index($string, $search);

  @if $index {
    @return str-slice($string, 1, $index - 1) + $replace + str-replace(str-slice($string, $index + str-length($search)), $search, $replace);
  }

  @return $string;
}

@mixin dpr-webp($selector, $img) {
  [data-dpr="2"] {
    $_img : str-replace("#{$img}", '\.png' ,'@2x.png');
    #{$selector} {
      background-image: url("#{$_img}");
    }
  }
  [data-dpr="3"] {
    $_img : str-replace("#{$img}", '\.png' ,'@3x.png');
    #{$selector} {
      background-image: url("#{$_img}");
    }
  }
  [data-webp="1"] {
    $_img : str-replace("#{$img}", '\.png' ,'.webp');
    #{$selector} {
      background-image: url("#{$_img}");
    }
  }

  [data-webp="1"][data-dpr="2"] {
    $_img : str-replace("#{$img}", '\.png' ,'@2x.webp');
    #{$selector} {
      background-image: url("#{$_img}");
    }
  }

  [data-webp="1"][data-dpr="3"] {
    $_img : str-replace("#{$img}", '\.png' ,'@3x.webp');
    #{$selector} {
      background-image: url("#{$_img}");
    }
  }
}
```

**结语** 由于要处理『高清适配』同时兼容Android在font size的bug，所以过程有些绕。另外，不直接使用 lib-flexible 是有两点，一方面不习惯它的类vw的尺寸书写方式（与1vw对应屏宽类似，使用10rem对应屏宽，开发时需要使用插件将测量值换算为目标值），另一方面它也没去解决Android的font size的bug。

by bx 2016.12.14








