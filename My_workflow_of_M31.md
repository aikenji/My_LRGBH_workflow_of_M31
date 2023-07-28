# My LRGB workflow of M31 with Ha

<!--toc:start-->
- [My LRGB workflow of M31 with Ha](#my-lrgb-workflow-of-m31-with-ha)
  - [workflow diagram](#workflow-diagram)
  - [linear process](#linear-process)
    - [star](#star)
<!--toc:end-->

by aiken 2023-07-28

## workflow diagram

## linear process

LRGBHa图像经过wbpp预处理后，开始进入线性处理阶段（linear process）。
线性阶段的图像在屏幕上显示通常较暗，需要通过stf（screen transfer function）才能显示正常，stf在win系统下的快捷键为`ctrl+a`。值得注意的是stf只会改变图像在屏幕上的显示，并不会改变图像实际的像素值。

### image registration

首先，LRGBHa的图像如果在预处理阶段，没有使用同一张refference图像对齐，那么需要首先进行一次图像对齐。使用StarAlignment工具对齐图像，使用一个通道作为refference，如R通道。

### crop

LRGBHa五个通道的图像经过对齐后，在边缘区域都会有一些黑边，需要通过DynamicCrop工具剪切掉。可以先选择L通道crop，再重复crop其他通道。值得注意的是当需要把一次crop的参数应用到所有图像时，PI中Dynmaic的组件比较特殊，设置好参数后，通过底边小三角拖出的instance不能直接应用于图像，需要先选取要应用的图像，再双击instance，然后再点击底边的绿色勾，才能执行相同剪裁。详细步骤可参考grapeot的b站视频：<https://www.bilibili.com/video/BV1rp4y1t7MC?t=222.9>。另外也可以使用imageContainer来实现重复操作。

![crop](./images/crop.png =200x)

### dbe

五个通道裁剪完成后，接着对每个图像分别执行dbe（DynamicBackgroudExtraction），这一步是用来去除背景的梯度，梯度通常是由光污染导致的。

![dbe](./images/dbe.png =200x)

对于星系这种集中的目标，背景和目标很容易区分，可以使用sample generation功能自动生成背景选点，选点的sample radius设置为30-50，注意去除和星系较近的选点。如果边缘的选点为红色，可以适当增加tolerance 如2.00。完成选点后，拖出instance，可以对其他图像执行相同选点的dbe，过程类似crop中的操作。详细步骤可参考grapeot的视频：
