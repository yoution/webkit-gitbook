# Static File 
研究静态资源的加载过程,包括Html,Js,Css,Image的从开始加载至加载完成的过程研究


WebCore/loader/cache/CachedResourceLoader  CachedResourceLoader::requestResource

## Html 文件的加载过程
Html文件是浏览器最先加载的资源，加载完成就开始解析，解析过程就是生成DOM树的过程
document  Loading
parser   ParsingState
parser   StoppingState
document Interactive
DOMContentLoadedEvent
document Complete
> parser   StoppedState
parser   DetachedState
LoadEvent
## Js 文件的加载过程
<script type='text/javascript'></script>
<script async src=''></script>
<script defer src=''></script>
<script defer async src=''></script>
<script  async src=''></script>
<script ></script>
<script type='module' src=''></script>  相当于defer
<script type='module'></script>  相当于defer
<script ></script>

<script></script> 卡住defer，但是不卡async
动态插入的script
## Css 文件的加载过程

## Image 文件的加载过程

预加载
仅针对Js文件和Css文件进行预加载
触发条件
1.<script src></script>
2.外部Css资源正在加载中，同时<script></script>,




