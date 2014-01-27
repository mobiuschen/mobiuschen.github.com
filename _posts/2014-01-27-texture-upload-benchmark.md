Texture Upload Benchmark Report
================================  
  
使用png2atf工具处理材质  
---------------------  
  
转换ATF时，大家应该都会使用png2atf工具。  
测试图：  
//TODO 插入图片  
  
###参数解释  
  
ATF工具的相关文档：    
[Adobe Texture Format (ATF) tools user's guide](http://www.adobe.com/devnet/flashruntimes/articles/atf-users-guide.html)      
[Introducing compressed textures with the ATF SDK](http://www.adobe.com/devnet/flashruntimes/articles/introducing-compressed-textures.html)    
但是我发现png2atf的相关文档、文章，有几个参数没有解释清楚。  
  
-c 进行Block Compression（块压缩）。相关：[Texture Compression](http://renderingpipeline.com/2012/07/texture-compression/) 
-r 对进行「块压缩」的文件再使用JPEG-XR+LZMA压缩。该参数只有在使用了"-c"情况下才有用，而且怀疑对于没有"-c"的ATF，"-r"是默认启用的。使用了这个参数之后，材质上传到GPU之前，需要一个解压的过程，这个过程是由CPU完成的。  
-q <0-180> 对JPEG-XR调整压缩质量，0表示无损，这个值是非线性的。没有使用"-c"时，默认应该是使用"-q 15"；使用了"-c"时，默认使用"-q 0"。  
-4|-2|-0 这组参数是设置color space。使用了"-c"默认为"-4"；没有使用"-c"，默认为"-0"。  
LZMA是7z使用的压缩算法，我认为这里的效果大致等同于对没有"-r"的文件手动压缩为"*.7z"文件.  
  
###Compare File Size  

下面对同一张png图片在不同情况下进行压缩，从文件名上表明了其使用的参数，并且全部都关闭了mipmap功能。

观察0：使用"-r"参数，压缩比例大概是60+%。而且跟对比第2和第3组数据，使用了"-r"和直接压缩成7z文件效果是类似的。因此这里有一个思路，可以把「解压」和「上传GPU」过程拆分开。  
观察1：不使用块压缩（"-c"）打包的ATF，反而比针对各平台进行块压缩的ATF小。  
观察2：test-1024.atf和test-1024-r-q15-0.atf文件大小完全一样，因此可以猜测"-q 15 -0"就是其默认参数。  
观察3：test-1024.atf和test-1024-cd-r-q0-4.atf文件大小完全一样，因此可以猜测使用了"-c"参数，"-q 0 -4"是其默认参数。  
  
//todo 美化数据格式  
  
Size    File name  
2639012 test-1024.png  
  
 309553 test-1024.atf  
 699212 test-1024-cd.atf  
 699212 test-1024-ce.atf  
 699284 test-1024-cp.atf  
2097412 test-1024-c.atf  
  
 309553 test-1024-r.atf  
 429124 test-1024-cd-r.atf  
 425295 test-1024-ce-r.atf  
 398188 test-1024-cp-r.atf  
1251871 test-1024-c-r.atf  
  
 457656 test-1024-cd.7z  
 427794 test-1024-ce.7z  
 439708 test-1024-cp.7z  
 309951 test-1024.7z  
  
 309553 test-1024-0.atf  
 405453 test-1024-2.atf  
 602695 test-1024-4.atf  
   
1604985 test-1024-q0.atf  
 726209 test-1024-q5.atf  
 338835 test-1024-q10.atf  
 309553 test-1024-q15.atf  
 269313 test-1024-q20.atf  
 223755 test-1024-q30.atf  
 157045 test-1024-q40.atf  
  94559 test-1024-q50.atf  
  60845 test-1024-q60.atf  
 699284 test-1024-cp-q0.atf  
  
 429124 test-1024-cd-r-q0-4.atf  
 309553 test-1024-r-q15-0.atf  
  
材质的显存占用
------------

没有使用「块压缩」的材质，最大的问题就是所占的显存大小。在没有关闭mipmap的情况下，会去到6 Bytes/pix。即：  
512 * 512 * 6 Bytes = 1.5 MB  
1024 * 1024 * 6 Bytes = 6 MB  
2048 * 2048 * 6 Bytes = 24 MB  

而使用了「块压缩」的材质，同样是关闭mipmap，所占显存只有6 bits/pix。即：  
512 * 512 * 6 bits = 192 KB  
1024 * 1024 * 6 Bytes = 768 KB  
2048 * 2048 * 6 Bytes = 3 MB  

这么一对比，「块压缩」是非常给力的，非常适合于场景地图这种显存大户。

另外，有些情况下，游戏中需要动态地将位图转为材质，这个时候是无法使用ATF的。这时候可以使用Texture.fromBitmapData()方法直接把位图转换为材质。  
这里有个format为不同参数时，所占每像素所占显存大小是不同的：
    * Context3DTextureFormat.BGRA：没有做压缩，保持BRGA每个通道1 Byte，即4 Bytes/pix。
    * Context3DTextureFormat.BGRA_PACKED：使用BGRA444，即4个通道都是各占4 bits，共16 bits/pix。
    * Context3DTextureFormat.BGR_PACKED：使用BGR565，即Green channel 6bits，Blue Channel、Red Channel 5bits，共16 bits/pix。
    
    Texture.fromBitmapData(bitmapData, false, false, 1, Context3DTextureFormat.BGRA);
    Texture.fromBitmapData(bitmapData, false, false, 1, Context3DTextureFormat.BGRA_PACKED);

使用Context3DTextureFormat.BGRA_PACKED或Context3DTextureFormat.BGR_PACKED时，实际上3 Bytes/pix。  

注意，这里都是关闭mimap的情况下的显存占用。开启mipmap之后，显存占用会大33%左右。

//TODO 这里有个问题没有搞明白，为什么Context3DTextureFormat.BGRA中，4个channel每个占8 bits，但Scout中显示6 Bytes/pix。  
//Context3DTextureFormat.BGRA_PACKED或Context3DTextureFormat.BGR_PACKED，通道共16 bits，但Scout中显示3 Bytes/pix。  

材质的上传
---------

材质上传的耗时受材质格式、材质尺寸影响、是否异步等因素影响。
废话少说，贴上实验数据，得到以下观察

观察 0："-r"，处理过的材质，需要在上传到gpu前，在cpu中进行解压。
"-r"，表示对已进行“块压缩”的材质，使用JPEG-XR+LZMA算法对材质进行压缩，可以有效减少atf大小（大概为61%）；代价是，在使用时，需要对材质进行解压，这个过程是在CPU里进行的。
而且，"-r"只在使用了"-c"的情况下有用。
对比，case2和case3，在cpu里解压这个操作还是比较久的。

观察 1：材质释放的效率和材质所占显存是线性相关的。
使用了"-c"的材质，释放的耗时大概是没有"-c"的材质的18%，这个接近各自所占gpu memory的比例.

观察 2：应该利用好CPU解压和GPU上传的流水线。
观察处理"-r"材质的case，发现同步处理要比异步处理慢。导致这种原因是因为处理流水线的问题：
同步处理的流水线
    解压材质1(cpu) - 上传材质1(GPU) - 解压材质2(cpu) - 上传材质2(GPU) - 解压材质3(cpu) - 上传材质3(GPU)
异步处理的流水线
    解压材质1(CPU) - 上传材质1(GPU)
                    解压材质2(CPU) - 上传材质2(GPU)
                                    解压材质3(CPU) - 上传材质3(GPU)

观察 4：对于"[-4|-2|-0]"使用不同color space解压时间 [-4] > [-2] > [-0].
观察case 4，5，6
rgb打包，默认使用"-0"；使用"-c"，默认使用"-4".

观察 5：对于"-q <0-180>]"，质量越高，解压越慢
观察case 7，8，9，10

怀疑 1：没有使用"-c"的情况下，默认使用了"-r"。
没有使用“块压缩”的情况下，使用rgb打包，上传gpu过程的耗时和"-c -r"是相似的，并且cpu会在这时陡高。
使用7z工具对"test-1024.atf"进行压缩后，文件大小基本不变。

怀疑 2："-r"解压的时间和文件大小线性相关，而上传gpu的时间和材质大小关系不大


[Starling] Display Driver: OpenGL (Baseline Constrained)
Test Case 1
Open file test-1024.png, size=2577.16KB:0 ms
Read file test-1024.png:65 ms
Decode png file test-1024.png:45 ms
Upload 20 BitmapData, format='bgra':1190 ms
Upload 20 BitmapData, format='bgrPacked565':3092 ms
Dispose 40 textures:10 ms
----------------------------------------
Test Case 2
Open file test-1024.atf, size=302.30KB:0 ms
Read file test-1024.atf:18 ms
Sync upload all 20 textures: 823 ms
Async upload all 20 texture: 734 ms
Dispose 40 textures:17 ms
----------------------------------------
Test Case 3
Open file test-1024-c.atf, size=2048.25KB:0 ms
Read file test-1024-c.atf:38 ms
Sync upload all 20 textures: 41 ms
Async upload all 20 texture: 50 ms
Dispose 40 textures:2 ms
----------------------------------------
Test Case 4
Open file test-1024-c-r.atf, size=1222.53KB:0 ms
Read file test-1024-c-r.atf:22 ms
Sync upload all 20 textures: 521 ms
Async upload all 20 texture: 326 ms
Dispose 40 textures:3 ms
----------------------------------------
Test Case 5
Open file test-1024-0.atf, size=302.30KB:0 ms
Read file test-1024-0.atf:15 ms
Sync upload all 20 textures: 820 ms
Async upload all 20 texture: 655 ms
Dispose 40 textures:16 ms
----------------------------------------
Test Case 6
Open file test-1024-2.atf, size=395.95KB:1 ms
Read file test-1024-2.atf:14 ms
Sync upload all 20 textures: 948 ms
Async upload all 20 texture: 756 ms
Dispose 40 textures:16 ms
----------------------------------------
Test Case 7
Open file test-1024-4.atf, size=588.57KB:1 ms
Read file test-1024-4.atf:26 ms
Sync upload all 20 textures: 1175 ms
Async upload all 20 texture: 996 ms
Dispose 40 textures:16 ms
----------------------------------------
Test Case 8
Open file test-1024-cd.atf, size=682.82KB:9 ms
Read file test-1024-cd.atf:28 ms
Sync upload all 5 textures: 4 ms
Async upload all 5 texture: 41 ms
Dispose 10 textures:1 ms
----------------------------------------
Test Case 9
Open file test-1024-cd.atf, size=682.82KB:1 ms
Read file test-1024-cd.atf:0 ms
Sync upload all 10 textures: 5 ms
Async upload all 10 texture: 43 ms
Dispose 20 textures:1 ms
----------------------------------------
Test Case 10
Open file test-1024-cd.atf, size=682.82KB:0 ms
Read file test-1024-cd.atf:0 ms
Sync upload all 20 textures: 10 ms
Async upload all 20 texture: 49 ms
Dispose 40 textures:3 ms
----------------------------------------
Test Case 11
Open file test-1024-cd-r.atf, size=419.07KB:1 ms
Read file test-1024-cd-r.atf:23 ms
Sync upload all 5 textures: 129 ms
Async upload all 5 texture: 120 ms
Dispose 10 textures:0 ms
----------------------------------------
Test Case 12
Open file test-1024-cd-r.atf, size=419.07KB:0 ms
Read file test-1024-cd-r.atf:0 ms
Sync upload all 10 textures: 260 ms
Async upload all 10 texture: 164 ms
Dispose 20 textures:1 ms
----------------------------------------
Test Case 13
Open file test-1024-cd-r.atf, size=419.07KB:0 ms
Read file test-1024-cd-r.atf:0 ms
Sync upload all 20 textures: 509 ms
Async upload all 20 texture: 326 ms
Dispose 40 textures:2 ms
----------------------------------------
Test Case 14
Open file test-256-cd.atf, size=42.80KB:2 ms
Read file test-256-cd.atf:7 ms
Sync upload all 20 textures: 4 ms
Async upload all 20 texture: 41 ms
Dispose 40 textures:0 ms
----------------------------------------
Test Case 15
Open file test-256-cd-r.atf, size=31.07KB:0 ms
Read file test-256-cd-r.atf:15 ms
Sync upload all 20 textures: 43 ms
Async upload all 20 texture: 39 ms
Dispose 40 textures:0 ms
----------------------------------------
Test Case 16
Open file test-512-cd.atf, size=170.81KB:0 ms
Read file test-512-cd.atf:16 ms
Sync upload all 20 textures: 4 ms
Async upload all 20 texture: 41 ms
Dispose 40 textures:1 ms
----------------------------------------
Test Case 17
Open file test-512-cd-r.atf, size=112.35KB:0 ms
Read file test-512-cd-r.atf:21 ms
Sync upload all 20 textures: 145 ms
Async upload all 20 texture: 90 ms
Dispose 40 textures:1 ms
----------------------------------------
Test Case 18
Open file test-1024-cd.atf, size=682.82KB:0 ms
Read file test-1024-cd.atf:1 ms
Sync upload all 20 textures: 11 ms
Async upload all 20 texture: 48 ms
Dispose 40 textures:2 ms
----------------------------------------
Test Case 19
Open file test-1024-cd-r.atf, size=419.07KB:0 ms
Read file test-1024-cd-r.atf:0 ms
Sync upload all 20 textures: 578 ms
Async upload all 20 texture: 323 ms
Dispose 40 textures:3 ms
----------------------------------------
Test Case 20
Open file test-2048-cd.atf, size=2730.84KB:0 ms
Read file test-2048-cd.atf:70 ms
Sync upload all 20 textures: 77 ms
Async upload all 20 texture: 95 ms
Dispose 40 textures:11 ms
----------------------------------------
Test Case 21
Open file test-2048-cd-r.atf, size=1571.78KB:0 ms
Read file test-2048-cd-r.atf:38 ms
Sync upload all 20 textures: 1924 ms
Async upload all 20 texture: 1211 ms
Dispose 40 textures:11 ms
----------------------------------------
Test Case 22
Open file test-1024-q60.atf, size=59.42KB:0 ms
Read file test-1024-q60.atf:15 ms
Sync upload all 20 textures: 595 ms
Async upload all 20 texture: 476 ms
Dispose 40 textures:16 ms
----------------------------------------
Test Case 23
Open file test-1024-q30.atf, size=218.51KB:1 ms
Read file test-1024-q30.atf:18 ms
Sync upload all 20 textures: 748 ms








  
