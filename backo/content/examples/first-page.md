+++
title = "Your first page"
description = "Write a guide and preview your changes."
weight = 1
+++

## Edit this page

Open `content/documentation/first-page.md` in a text editor. Change `title` between
the `+++` lines to rename the page. Write the page text below those lines.
Start a line with `## ` to add a heading to **On this page**.

## Add a page

Copy this file to `content/documentation/second-page.md`. Give the copy a new `title`
and set `weight = 2` to put it after this page in the sidebar. The copy will
open at `/documentation/second-page/`.

## Preview your changes

Run this command in the folder that contains `zola.toml`:

```sh
zola serve
```

Open the address printed in the terminal. Leave the command running while you
edit and press Ctrl+C when you're done.

Before publishing, set `base_url` in `zola.toml` to your site's public address
and run `zola build`. Publish the contents of `public/` on your web host.


{% <devlab.timeline> %} 
{% <devlab.timeline_item title="Bring-up" status="done" date="2026-08"> %} UART, U-Boot, kernel, Device Tree, microSD and eMMC. 
{% </devlab.timeline_item> %} 
{% <devlab.timeline_item title="Hardware validation" status="active" date="2026-09"> %} Ethernet, USB, Wi-Fi and Bluetooth. 
{% </devlab.timeline_item> %} 
{% </devlab.timeline> %}