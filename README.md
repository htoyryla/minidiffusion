## Experimenting with image making using denoising diffusion

Based on the excellent and clear codebase by @lucidrains

Let's call this miniDiffusion.

This is an experimental project with emphasis on visual work. I would not mind if somebody trained a really cool model,
but in general it is best to try in small scale. Even 200 images and a few hours of training. Select training images based on visual features (how they look) instead of semantics (what things there are). This is not a classification project. See discussions.

Model architectures unetcn0 (Unet with convnext) and unet0 (unet with resnet) are known to work. Both copied from commit history of @lucidrains repo

My intention is

* add scripts for convenient training and image generation / transformation
* experiment with image transformation using diffusion
* especially training with small homogenous datasets that can be quickly trained and retrained
* and use the trained models as "brushes" or "filters" 
* starting from seed images and using various means of guidance (text/CLIP, texture/VGG+GRAM etc) 
* experiment with various model structures

Some early results, using datasets of a few hundred images, selected according to visual and not semantic criteria. Using ConvNext based Unet, not the current version but from before the recent changes, training at 512px for 1 - 2 days. It also appears that once we have a pretrained model, it can quite fast be retrained with a visually different set. BTW, I have not succeeded to get the newest Unet/Convnext to train satisfactorily (but it might just be that I have not yet found a correct lr)

<p float="left">
<img src="https://user-images.githubusercontent.com/15064373/163939108-f131d71d-384a-4211-95ab-573e1cfe33ee.jpeg" width="360"></img>
<img src="https://user-images.githubusercontent.com/15064373/163939115-adf16ff6-8ef7-4003-98c0-6cfc604a44c3.jpeg" width="360"></img>
<img src="https://user-images.githubusercontent.com/15064373/163939127-acac4f49-7a42-4c81-928a-5d8d56998a22.png" width="360"></img>
<img src="https://user-images.githubusercontent.com/15064373/163940103-836b4e52-57e5-4375-b292-3d538f1e3f6d.jpeg" width="360"></img>
</p>

## Important

Do not do 
```
pip install denoising_diffusion_pytorch 
```

as it will install lucidrains' version. Minidiffusion uses its own version directly from the subfolder. 

## How to train

Using unetcn0 and unet0 I have been able to train useful models with as few as 250 images, using 512px imagesize and 6+6 layers unet.

```
python diffutrainer.py --images path_to_your_image_folder --lr 5e-5 --steps 1000 --accum 10 --dir output_folder --imageSize 512 --batchSize 2 --saveEvery 100 --nsamples 2 --mults 1 1 2 2 4 8 --model unet0

--lr learning rate
--steps diffusion steps
--accum accumulate gradient over x iters
--dir output_folder 
--imageSize native image size for the model to be trained
--batchSize  
--saveEvery save and sample at given intervals
--nsamples how many samples to generate when saving, small number can save much time
--mults multipliers affecting number of feature maps per level, use larger numbers for the later, more abstract levels
        number of multipliers also determines depth of the model, good rule is to  have 5 or 6 numbers of 256 and 6 or 7 numbers for 512
        too few layers will make model fail to perceive the image as a whole
--model unetcn0 (selects which model architecture is used, to be explained later, this one at least works)
```
## How to sample

Currently supports CLIP guidance and use of target image. Use textw for tuning how much effect CLIP has and ssimw to guide target image weight. SSIM is used for target image loss.

```
python diffudiver.py --text prompt_for_clip --dir output_folder --name basename_for_stored_images --tgt_image path_to_target_image --lr 0.002 --imageSize 1024 --modelSize 512 --load path_to_stored_model --mults same_as_in_training --ema --saveEvery 50 --saveAfter 550  --model  unet0 --ssimw 1 --textw 0.02

--lr 0.002 rate of change during optimization, experimetn
--imageSize 1024 generated image size, multiple of 32
--modelSize 512 native size of the model
--ema  use ema checkpoint (you can test what difference does it make)
--saveEvery 50   saving of frames during iteration
--saveAfter 550  start saving only after given step
--model  unetcn0 as in training
--mults as in training

--show 

--text a text prompt
--textw text guidance weight
--tgt_image target image for structural guidance
--ssimw structural guidance weight
```

There is also an experimental option to start from a noised seed image instead of pure noise. Use --image for the seed image and set --mul slightly above 1. You can also "weaken" the seed image using values of --weak less than 1.   

## Better CLIP guidance

Diffudiver is not ideal for CLIP guidance, it needs less memory than proper guidance, but image quality can be poor unless one finds just the correct balance between the different weights.

Diffumaker has now been added and it does proper guidance.

Example

```
python diffumaker.py --text "an abstract painting of an alley in an old town" --lr 500 --dir k022a --name mq314y --textw 5 --cutn 48 --low 0.4  --imageSize 1024 --modelSize 512 --image path_to_init_image --tgt_image path_to_target_image   --saveEvery 50 --saveAfter 50 --load path_to_trained_model --model unet1 --mults 1 1 2 2 4 4 8  --ssimw 100  --ema --showLosses --spher  --skip 20 --weak 0.45 --mul 1.3 --mid 0
```

Options weak, mul and mid help in tuning the levels of init image and noise. This is a bit complex topic, so better return to it in discussions.
