---
layout: post
title: "GXT Widget Expander 1.0 release"
date: 2010-02-09 00:37
comments: true
categories: java, gwt
---

This project aims to add flexibility to GXT row expander by enabling putting arbitrary widgets in the expanded row.

GXT's grid widget is great, especially with the RowExpander plugin, it allows the creation of a Google Reader style grid (a list with expandable rows). However, the current GXT's RowExpander design only allows rendering by XTemplate. XTemplate falls short in a number of ways, especially when it comes to user interaction inside the expanded rows.

Using the WidgetExpander, you'll write something similar to the following
```java
    WidgetExpander<SomeModel> expander = new WidgetExpander<SomeModel>(new WidgetRowRenderer<SomeModel>() {

        public Widget render(final SomeModel model, final int rowIdx) {
            // Create and return a widget according to the model
        }
    });

    // Add expander into a list of ColumnConfigs
    // Create a grid with the ColumnConfig list
    grid.addGridPlugin(expander);
```

With this enhancement, we're now able to do this:
<a href="http://reminiscential.files.wordpress.com/2010/02/2010-02-24_1654.png"><img src="http://reminiscential.files.wordpress.com/2010/02/2010-02-24_1654.png?w=300" alt="" title="effect" width="300" height="101" class="aligncenter size-medium wp-image-141" /></a>

Check it out here on Github: <a href="http://github.com/kevinjqiu/gxt-widget-expander">http://github.com/kevinjqiu/gxt-widget-expander</a>

