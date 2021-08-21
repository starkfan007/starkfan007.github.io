---

layout: post

title: The software ISP implementation in libcamera

---



## What is ISP

The mainstream CMOS and CCD sensors almost all output RAW data in Bayer mosaic format. This data format cannot be viewed directly. It must be converted to a common RGB or YUV format to be supported by mainstream image processing software. For camera products, it is generally necessary to further convert RGB or YUV images into JPEG format to facilitate storage. The above-mentioned image processing processes are collectively referred to as image signal processing (Image Signal Processing, ISP). The generalized ISP includes JPEG and H.264/265 image compression processing, while the narrowly defined ISP only includes the processing process of converting from RAW format to RGB or YUV.

## Why we need software ISP

Because image signal processing involves a large amount of data and strict real-time requirements, ISP usually adopts hardware implementation, which makes difficult to customize the imaging algorithm for developers, especially in certain scenarios, the default camera pipeline may not meet the imaging requirements, and we need to design better algorithms. So a software-based ISP would be useful for testing and experimentation.

## What is different when add software ISP to libcamera

Adding software ISP to libcamera have to adapt all things to libcamera. You cannot capture images from the test scenarios and export RAW images like operating a digital camera. In a digital camera, the camera driver has been written for you,  and you just need to carefully design a certain algorithm (such as white balance) and then evaluate the image quality. Therefore, many researcher implement there own algorithms based on digital camera and test them on PC.

In libcamera, you need to call the wrapped V4L2VideoDevice class to drive the embedding camera (such as OV5647). At the same time, you need to design the mechanism of data flow in the Pipeline handler, which means how image data is transferred and converted between applications and sensors. I will show more details next section. As for ISP algorithms in detail, fancy libcamera algorithms are not what we need in the beginning, so I choose list of basic algorithms. Maybe we could add more advanced algorithms to software ISP in libcamera . (If we really need it.)

## What is done

### Patches

All the patches submmited can be seen [here](https://github.com/starkfan007/GSoc2021)

```
pipeline: isp: The software ISP-based pipeline handler
libcamera: swisp: The software ISP class
libcamera: framebuffer: Add the friend class ISPCPU
pipeline: isp: All meson configure files
```

### PipelineHandler for testing

In order to test algorithm, I wrote a pipeline handler for testing, through which data flows interact within between the sensor and application, the corresponding isp algorithm will be called in the pipeline handler, and the final output is a binary file. You can use the higher-level tools -- python /matlab to write a script for displaying.

The data interaction process in the pipeline handler can be summarized as follows:

Application applies for the output image format (e.g. RGB888) and allocation of memory. ```generateConfiguration()``` and ```validate()``` are responsible for initializing and verifying whether the applied format matches the format supported by the ISP. ```exportBuffer()``` is responsible for exporting the allocated memory to Application. ```configure()``` is responsible for configuring the ISP input format or sensor output format (e.g. SBGGR10). The corresponding memory is allocated and managed by the Camera driver. It is called in ```start()``` and ```queueDeviceRequest()```, and two queues are defined to manage input and output image. When the ```signal::bufferReady``` is emitted, the corresponding Bayer image data is filled into the memory managed by ```rawBufferQueue```. At this time, the input buffer and output buffer are taken out of the queue and sent to the ISP, processed by the ISP pipeline, and output the image in the requested format. Finally, the resources of each application are released, and the entire software ISP  is now complete.

### Software-based ISP

#### Interface design

Speaking of interfaces, this implementation creates a abstract base class that defines the software ISP API. This CPU-based implementation would then inherit from the base class, and a future GPU-based implementation would also do the same, exposing the same API. This will allow switching between the two implementations seamlessly, without any change in the pipeline handler. 

#### ISP Algorithm

##### Black Level Correction

Many cameras provide a Black Level parameter that represents the black level of the sensor that deviates from 0 due to sensor
noise. However, the actual black level value may be different from the given value. The software ISP implements the calibration of the black level value of the ov5647 camera as an initialization parameter. A simple method that can be done is to use a lens cover or black cloth to block the lens to ensure that no light enters. Take the average of the b, gb, gr, and r channels respectively to get the black level of the corresponding channel.

##### Demosaic

The simplest approach to demosaicing is bilinear interpolation, in which the three color planes are independently interpolated using symmetric bilinear interpolation from the nearest neighbors of the same color. Demosaic is designed based on bilinear interpolation filter. The bilinear demosaicing the green value  at a pixel location  that falls in a red or blue pixel, is computed by the average of the neighboring green values in a cross pattern. For the red and blue value, the interpolation method is similar.

##### Auto White Balance 

When we discuss white balance, the key is to realize the estimation of the color temperature of the scene light source. As a beginning, we use the basic gray-scale world model to estimate the color temperature of the scene. The hypothesis is that for an image with a large number of color changes, the statistical average of the three components of RGB tends to the same gray value. If the average value of the RGB components in the image deviates from 1:1:1, the algorithm will adjust the RGB gain to compensate for changes in ambient light.

##### Gamma Correction

Gamma correction helps to map data into a more perceptually uniform domain, so as to optimize perceptual performance of a limited signal range, such as a limited number of bits in each RGB component. Gamma correction is, in the simplest cases, defined by $V_{out} = AV_{in}^{\gamma}$. ​​Nowadays, gamma correction is not only used as a method to match the nonlinear response of
display devices, but a tool to adjust the input by mapping to nonlinear response to extend or compress the dynamic range of image.

##### Tone Mapping

We use the automatic contrast algorithm to achieve global tone mapping. Global tone mapping uses the same mapping function for all pixels in the image, so the same input pixel value will be definitely mapped to the same output pixel value. The specific method is to first calculate the histogram of the RGB channels separately, calculate the maximum and minimum color scale values of each channel according to the given parameters, construct the LUT according to the color scale values of each channel, and map all pixels of the image according to the LUT.

##### Noise Reduction

## Further work could be done

- Fix and add more ISP algorithms to ISPCPU
- Add ISPGPU implementations

- Instead of test pipeline handler, add software ISP module to simple pipeline handler

## Final words

Thanks to my mentor ***Paul. Elder*** for such a wonderful experience.

I have become a better programmer. I learned a lot of useful skills, use GDB for debugging, Git for version control, V4L2 for camera driver. My mentor ***Paul.Elder*** gave me so many useful advices and taught me what a good commit should be and how to write robust code. Thanks to ***Laurent.Pinchart*** gave me the advice how to design software ISP API. 

Best regards

Siyuan

