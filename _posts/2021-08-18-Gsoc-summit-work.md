---

layout: post

title: The software ISP implementation in libcamera

---



## What is ISP?

> An image processor, also known as an image processing engine**, **image processing unit (IPU), or image signal processor (ISP), is a type of media processor or specialized digital signal processor(DSP) used for image processing, in digital cameras or other devices.
>
> -from wikipedia 

The mainstream CMOS and CCD sensors almost all output RAW data in Bayer mosaic format. This data format cannot be viewed directly. It must be converted to a common RGB or YUV format to be supported by mainstream image processing software. For camera products, it is generally necessary to further convert RGB or YUV images into JPEG format to facilitate storage. The above-mentioned image processing processes are collectively referred to as image signal processing (Image Signal Processing, ISP). The generalized ISP includes JPEG and H.264/265 image compression processing, while the narrowly defined ISP only includes the processing process of converting from RAW format to RGB or YUV.

## Why we need software ISP in libcamera

Because image signal processing involves a large amount of data and strict real-time requirements, ISP usually adopts hardware implementation, which makes difficult to customize the imaging algorithm for developers, especially in certain scenarios, the default camera pipeline may not meet the imaging requirements, and we need to design better algorithms. So a software-based ISP would be useful for testing and experimentation.

## What is different when add software ISP to libcamera

Adding software ISP to libcamera have to adapt all things to libcamera. You cannot capture images from the test scenarios and export RAW images like operating a digital camera. In a digital camera, the camera driver has been written for you,  and you just need to carefully design a certain algorithm (such as white balance) and then evaluate the image quality. Therefore, many researcher implement there own algorithms based on digital camera and test them on PC.

In libcamera, you need to call the wrapped V4L2VideoDevice class to drive the embedding camera (such as OV5647). At the same time, you need to design the mechanism of data flow in the Pipeline handler, which means how image data is transferred and converted between applications and sensors. I will show more details next section. As for ISP algorithms in detail, fancy libcamera algorithms are not what we need in the beginning, so I choose list of basic algorithms. Maybe we could add more advanced algorithms to software ISP in libcamera . (If we really need it.)

## What is done

### Patches

### PipelineHandlerISP for testing

### Software-based ISP

#### Interface design

#### ISP Algorithm

## Further work could be done

## Final words

## For ISP in libcamera

## For GSoC

