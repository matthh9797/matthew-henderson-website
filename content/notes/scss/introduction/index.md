---
title: Introduction
weight: 10
menu:
  notes:
    name: Introduction
    identifier: notes-scss-intro
    parent: notes-scss
    weight: 10
---
<!-- Introduction -->
{{< note title="Introduction" >}}

SASS is a styling framework which sits on top of CSS, which allows you to use loops, variables ... 

Every css file is a valid scss file (i.e. if you have a bunch of css files already you can simply convert them to .scss files to get started).

{{< /note >}}

<!-- Installation on Mac -->
{{< note title="Installation on Mac" >}}

SASS is written in Ruby, hence you need to download Ruby to use SASS. The two programmes required to use SASS are Ruby and Ruby Gems (*a package manager for Ruby*).

On a Mac both Ruby and Ruby Gems are pre-installed. To check these are already installed run the below code on the terminal and if they return a version number you are ready to install SASS.

```bash
ruby -v
gem -v
```

Follow the [SASS Install](https://sass-lang.com/install) instruction to install SASS. Then check it has worked by running `sass --version`.

Issues (2022-01-18): I also had to install developer tools on the mac by running `xcode-select --install` 

{{< /note >}}


<!-- Using SCSS -->
{{< note title="Using SCSS" >}}

Worth noting that SCSS and SASS are a slightly different syntax but can be used for the exact same things. I prefer SCSS since any .css file is a valid .scss file.

To compile your scss files into css use the following in the terminal. 

```bash
sass --watch <NAME_OF_FOLDER> (i.e. style)
```

Then import the resulting .css file into your html document.

{{< /note >}}
