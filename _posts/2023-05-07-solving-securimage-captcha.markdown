---
layout: post
title:  "Solving Securimage Captcha"
date:   2023-05-07 13:23:27 +0700
categories: blog
author: Reynaldi Chernando
---
CAPTCHA (Completely Automated Public Turing test to tell Computers and Humans Apart) is a tool used to differentiate between real users and bots or automated software, commonly found when submitting a form in a website. [Securimage](https://github.com/dapphp/securimage){:target="_blank"} (aka PHP Captcha) is one of them, it was one of the popular captcha library for PHP. I first encountered it while using the [JNE website](https://www.jne.co.id/id/beranda){:target="_blank"} and felt like it was a challenge for me on how to solve the captcha programmatically.

![Securimage Captcha](https://imgur.com/eGYhcvF.jpeg)

*Securimage Captcha*


## An OCR Approach
Fast forward to recently, I found an article about someone [indexing an entire internet memes](https://findthatmeme.com/blog/2023/01/08/image-stacks-and-iphone-racks-building-an-internet-scale-meme-search-engine-Qzrz7V6T.html){:target="_blank"}, and they are using Apple OCR with racks of iPhone to extract the images to text, as the iPhone is able to pick up the jumbled text. This got me to try and see if the iPhone strategy also works for the securimage captcha, and sure enough it did!

![The iPhone picked up the captcha as text](https://imgur.com/Y4E2MqI.jpeg)

*The iPhone picked up the captcha as text*

Now I don’t have racks of iPhone lying around to use, instead I look for alternative way to use this Apple OCR technology, with a simple Google search I found that it is called [Apple Vision framework](https://developer.apple.com/documentation/vision){:target="_blank"}, and I see that it is supported on MacOS, which I do have. Since I recently did a Hackintosh setup with the intention of making an iPhone app, which to this day I haven’t started, but regardless, the Hackintosh finally proves its usefulness.

Back to the Vision framework, it is using Objective-C language, which I don’t have any experience with, but turns out someone already made this [Apple Vision framework implementation in Python](https://gist.github.com/RhetTbull/1c34fc07c95733642cffcd1ac587fc4c){:target="_blank"}, using library called [PyObjC](https://pyobjc.readthedocs.io/en/latest/){:target="_blank"}, a bridge for Python and Objective-C, which is perfect for my use case, I just need to modify it a bit to make it work.


![Solving captcha using Apple Vision Framework](https://imgur.com/zZboJse.jpeg)

*Solving captcha using Apple Vision Framework*

## Creating the Dataset
Other than the OCR code secured, I also want to check on how accurate it is, and to do that, I need to have some amount of dataset of the captcha from the website. I already knew that it was using securimage by looking at Chrome DevTools. So I forked the securimage repository, setup PHP 7.4 (which is not supported anymore on homebrew by the way), and then configured it to work with the securimage.

![Inspecting the image source](https://imgur.com/je0ouIU.jpeg)

*Inspecting the image source*

![Default securimage running on local machine](https://imgur.com/fX8dS0h.jpeg)

*Default securimage running on local machine*

Right off the bat, I can see that the captcha generated is different than the one from the website, as this one has more complexity, and the website version is simpler (which is good for me). So I have to make some adjustment to the code to try and mimic the looks to be similar with the website. I also need to obtain the actual value for the captcha, so I did just that.

![Adjusted the captcha to be similar with JNE setup](https://imgur.com/KzoOwVP.jpeg)

*Adjusted the captcha to be similar with JNE setup*

![Endpoint displaying the current captcha code](https://imgur.com/ItpkbrM.jpeg)

*Endpoint displaying the current captcha code*

And it is now looking much better. To generate the data, I wrote a simple Python script to do the job, and I decided on 10.000 images for no particular reason, with the format [captcha_code].png.

![The dataset](https://imgur.com/OaqUvb2.jpeg)

*The dataset*

## First Try
Now that I have the dataset, I can start testing the performance of the Apple Vision framework. And the results are

![Result from using Apple Vision framework](https://imgur.com/VjuHfvA.jpeg)

*Result from using Apple Vision framework*

...underwhelming, it shows 44.5% accuracy with the total time for 10k images of around 16 minutes, averaging around 96ms. The speed is actually impressive, considering I am not even on a real Mac. I have an i5 CPU and 8GB RAM. I suppose I could just do some retries until the result is correct, it might only require several attempts (of different captcha). But still, the accuracy is too low, and I was devastated with the result.

## Realization
During working on this I also thought to myself, if I were to deploy this on a server, this would be challenging since PyObjC is only supported on Apple devices, and most servers are like Linux. So there is actually no way I’m going with the Apple Vision framework. Thus, I looked at the next obvious thing, which is OCR using deep learning. I found several options, but ended up with only 2 that I’m going to use, which are [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR){:target="_blank"} and [PARSeq](https://github.com/baudm/parseq){:target="_blank"}.

First is PaddleOCR, setting it up is simple enough as the documentation is very clear, I can adapt the code to be similar to the one from my Apple Vision code. From my limited knowledge while doing this project, in an OCR setup there are mainly 2 aspects, which are the text detection and text recognition. They are as the name suggests, text detection is for detecting text in an image as in it will crop out the text section of the image, and text recognition is to extract the actual text from the 'cropped' out image.

In my use case however, the securimage captcha is already 'cropped' out, as there is nothing else in the image other than the text. This is why my PaddleOCR setup is only using the text recognition model (it is called [SVTR](https://github.com/PaddlePaddle/PaddleOCR/blob/release/2.6/doc/doc_en/algorithm_rec_svtr_en.md){:target="_blank"}), and not the text detection nor the text alignment (this is unique to PaddleOCR, it rotate and aligns the image appropriately). And the results are 33.4% accuracy and 11 minutes for 10k images, somehow it performs worse than Apple OCR. 

![Result from using PaddleOCR (SVTR)](https://imgur.com/1Jn7qhw.jpeg)

*Result from using PaddleOCR (SVTR)*

Now, I’m not saying it is bad in any way, if anything maybe I missed something, or the model is not suited for this use case.

Second is PARSeq, which is used for scene text recognition. Setting it up is also super simple since it already has a working code at their GitHub page. Testing it shows promising result, the base model of PARSeq resulted 67.3% accuracy and 23 minutes for 10k images, not too shabby.

![Result from using PARSeq base](https://imgur.com/OGF7APa.jpeg)

*Result from using PARSeq base*

It also has a way of fine tuning to a custom dataset, which is perfect. However it is kind of hard since there is no guide for this, and I have to rely on issues other people made in their GitHub page. Eventually I figured it out, first, I need to prepare my dataset, in which I have to split it to train, test, validation sets, and then convert it into what’s called a [LMDB](http://www.lmdb.tech/doc/){:target="_blank"} (Light Memory-Mapped Database) format, after that I have to adjust the [Hydra](https://hydra.cc/){:target="_blank"} config and only then I can start training. 

![LMDB setup](https://imgur.com/9q0KcAs.jpeg)

*LMDB setup*

![Hydra config](https://imgur.com/izuDNrA.jpeg)

*Hydra config*

I have to use Google Colab for the fine tuning, since it requires a GPU, and my Hackintosh doesn’t have one. After waiting a while for the training, and then testing, I finally did it. The result shows 99.8% accuracy! I can then continue by saving the PyTorch checkpoint file which is the output for fine tuning the model.

![Result after fine tuning PARSeq](https://imgur.com/u77Tiug.jpeg)

*Result after fine tuning PARSeq*

I am aware that there might be a possibility where it is crazily overfitted, so just to make sure I quickly created a new set of data, from my earlier script, and then to test with the checkpoint file, and the result is consistent, also 99.8%, with 23 minutes for 10k images.

![Result from using fine tuned PARSeq base](https://imgur.com/FFpcuFy.jpeg)

*Result from using fine tuned PARSeq base*

We can go even better than this, the PARSeq model has a tiny version, which I would assume would be faster in terms of inference (fancy word the researchers are using for prediction). So I redid the training with the tiny model, and result shows that it still has 99.5% accuracy, but faster at 12 minutes for 10k images, that is roughly 70ms per image. Which is perfect, I finally have a solution for solving the captcha.

![Result from using fine tuned PARSeq base](https://imgur.com/PAMf2x2.jpeg)

*Result from using fine tuned PARSeq tiny*

Lastly, to demonstrate how someone might automate a process of retrieving data from the JNE website, I created a simple server which scrapes the site for shipping costs based on origin and destination.

I will spare you the detail, here are the steps:
1. Start session, open [JNE tariff page](https://www.jne.co.id/id/tracking/tarif){:target="_blank"}
2. Fetch the captcha image, perform captcha solving and receive code
3. Take XSRF token from page using BeautifulSoup
4. Directly POST request to the JNE endpoint, with the parameters
5. Parse the HTML table into dictionary
6. Return the dictionary as JSON

![Making a request to the /tariff endpoint](https://imgur.com/d38uHYi.jpeg)

*Making a request to the /tariff endpoint*

And it works! Latency is not too bad, at around 400ms, and the response is nicely formatted using JSON.

You can find the [online demo](https://jne-api-demo-4kwyrsebtq-as.a.run.app/apidocs){:target="_blank"} here, I have setup a simple Swagger documentation for the API. Note that it might take a while to load because of the cold start.

## Conclusion
At this day and age, using these types of captcha probably won’t work anymore. If you are still using it, you may want to consider more advanced ones like Google reCaptchaV3 or Cloudflare Turnstile. Just don’t use the ones like [this](https://www.reddit.com/r/softwaregore/comments/rtgrw8/microsoft_captcha_ive_been_trying_to_create_an/?utm_source=share&utm_medium=web2x&context=3){:target="_blank"}.

Thank you for reading!

All the code used for this project are available in these repositories:
- [https://github.com/reynaldichernando/securimage](https://github.com/reynaldichernando/securimage){:target="_blank"}
- [https://github.com/reynaldichernando/securimage-dataset](https://github.com/reynaldichernando/securimage-dataset){:target="_blank"}
- [https://github.com/reynaldichernando/securimage-solver](https://github.com/reynaldichernando/securimage-solver){:target="_blank"}
- [https://github.com/reynaldichernando/parseq](https://github.com/reynaldichernando/parseq){:target="_blank"}
- [https://github.com/reynaldichernando/jne-api](https://github.com/reynaldichernando/jne-api){:target="_blank"}
