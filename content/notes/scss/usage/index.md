---
title: Usage
weight: 20
menu:
  notes:
    name: Usage
    identifier: notes-scss-usage
    parent: notes-scss
    weight: 20
---

<!-- Variables -->
{{< note title="Variables" >}}

Variables are intuitive as in any other programming language. The syntax for variables is like the below.

```scss
$my-color = "blue"

.my_class {
  color: $my-color;
}
```
{{< /note >}}

<!-- Nesting -->
{{< note title="Nesting" >}}

This is also intuitive but a very useful feature of scss. You can nest css classes. This is much more readable when you have several nested elements.

```css
main article p {
  color: yellow
}
```

scss equivelant:

```scss
main {
  article {
    p {
      color: yellow;
    }
  }
}
```
{{< /note >}}

<!-- Partials and Imports -->
{{< note title="Partials and Imports" >}}

You can create a partial in scss by naming your file with a leading _ (i.e. _header.scss). Then import the file with:

```scss
@import "_header"; /* relative link to file */

header {
  color: red; /* overrides the code inside _header.scss */
}
```

{{< /note >}}

<!-- Mixins -->
{{< note title="Mixins" >}}

These are reusable components which when created can be used within your css elements. You can also include parameters. (These are basically css functions).

```scss
@mixin fancy-border($size: 1px, $color:black) {
  border $size dashed $color;
  border-radius: 5px;
}

header {
  @include fancy-border(10px, blue);
  background-color: yellow;
}
```

{{< /note >}}

<!-- Extends and Inheritance -->
{{< note title="Extends and Inheritance" >}}

Extends can be used to inherit all the parameters in a class to another. The key difference between extends and mixins is that with extends you can use the original class which you extend from in your html. This is like building a class on top of another class in python (i.e. `class MyClass2(MyClass1)`).

```scss
.message {
  font-size: 20px;
  border: 1px solid black;
}

.warning {
  @extend .message;
  color: yellow;
}
```

{{< /note >}}

<!-- Operators -->
{{< note title="Operators" >}}

In css you can use operators like `*2` or `/2`. 

```scss 
$base-size: 20px;
$base-color: red;

p {
  font-size: $base-size;
  color: $base-color;
}
button {
  font-size: $base-size * 2;
  background-color: $base-color - 200; /* 200 shades lighter */
}
```

{{< /note >}}

<!-- Conditionals -->
{{< note title="Conditionals" >}}

Basically just `if-else` statements. The following code makes the color of the text in the header to change based on the size of the page.

```scss
@mixin text-style($size) {
  font-size: $size;
  @if $size > 20px {
    color: blue;
  } @elseif $size == 20px {
    color: red;
  } @else {
    color: green;
  }
}

header {
  @include text-style(30px);
}
```

{{< /note >}}
