---
title: "Efficient Hybrid Zoom using Camera Fusion on Mobile Phones Reading Notes"
date: 2025-01-15
draft: false
description: "a description"
math: true
tags: ["example", "tag", "paper"]
---
{{< katex >}}

## Abstract
DSLR cameras can achieve various zoom levels by changing the lens distance or swapping lens types. However, due to space constraints, these techniques are not feasible on mobile devices. Most smartphones adopt a **hybrid zoom system**: typically using a wide-angle (W) camera for low zoom levels and a telephoto (T) camera for high zoom levels. To simulate zoom levels between W and T, these systems crop and digitally enlarge images from W, resulting in significant detail loss. In this paper, we propose an efficient hybrid zoom super-resolution system for mobile devices. This system captures **synchronized W and T image pairs** and **utilizes a machine learning model to align and transfer details from T to W**. We further develop an **adaptive blending method** that can handle **depth of field mismatches, scene occlusions, motion uncertainties, and alignment errors**. To minimize domain differences, we design a dual smartphone camera setup to capture real scene inputs and real label data for supervised training. In extensive evaluations on real scenes, our method can generate 12-megapixel images on mobile platforms in 500 milliseconds, achieving TA.

## Key word
hybrid zoom, dual camera fusion, deep neural networks
## Key Takeaways

### Problem Addressed
Almost all upsampling methods (such as bilinear interpolation, bicubic interpolation, etc.) lead to varying degrees of image quality degradation. Consequently, numerous methods have emerged that utilize high zoom T images as references to add real details to low zoom W images. Commercial solutions are not publicly available, and academic research is relatively inefficient. These methods often perform poorly on mobile devices, are susceptible to defects in reference images, and may introduce domain differences between training and inference. To address these issues, we propose a Hybrid Zoom Super-Resolution (HZSR) system.

### Main Contributions
- **A machine learning-based hybrid zoom super-resolution (HZSR) system** that operates efficiently on mobile devices and exhibits strong robustness to imperfections in real scene images (see Section 3).
- **A training strategy** that minimizes domain differences through a dual smartphone camera platform and avoids learning trivial mappings in reference super-resolution (RefSR) tasks (see Section 4).
- **A high-quality synchronized dataset containing 150 pairs of high-resolution (12MP) W and T images**, named the Hzsr dataset, which will be released on our project website for future research (see Section 5).

## Methodology
### Approach
#### **Adapting to Imperfect Reference Images**
We propose an efficient **defocus detection algorithm** that **excludes defocused areas based on the correlation between scene depth and optical flow**. By combining **defocus maps, alignment errors, optical flow uncertainties, and scene occlusion information**, we design an **adaptive blending mechanism** to generate high-quality, artifact-free super-resolution results.

#### Minimizing Domain Differences through Real Scene Inputs
In reference super-resolution (RefSR) tasks, collecting perfectly aligned W/T image pairs as training data is very challenging. Therefore, previous research has explored two possible but suboptimal solutions:
1. **Using the reference image T as the training target** (e.g., Wang et al. 2021 and Zhang et al. 2022a), which may transmit defects from the reference image or lead the network to learn identity mapping.
2. **Synthesizing low-resolution inputs from target images using degradation models** (e.g., Trinidad et al. 2019 and Zhang et al. 2022a), but this method introduces domain differences between training and inference stages, thereby reducing super-resolution performance on real scene images.
To avoid learning identity mappings and minimize domain differences, we adopt a design where, during training, a second smartphone of the same model mounted on the photography platform synchronously captures additional T images as references (see Figure 6).
![povIB4GXtGm5Ljl](https://i.imgur.com/fbmyRgT.png)

With this design, the fusion model uses real W images as input during both training and inference, eliminating domain differences. Additionally, the reference and target images are captured by T cameras from different devices to prevent the network from learning identity mappings. Compared to existing dual zoom RefSR datasets, our design has significant advantages:
- Some datasets exhibit strong temporal motion between W and T (e.g., Wang et al. 2021);
- Some datasets are limited to static scenes (e.g., Wei et al. 2020).
We have collected a large-scale dataset containing high-quality W/T synchronized data, covering dynamic scenes, including portraits, architecture, landscapes, as well as challenging scenes with moving objects and nighttime scenarios. Experiments show that our method outperforms current state-of-the-art methods on both existing dual zoom RefSR datasets and our new dataset.

### Model/Algorithm
When the user adjusts the zoom to a medium range (e.g., 3-5 times) and presses the shutter button, the system captures a pair of synchronized images. The processing flow is as follows:
#### **Image Alignment**: 
First, we perform **global coarse alignment through keypoint matching**, followed by **local dense alignment using optical flow** (see Section 3.1).
	**Coarse Alignment:**  
		First, we crop the image from the wide-angle camera (W) to match the field of view (FOV) of the telephoto camera (T). Next, we use bicubic interpolation to adjust the spatial resolution of W to match T (4k×3k). Then, we estimate the global 2D translation vector using the FAST feature point matching algorithm (Rosten and Drummond 2006) and adjust the cropped W to obtain the adjusted image \\( I_{src} \\).
	**Dense Alignment:**  
		We use PWC-Net to estimate the dense optical flow between \\( I_{src} \\) and \\( I_{ref} \\). It is important to note that the average offset between W and T at 12MP resolution is about 150 pixels, which is much larger than the motion range in most optical flow training data. Therefore, the optical flow estimated from 12MP images is often very noisy. 
		To improve accuracy, we downsample \\( I_{src} \\) and \\( I_{ref} \\) to a **smaller resolution of 384×512 to predict** the optical flow, then **upsample the flow to the original resolution** and deform \\( I_{ref} \\) to obtain \\( \tilde{I}_{ref} \\) using bilinear resampling. This method provides more accurate and robust optical flow estimates at a smaller scale.
	**Optimization**
		To accommodate the computational resource limitations of mobile devices, we removed the DenseNet structure from the original PWC-Net, reducing the model size by **50%**, latency by **56%**, and peak memory by **63%**. Although the endpoint error (EPE) of optical flow increased by **8%** on the Sintel dataset, the visual quality of the optical flow remains similar.  
		Additionally, we generate an occlusion map \\( M_{occ} \\) using forward-backward consistency checks (Alvarez et al. 2007) to identify areas occluded during alignment.
#### **Image Fusion**:  
We use UNet (Ronneberger et al. 2015) to fuse the brightness channel of the cropped image from the wide-angle camera (W) with the deformed reference image from the telephoto camera (T) (see Section 3.2).

Using the source image, the deformed reference image, and the occlusion mask as inputs
**Fusion Process:**  
To maintain color consistency in the W image, we perform fusion only in the luminance space. The specific implementation is as follows:
1. **Input Preparation:**
    - Convert the source image \\( I_{src} \\) to a grayscale image, denoted as \\( Y_{src} \\).
    - Convert the deformed reference image \\( I_{ref} \\) to a grayscale image, denoted as \\( \tilde{Y}_{ref} \\).
    - Input the occlusion mask \\( M_{occ} \\).
2. **Fusion Network:**
    - We construct a 5-layer UNet network, using \\( Y_{src} \\), \\( \tilde{Y}_{ref} \\), and \\( M_{occ} \\) as inputs to generate the fused grayscale image \\( Y_{fusion} \\).
    - The detailed architecture of this UNet is provided in the appendix.
3. **Color Recovery:**
    - Combine the fused grayscale image \\( Y_{fusion} \\) with the UV color channels of \\( I_{src} \\).
    - Convert back to RGB space to generate the fused output image \\( I_{fusion} \\).
#### **Adaptive Blending**:  

Although machine learning models have strong capabilities in image alignment and fusion, mismatches between W and T can still introduce noticeable artifacts in the output. These mismatches include:

1. **Depth of Field differences**
2. **Occluded pixels**
3. **Warping artifacts during the alignment stage**

To address these issues, we propose an adaptive blending strategy that combines **alpha masks** derived from **defocus maps**, **occlusion maps**, **optical flow uncertainty maps**, and **alignment rejection maps** to adaptively blend \\( Y_{src} \\) and \\( Y_{fusion} \\). The final output image eliminates significant artifacts and maintains high robustness in cases where pixel-level consistency issues exist between W and T.
##### The Narrow Depth of Field Issue of Telephoto (T)
We observe that the telephoto camera (T) on mobile devices typically has a narrower depth of field (DoF) than the wide-angle camera (W). This is because depth of field is proportional to the square of the f-stop number and focal length. Typically, the focal length ratio between T and W is greater than 3, while the f-stop ratio is less than 2.5. Therefore, the depth of field of T is usually significantly narrower than that of W. Thus, it is necessary to exclude defocused pixel areas using the defocus map to avoid introducing artifacts. Estimating the defocus map from a single image is a pathological problem that usually requires complex and computationally intensive machine learning models. To this end, we propose an efficient algorithm that reuses the optical flow information computed during the alignment stage to generate the defocus map:

- The optical flow information contains depth and motion features of the image. By analyzing the relationship between optical flow and scene depth, we can efficiently generate the defocus map without additional complex models.

![xwEY9I7mbXrkEog](https://i.imgur.com/8n90i6w.png)

##### Defocus Map Generation Method
To estimate the defocus map, we need to determine two key pieces of information:
1. **Camera's Focus Position**: the center area of the image that is in sharp focus.
2. **Relative Depth of Each Pixel with Respect to the Focus Position**.
Since the FOV of W and T is approximately parallel (fronto-parallel), and the magnitude of optical flow is proportional to the camera disparity, which is related to scene depth, we design an efficient optical flow-based defocus map estimation algorithm, as follows (see Figure 5):
- **Obtain the Focus Region of Interest (ROI)**:
    - Obtain the focus area from the camera's auto-focus module. This module provides a rectangular area representing the region where most pixels in the T image are sharply focused (i.e., the focus ROI).
- **Estimate the Focus Position \\( x_f \\) based on Optical Flow**:
    - Using dual-camera stereo vision, treat optical flow as a proxy for scene depth. Assuming that in a static scene, pixels located in the same focal plane have similar optical flow vectors (Szeliski 2022).
    - Apply the **k-means clustering algorithm** to the optical flow vectors within the focus ROI to determine the area with the highest clustering density (i.e., the maximum cluster).
    - The cluster center is defined as the focus position \\( x_f \\).
- **Estimate Relative Depth of Pixels and Generate the Defocus Map**:
    - Calculate the Euclidean distance (L2 distance) between the optical flow vector of each pixel and the optical flow vector at the focus position \\( x_f \\).

##### Defocus Map Calculation Formula
The calculation formula for the defocus map \\( M_{defocus}(x) \\) is as follows:
\\(
M_{defocus}(x) = \text{sigmoid} \left( \frac{\| F_{fwd}(x) - F_{fwd}(x_f) \|_2^2 - \gamma}{\sigma_f} \right)
\\)
where:
-  \\( F_{fwd}(x) \\): the forward optical flow vector of pixel \\( x \\).
- \\( F_{fwd}(x_f) \\): the forward optical flow vector at the focus position \\( x_f \\).
-  \\( \gamma \\): a threshold that controls the tolerance range.
-  \\( \sigma_f \\): a parameter that controls the smoothness of the defocus map.

##### Occlusion Map Calculation Formula
The occlusion map \\( M_{occ}(x) \\) is calculated based on forward-backward optical flow consistency, as follows:
\\(
M_{occ}(x) = \min\left(s \cdot \| W(W(x; F_{fwd}); F_{bwd}) - x \|_2, 1 \right)
\\)
where:

- \\( W \\): bilinear warping operator, used to map image coordinates \\( x \\) to new positions based on optical flow.
- \\( F_{fwd} \\): forward optical flow from the source image \\( I_{src} \\) to the reference image \\( I_{ref} \\).
- \\( F_{bwd} \\): backward optical flow from the reference image \\( I_{ref} \\) to the source image \\( I_{src} \\).
- \\( s \\): a scaling factor used to adjust the sensitivity of optical flow differences.
- \\( x \\): 2D image coordinates on the source image.

##### Optical Flow Uncertainty Map
Due to the inherently ill-posed nature of dense correspondence problems, we enhance the functionality of PWC-Net to output an optical flow uncertainty map, which describes the uncertainty of each pixel's optical flow prediction. The method is as follows:
1. **Uncertainty Modeling**:  
    The enhanced PWC-Net predicts a multivariate Laplacian distribution for the optical flow vector of each pixel, rather than a single point estimate. It predicts two additional channels representing the log-variance of optical flow in the x and y directions, denoted as \\( {Var}_x \\) and \\( {Var}_y \\).
    
2. **Convert to Pixel Units**:  
    The log-variance is converted to pixel units using the following formula—here, \\( S(x) \\) is the optical flow uncertainty value for each pixel.
\\(
S(x) = \sqrt{\exp(\log(\text{Var}_x(x))) + \exp(\log(\text{Var}_y(x)))}
\\)
3. **Generate Optical Flow Uncertainty Map**:  
\\(
M_{\text{flow}}(x) = \frac{\min(S(x), s_{\text{max}})}{s_{\text{max}}}
\\)
The optical flow uncertainty map typically has higher values in object boundaries or texture-less regions.

##### Alignment Rejection Map
To exclude artifacts introduced by **alignment errors**, we generate an alignment rejection map by comparing the local similarity between the source image and the aligned reference image. The method is as follows:
- **Match Optical Resolution**:
    - Use bilinear interpolation to adjust the resolution of the aligned reference frame \\( \tilde{Y}_{\text{ref}} \\) to match the optical resolution of W, resulting in a downsampled image \\( \tilde{Y}_{\text{ref}}^\downarrow \\).
- **Calculate Local Differences**:
    - For local patches \\( P_{\text{src}} \\) from the source image and \\( P_{\tilde{\text{ref}}} \\) from the aligned reference image:
        1. Subtract the mean of the patches \\( u_{\text{src}} \\) and \\( u_{\text{ref}} \\).
        2. Calculate the normalized difference:
\\(
P_\delta = (P_{\text{src}} - \mu_{\text{src}}) - (P_{\tilde{\text{ref}}} - \mu_{\text{ref}})
\\)
	- Generate the alignment rejection map
\\(
M_{\text{reject}}(x) = 1 - \exp\left(-\frac{\|P_\delta(x)\|_2^2}{\sigma_{\text{src}}^2(x) + \epsilon_0}\right)
\\)
#### Final Blending

\\(
M_{\text{blend}} = \max(1 - M_{\text{occ}} - M_{\text{defocus}} - M_{\text{flow}} - M_{\text{reject}}, 0)
\\)
The final output image is generated through alpha blending and cropped back to the full W image:
\\(
I_{\text{final}} = \text{uncrop}\left(M_{\text{blend}} \odot I_{\text{fusion}} + (1 - M_{\text{blend}}) \odot I_{\text{src}}, W\right)
\\)
By combining multiple masks (such as occlusion maps, defocus maps, optical flow uncertainty maps, and alignment rejection maps), we ensure high quality and robustness of the fusion results, avoiding the introduction of artifacts, blurriness, or alignment errors.

### Implementation Details
## Results and Evaluation
### Experiments: 
### Results
### Comparisons
![WM6ILjWeYsM35Sc](https://i.imgur.com/rlLj4d0.png)

![X8MjkLL7M9laHpI](https://i.imgur.com/NjA4JmF.png)

## Critical Analysis
### Strengths
### Weaknesses
### Insights
## Broader Impact and Future Work
### Broader Implications
### Open Questions
### Future Work

## Personal Reflection
### Relevance to My Work
### Practical Applications
### Key Learnings

## Related Work and Context
### Prior Work
**Learning-based Single Image Super-Resolution (SISR)**  
Over the past decade, various methods (e.g., Christian Ledig 2017; Dong et al. 2014; Kim et al. 2016; Lai et al. 2017; Wang et al. 2018; Xu et al. 2023; Zhang et al. 2019a, 2022b, 2018) have demonstrated outstanding results in the field of single image super-resolution. However, due to the severely ill-posed nature of this task, these methods tend to generate blurry details under **large upsampling factors (e.g., 2-5 times, common in smartphone hybrid zoom)**. Additionally, some methods are only applicable to specific domains, such as face super-resolution (Chan et al. 2021; Gu et al. 2020; He et al. 2022; Menon et al. 2020).

**Reference Super-Resolution (RefSR) Based on Internet Images**  
RefSR generates high-resolution results from low-resolution inputs and one or more high-resolution reference images (Pesavento et al. 2021). Traditional RefSR methods assume that reference images come from the internet (Sun and Hays 2012) or are captured at different times, locations, or camera models during the same event (Wang et al. 2016; Zhang et al. 2019b), focusing on improving dense alignment between source and reference images (Huang et al. 2022; Jiang et al. 2021; Xia et al. 2022; Zheng et al. 2018) or enhancing robustness to unrelated reference images (Lu et al. 2021; Shim et al. 2020; Xie et al. 2020; Yang et al. 2020; Zhang et al. 2019b).  
In contrast, we alleviate the alignment challenge by synchronously capturing W and T images, avoiding alignment issues caused by object motion.

**Reference Super-Resolution (RefSR) Based on Auxiliary Cameras**  
Recent RefSR studies (Trinidad et al. 2019; Wang et al. 2021; Zhang et al. 2022a) have captured reference images of the same scene using auxiliary cameras. However, due to the lack of pixel-aligned inputs and real label image pairs, PixelFusionNet (Trinidad et al. 2019) synthesizes low-resolution inputs from high-resolution reference images using degradation models and trains using pixel-level losses (e.g., l1 and VGG losses). However, this model performs poorly when faced with **real scene inputs**, primarily due to domain differences between training and inference stage images.

On the other hand, SelfDZSR (Zhang et al. 2022a), DCSR (Wang et al. 2021), and RefVSR (Lee et al. 2022) treat reference images as targets for training or fine-tuning. We observe that this training setup is prone to getting stuck in degraded local minima: the model often learns **identity mappings**, merely copying the content of T images to the output. This leads to severe alignment errors, color shifts, and depth of field mismatches, which are unacceptable in practical photography.

To address these issues, we additionally capture a T image during training to alleviate the aforementioned problems and improve training robustness.

**Efficient Mobile Device Reference Super-Resolution (RefSR)**  
Existing methods often consume significant memory due to the use of attention mechanisms/Transformers (e.g., Wang et al. 2021; Yang et al. 2020) or deep network architectures (e.g., Zhang et al. 2022a). These methods may encounter out-of-memory (OOM) issues even on a desktop GPU with 40GB RAM when processing 12MP input resolutions, making them **impractical for mobile devices**. In contrast, our system processes 12MP inputs on mobile GPUs in just 500 milliseconds, using only 300MB of memory.

Our system design is inspired by reference image deblurring methods for faces [Lai et al. 2022], but the problems we address are more challenging:

1. **Super-Resolution for General Scenes**: We apply super-resolution to general images rather than focusing solely on faces. Therefore, our system needs to be more robust to diverse scenes and capable of handling various imperfections and mismatches from both cameras.
2. **Domain Differences in Training Data**: Unlike face deblurring models that can learn from synthetic data, image super-resolution models are more sensitive to domain differences in training data. Additionally, collecting real training data for reference-based super-resolution tasks is more challenging. Thus, our proposed adaptive blending method and dual smartphone platform design become key distinctions from the method in [Lai et al. 2022].

### Historical Context

## References
