---
title: "Google Drive Source Image"
date: 2023-03-27T09:31:44+07:00
description: How to use Google Drive as a source for Markdown images
menu:
  sidebar:
    name: Src Image GDrive
    identifier: gdrive-src-img
    parent: web-development
    weight: 20
hero: images/forest.jpg
tags: ["Markdown","Google Drive"]
categories: ["Basic"]
---


This is a quick tutorial which shows you how to use images from your google drive on a website.

First create a new folder on your Google Drive if you don't have one already. Usually, when I create a website I like to have a folder insite my website directory called "Static" with all of the images for the site. I will make this folder public to avoid repitition. 

Next, make sure that the folder is shared publicly. Right click on the folder and click "Share". Then, share with "Anyone with the link" as shown below.

![Make Google Drive folder public](Https://drive.google.com/uc?export=view&id=12qUMZIW3rTFGs_LYkRo0dGEioGnfEvbS)

Next, add an image to the folder if you haven't got one there already. Right click the image and click "Share" again. This time click "Copy Link" to copy a link to the image to your clipboard.

![Make Google Drive folder public](Https://drive.google.com/uc?export=view&id=12uYLvlIY8xfpJRYVkGIJV2b7FXuXuJn_)

Next, paste the image link into a text editor. You will have a link in the form of:

> https://drive.google.com/file/d/**<IMAGE_ID>**/view?usp=sharing

To get a shareable link to the image you will need to copy the <IMAGE_ID> part of the link and paste it into the following URL:

> Https://drive.google.com/uc?export=view&id=**<IMAGE_ID>**

For example, to link to the above image on markdown you can use the following link. 

    ![Alt text](Https://drive.google.com/uc?export=view&id=12uYLvlIY8xfpJRYVkGIJV2b7FXuXuJn_)

And that's all, it's not the prettiest but it works. Thanks for reading!