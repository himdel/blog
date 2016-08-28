---
layout: post
title: "picasaweb and f-spot"
date: 2007-05-09
---

I've tried out Picasa some time ago, I found it enjoyable but rather impractical, I usually know where my pictures are, not when. One feature that's great is that you can upload albums to Picasa Web Albums (http://picasaweb.google.com). I'd use Flickr (http://www.flickr.com) but hey, Yahoo things feel uncomfortable.
I've just come from a trip to Italy so I wanted to upload some photos. Unfortunately, the current linux version of Picasa doesn't support this feature yet. After some googling I found F-Spot (http://www.f-spot.org) does. I've been meaing to give it a try anyway so I installed it.
Now, I should mention I'm using fvwm-crystal as my window manager, no desktop manager, no gnome or kde stuff. Well, F-Spot crashed some time befor I installed all the necessary stuff like gnome keyring a dbus and figured out I have to run it with 'dbus-launch f-spot'. That way, it starts just fine. I used the --view interface since it's simpler and more closer to what I like, unfortunately it doesn't have any way to sort the files (I've found no such way in the non-view interface either) and because it runs the images through a filter (convert to jpg, resize), it uploaded the files with a randomish filename. No sorting on PicasaWeb either, then. (Before I got even there, I had to install the SVN version instead of the stable one since there was a change of protocol.)
Being curious about C#, the language f-spot is written in, I decided to write a patch. Took a relatively short time, you can find the results at http://bugzilla.gnome.org/show_bug.cgi?id=437044 . I think I'm definitely going to play more with it. I should probably add a way to sort in the --view UI.
