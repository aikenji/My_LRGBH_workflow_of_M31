# My LRGB workflow of M31 with Ha

by aiken 07/29/2023

<!--toc:start-->
- [My LRGB workflow of M31 with Ha](#my-lrgb-workflow-of-m31-with-ha)
  - [Workflow diagram](#workflow-diagram)
  - [Linear process](#linear-process)
    - [Image Registration](#image-registration)
    - [Crop](#crop)
    - [DBE](#dbe)
    - [LinearFit](#linearfit)
    - [RGB channel](#rgb-channel)
      - [RGBHaCombination](#rgbhacombination)
      - [ColorCalibration](#colorcalibration)
      - [NXT](#nxt)
    - [L channel](#l-channel)
      - [BXT](#bxt)
      - [NXT](#nxt)
  - [Nonlinear process](#nonlinear-process)
    - [RGB channel](#rgb-channel)
      - [Stretch](#stretch)
      - [ColorBalance](#colorbalance)
    - [L channel](#l-channel)
      - [LHE](#lhe)
    - [LRGB Combination](#lrgb-combination)
    - [DarkStructureEnhance](#darkstructureenhance)
    - [HDR](#hdr)
    - [Saturation](#saturation)
  - [Conclution](#conclution)
<!--toc:end-->

## Workflow diagram

<img src="./images/workflow.png" alt="workflow">

## Linear process

LRGBHa图像经过wbpp预处理后，开始进入线性处理阶段（linear process）。
线性阶段的图像在屏幕上显示通常较暗，需要通过stf（screen transfer function）才能显示正常，stf在win系统下的快捷键为`ctrl+a`。值得注意的是stf只会改变图像在屏幕上的显示，并不会改变图像实际的像素值。

### Image Registration

首先，LRGBHa的图像如果在预处理阶段，没有使用同一张refference图像对齐，那么需要先进行一次图像对齐。使用StarAlignment工具对齐图像，选取一个通道作为refference，如R通道。

### Crop

LRGBHa五个通道的图像经过对齐后，在边缘区域都会有一些黑边，需要通过DynamicCrop工具剪切掉。可以先选择L通道crop，再重复crop其他通道。值得注意的是当需要把一次crop的参数应用到所有图像时，PI中Dynmaic的组件比较特殊，设置好参数后，通过底边小三角拖出的instance不能直接应用于图像，需要先选取要应用的图像，再双击instance，然后再点击底边的绿色勾，才能执行相同剪裁。详细步骤可参考grapeot的b站视频：<https://www.bilibili.com/video/BV1rp4y1t7MC?t=222.9>。另外也可以使用imageContainer来实现重复操作。

<img src="./images/crop.png" alt="crop" height=200>

### DBE

五个通道裁剪完成后，接着对每个图像分别执行dbe（DynamicBackgroudExtraction），这一步是用来去除背景的梯度，梯度通常是由光污染导致的。

<img src="./images/dbe.png" alt="dbe" height=200>

对于星系这种集中的目标，背景和目标很容易区分，可以使用sample generation功能自动生成背景选点，选点的sample radius设置为30-50，注意去除和星系较近的选点。如果边缘的选点为红色，可以适当增加tolerance 如2.00。完成选点后，拖出instance，可以对其他图像执行相同选点的dbe，过程类似crop中的操作。详细步骤可参考grapeot的视频：<https://www.bilibili.com/video/BV1rp4y1t7MC?t=372.8> 

dbe环节的注意点

- dbe的处理节点可在RGB合成后也可以在合成前，此处还有争议，有待后续研究。
- dbe环节尽可能只做一次，不要重复多次。
- dbe环节的背景去除是用减法还是除法，也没有定论，有待后续研究。

### LinearFit

接下来，图像将分为两个通道分别处理，L通道负责图像的细节，RGBHa（以下简称RGB）通道负责颜色。为了保证各通道图像合成的比例均匀，我们需要将图像在合成前做一次LinearFit。LinearFit的过程很简单，只需选取一个通道作为refference，对剩下的四个通道分别执行一次LinearFit即可。 

![linearfit](./images/linearfit.png)

### RGB channel

---

#### RGBHaCombination

之后图像被分为两个通道，我们先讨论RGB通道。当图像做完LinearFit后，我可以通过PixelMath合成RGBHa的图像，此过程也可以通过RGBCombination模块来实现。使用PixelMath合成时，要注意Ha的比例，为了让Ha呈现出酒红色的小红花，我们可以用0.15的R和0.08的B合成Ha。这样我们就得到了一张RGBH的合成图像，以下简称RGB图像。

![rgbh](./images/rgbh.png)

#### ColorCalibration

接着对合成的RGB图像进行颜色校准，此过程可以通过脚本AutoColor自动完成，颜色校准后的图像会有一些偏色，如偏绿，我们在非线性的过程再来去除。

#### NXT

对色彩校准后的图像进行一次denoice，推荐使用三件套中NXT来完成，需要注意的是NXT的参数，可以通过对一小块区域的preview尝试得到，denoice的标准，参考NGC马拉松，即背景区域有一些黑色斑驳即可，如下图，如果denoice太大，会给人画面油油的感觉。

![nxt](./images/denoice.png)

### L channel

---

#### BXT

对于L通道我们主要考虑突出图像的细节，在线性处理阶段，我们第一步要做的是deconvlution，这一步主要是抵消成像设备的点扩散函数对图像的影响，提升星云和星系的细节，缩小星点的大小，有助于改善星点脱线和对焦不准等问题。推荐使用三件套的BXT工具，或者Ezdecon脚本。

#### NXT

完成BXT后，做一次NXT去噪，完成图像线性部分的处理工作。

## Nonlinear process

线性处理阶段之后便是非线性处理阶段, 该阶段也可分为L通道和RGB通道。

### RGB channel

---

#### Stretch

图像的拉伸是最重要的一步,也是处理图像好坏的最关键一步.图像拉伸主要有以下几种方法:

- HT(HistogramTransform)
- CT(CurveTransform)
- ArcsinhStreth
- EZ SoftStreth
- GHS

每种方法都有其特点,通常一次成功的拉伸,需要综合使用多种工具.其中

HT的特点是可以实时显示图像的直方图,定量控制背景的强度,并且可以通过STF拖拽到HT上实现最简单的拉伸(通常这种方法有些过度拉伸).

CT的特点是可以精确控制图像某个部位的拉伸.

AsinhS的特点是拉伸比较柔和,能够比较好的保护图像细节,常作为拉伸图像的前期拉伸,grapeot的视频中采用了这种方法.

EZSS的特点是和AsinhS类似, 效果比较柔和,可以保护图像细节.

GHS是目前官方比较推荐的一种比较高级的拉伸办法,目前我还没用过,有待后续研究.

本次处理M31的过程主要使用HT拉伸, 如图所示提高shadow(不要出现截断),调低midtone,使得预览的直方图基本落在1/8的位置, 可以重复多次拉伸, 注意不要拉出锯齿,直方图采用12-16bit定量.

<img src="./images/HT.png" alt="HT" height=size>

如果图像除了星系部分,还有暗云气,那么可以分两步拉伸:首先拉伸图像的星系部分;然后拉伸暗云气,注意由于云气部分很暗,拉伸的时候很容易把星系拉曝,为了解决这个问题,可以使用背景mask拉伸.背景mask的制作推荐使用RangeSelection工具,如图所示调低upper limit,使得星系部分为黑色,提高smoothness使得mask的过渡平滑.这样就得到一张只露出背景的mask了.

<img src="./images/range.png" alt="rangemask" height=350>

#### ColorBalance

根据直方图的R、G、B成分是不是重合,确定背景是不是中性灰.如果偏绿,可以使用SCNR去绿,注意搭配背景mask,保护星系的颜色. 之后做RGB的色彩平衡,此处的色彩调整是很主观的,和实际的星系颜色关系不大.同样加上背景mask保护好背景的中性灰.采用CT工具调整颜色平衡,首先拉高Satruatioon,调整绿色曲线,点击图像中要修改色彩的地方,CT会显示该色彩在曲线图中的位置,若要增加绿色,则拉高曲线,反之拉低曲线.若要维持该点的绿色成分,则可以使用多个锚点保护不需要处理的区域.

<img src="./images/colbal.png" alt="colbal" height=400>

另外还有一种方法是调整a、b、c曲线,此方法有待研究. 最后做一步convolution, 减少RGB图像的细节特征, 这一步optional.

###  L channel

---

完成RGB通道的处理后, 我们开始L通道的非线性处理, 为了突出L通道的细节特征, 可以在HT拉伸后, 再用CT对局部拉伸, 方法同上. 需要注意的是,曲线在该点斜率大于1, 表示该点附近的细节被突出, 反之小于1, 细节被掩盖.

#### LHE

L通道拉伸完后, 为了进一步压榨细节, 可以使用LHE工具对局部特征再次进行拉伸. LHE的原理就是对局部做HE, 这样的好处是可以使亮部和暗部的细节都得到提升. 如之前的拉伸使得M31盘面亮部的细节很明显,但是星系的暗云气细节不够明显(注意这里是星系的暗云气, 不是背景的暗云气). 我们使用LHE拉伸这部分云气, 注意一定要使用背景mask, 保护背景, 不然LHE会拉出背景的噪声. L通道的处理就算基本完成了.

<img src="./images/LHE.png" alt="LHE" height=size>

### LRGB Combination

完成两个通道的处理后, 可以对LRGB的合成了. 采用LRGBCombination工具如图所示, 将L通道合成到RGB图像中, 值得注意的是这里的saturation数值越小, 饱和度越大. 可以勾选上Chrominance Noise Reduction 去除图像中的彩色噪声.

<img src="./images/lrgb.png" alt="lrgb" height=300>

合成完后的图像就没有严格的处理顺序了.

### DarkStructureEnhance

该脚本和LHE类似, 其作用是只针对图像的暗结构做加强, M31的星系螺旋暗云气就是用这种方法加强的.不过后期尝试了以下LHE发现效果基本一致.

### HDR

由于星系的核心一般较亮,很容易拉伸过曝, 特别是M31和M42, 所以可以使用EZ HDR脚本减少中心的亮度.

### Saturation

最后,图像的饱和度推荐使用PS来处理, 效果会更好,操作也更简单. 用pixinsight导出16tiff.
导入到photoshop中, 采用cameraRAW filter滤镜中的color mixer工具, 调整特定颜色的饱和度和亮度以及Hue值.

## Conclution

最后得到一张漂亮的M31星系.

<img src="./images/M31.png" alt="m31" height=size>
