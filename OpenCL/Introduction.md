[OpenCL基本教程](https://www.bilibili.com/video/BV1c54y1Y75t?p=2&vd_source=669f6fea14599939187b680ddc901c20)

##EXAMPLE##

**图像旋转原理**

图像旋转是指把定义的图像绕某一点以逆时针或顺时针方向旋转一定的角度， 通常是指绕图像的中心以逆时针方向旋转。假设图像的左上角为(l, t), 右下角为(r, b)，则图像上任意点(x, y) 绕其中心(xcenter, ycenter)逆时针旋转θ角度后， 新的坐标位置(x',y')的计算公式为：

x′ = (x - xcenter) cosθ - (y － ycenter) sinθ + xcenter,

y′ = (x - xcenter) sinθ + (y － ycenter) cosθ + ycenter.

C代码
```
void rotate(
      unsigned char* inbuf,
      unsigned char* outbuf,
      int w, int h,
      float sinTheta,
      float cosTheta)
{
   int i, j;
   int xc = w/2;
   int yc = h/2;
   for(i = 0; i < h; i++)
   {
     for(j=0; j< w; j++)
     {
       int xpos =  (j-xc)*cosTheta - (i - yc) * sinTheta + xc;
       int ypos =  (j-xc)*sinTheta + (i - yc) * cosTheta + yc;
       if(xpos>=0&&ypos>=0&&xpos<w&&ypos<h)
          outbuf[ypos*w + xpos] = inbuf[i*w+j];
     }
   }
}

```

OpenCL C kernel代码：
```
#pragma OPENCL EXTENSION cl_amd_printf : enable
__kernel  void image_rotate(
      __global uchar * src_data,
      __global uchar * dest_data,        //Data in global memory
      int W,    int H,                   //Image Dimensions
      float sinTheta, float cosTheta )   //Rotation Parameters
{
   const int ix = get_global_id(0);
   const int iy = get_global_id(1);
   int xc = W/2;
   int yc = H/2;
   int xpos =  ( ix-xc)*cosTheta - (iy-yc)*sinTheta+xc;
   int ypos =  (ix-xc)*sinTheta + ( iy-yc)*cosTheta+yc;
   if ((xpos>=0) && (xpos< W)   && (ypos>=0) && (ypos< H))
      dest_data[ypos*W+xpos]= src_data[iy*W+ix];
}
```

正如上面代码中所给出的那样，在C代码中需要两重循环来计算横纵坐标上新的 坐标位置。其实，在图像旋转的算法中每个点的计算可以独立进行，与其它点的 坐标位置没有关系，所以并行处理较为方便。OpenCL C kernel代码中用了并行处理。

