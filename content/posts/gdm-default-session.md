---
title: "Set GDM Default session (not the last used)"
date: 2024-08-19T20:18:08+02:00
draft: false
tags: ["doc", "gdm", "regolith", "starlite", "gnome"]
author: "Alexandre"
---

Some context: I [own a Starlite V](https://wazablog.fr/posts/starlite-debian12/), and I use it as a hybrid PC with Gnome and [Regolith Desktop](https://regolith-desktop.com/). When I work at my desk, I use Regolith, then I turn off the computer and go away from keyboard. Then on the next use, I am on my couch and I boot in tablet mode, then GDM3 by default will send me back to the last session used (Regolith here), and... Bam! I'm stuck in a keyboard-oriented WM without a keyboard.

So, I don't trust myself to think about the session switch at each boot, so I started digging on how to always set Gnome as the default choice on GDM3. This is not as easy as it may seem... I found [this article](https://brokkr.net/2016/10/27/setting-default-user-session-in-gdm-default-latest/), but I needed some adaptation to make it work. Here is how to do it in Debian 12 with Gnome and GDM3. (If you don't want to read the details and just get it done: [TL;DR](#tldr))

## How GDM is fetching the last-used session

GDM relies on AccountsService to save the last-used session in `/var/lib/AccountsService/users/$USER`. When a user log into a session with GDM, it will be recorded into this file with `Session=regolith` for example.

GDM will always set this value as the default session. From what I found online, there is no way to change that behavior, so we are going to need a dirty hack.

## Dirty hack

This is directly inspired by the article quoted in the introduction. But the solution provided did not work for me, so I adapted it a bit.

You are going to edit `/etc/gdm3/PreSession/Default` This script is executed after the login of the user. From what I can tell, the update of the AccountService file happens before the PreSession so you can override it at this step. If you want to learn more about the GDM3 scripts, you can [read this page](https://help.gnome.org/admin/gdm/stable/configuration.html.en).

Then you can add this line at the end of the file :

```bash
sed -ie 's|^Session=.*|Session=gnome|' /var/lib/AccountsService/users/CHANGEMEYOURUSER
```

This sed command will replace in the file directly (-i) the matching regular expression (-e) Session=whatever with Session=gnome.

You then need to log out and log in once for it to start working. If you want it to be effective directly at the next log in, you can also run the sed command as root directly.

## TL;DR

1. Edit `/etc/gdm3/PreSession/Default`
2. Add `sed -ie 's|^Session=.*|Session=gnome|' /var/lib/AccountsService/users/CHANGEMEYOURUSER` at the end of the file
3. Log out/login
4. Profit

*Comment this article [on Mastodon](https://h4.io/@wazaby/112990097400063492)*
