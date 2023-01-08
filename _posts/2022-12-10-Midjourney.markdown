---
layout: post
title:  "Midjourney"
date:   2022-12-10 10:15:00 +0000
categories: graphics
tags: graphics midjourney ai
---
Creating pictures using Midjourney can be a lot of fun and could even be somewhat useful. E.g. all pictures in this website has been created using midjourney.



# Recommendations for prompting in Midjourney
## Use "seed"
To be able to precisely work with your image, you should specify the seed. This ensures that if you generate the image again with the same prompt, you will get more or less the same result. With this, you can start changing the prompt step by step to tweak it to what you want

* /imagine prompt: some example --seed 999

You can also go to a picture you or someone else generated and retrieve the prompt and seed. Now you can use the seed to continue tweaking the image
* Click the reaction button on the original post in discord
* Select "envelope" (you might need to search for it)
* Midjourney bot will send you a DM with the seed number

One more tip is to set the seed to a static number so that you do not have to add / find it all the time. Do this by using the /prefer suffix command
* /prefer suffix --seed 1234

## Set aspect ratio
The standard aspect ratio is 1:1. You can change this to other ratios, but at the time of writing, you can only use 3:2 / 2:3 / 1:1 in midjourney v4
* /imagine ... --ar 3:2

Note that you can set a aspect ratio as default using the /prefer suffix command
* /prefer suffix --ar 3:2
(Note that if you want more suffixes, e.g. also setting the seed you need to add them all in the same command like /prefer suffix --ar 3:2 --seed 1234)

## Create your own styles
You can create your own styles and save them for easier use
* /prefer option set [NAME-OF-STYLE][STYLE-PROMPT]
* **Example:** /prefer option set character ::5 Character Concept Art::4 creative, expressive, detailed, colorful, stylized anatomy, digital art, 3D rendering, unique, award-winning, Adobe Photoshop, 3D Studio Max, V-Ray, professional, glibatree style, well-developed concept, distinct personality, consistent style::3 deformed, simple, undeveloped concept, generic personality, inconsistent style::-2

(Make sure to press tab + select "value" before entering the STYLE-PROMPT)

To use the style, just add --[NAME-OF-STYLE] at your prompt
* **Example:** /imagine an ugly toad --character
(If using more arguments, you need to add them at the end)

You can find a number of nice styles here:
[Glibatree AI Art /prefer option set](https://pastebin.com/RhGZ7B9s)

--

## Example prompts
Below some example prompts
```
some prompt examples...
```

# Some examples
## Me myself and I
![image](https://cdn.midjourney.com/be41c889-3dd6-43b3-b7a7-182b1abb7c22/grid_0.png)
![image](https://cdn.midjourney.com/854da033-06ff-4640-af4b-52aa1393a9b7/grid_0.png)
![image](https://cdn.midjourney.com/f3319201-64db-4d91-8221-cf0c6f66c44e/grid_0.png)
![image](https://cdn.midjourney.com/c2e6c429-0f7f-4bca-b1e1-23365d1785d8/grid_0.png)
![image](https://cdn.midjourney.com/ced968ad-2842-433c-b9ff-88e7a6e1583c/grid_0.png)
![image](https://cdn.midjourney.com/9b5ab857-7fda-49bd-b5dc-33ff0989dc19/grid_0.png)
![image](https://cdn.midjourney.com/cc47c65b-6fc0-4ea0-ad4a-0e46588c88c2/grid_0.png)
![image](https://cdn.midjourney.com/5bd1c8e8-bbf0-4d98-8515-54225528aef0/grid_0.png)
![image](https://cdn.midjourney.com/54d5dbe9-382b-4adb-a35a-3b1c39324a38/grid_0.png)

## Batman
![image](https://cdn.midjourney.com/b92185f4-2580-4566-b19e-348a15b8e9f7/grid_0.png)
![image](https://cdn.midjourney.com/492becae-eebc-44da-9bf6-b8a6475e7ddd/grid_0.png)
![image](https://cdn.midjourney.com/5484fe26-8952-4abb-bcb0-09d22a28dfbc/grid_0.png)
![image](https://cdn.midjourney.com/8671a6d5-a1ff-43ae-a584-e0d06e5aeb7c/grid_0.png)