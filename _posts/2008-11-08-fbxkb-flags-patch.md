---
layout: post
title: "fbxkb flags patch"
date: 2008-11-08
categories: fbxkb keyboard linux patch X
---

My primary keyboard layout is US (qwerty), but from time to time, one needs a different one (usually cz_qwerty or hu for me). So I have this handy line in my ~/.xinitrc that let's me switch to the alternate layout by using Right Alt:
setxkbmap -option grp:switch,grp:alts_toggle,grp_led:scroll us,cz_qwerty

And I use fbxkb to see the current layout in the tray. The only problem is, it doesn't work .. instead of the us flag, it shows question marks and when i switch to the us,hu layout combo, it shows question marks for both layouts.

Well, no more. It turns out, it was quite an easy bug to fix so here's a patch against fbxkb-0.6.

---<a href="#fbxkb-patch" onclick="document.getElementById('fbxkb-patch').style.display = 'block'">Show the patch</a>---
<pre id="fbxkb-patch" style="display: none">--- a/fbxkb.c 2006-12-18 21:47:52.000000000 +0000
+++ b/fbxkb.c 2008-11-08 16:25:32.000000000 +0000
@@ -378,10 +378,11 @@
g_assert((no >= 0) && (no < ngroups));
             if (group2info[no].sym != NULL) {
                 ERR("xkb group #%d is already defined\n", no);
+            } else {
+                group2info[no].sym = g_strdup(tok);
+                group2info[no].flag = sym2flag(tok);
+                group2info[no].name = XGetAtomName(dpy, kbd_desc_ptr->names->groups[no]);           
}
-            group2info[no].sym = g_strdup(tok);
-            group2info[no].flag = sym2flag(tok);
-            group2info[no].name = XGetAtomName(dpy, kbd_desc_ptr->names->groups[no]);           
}
XFree(sym_name);
}
</pre>

---

Comments:

how do you apply it?
Bryce
on 11/22/08

you unpack the source, enter into the fbxkb-version/ directory and use patch -Np1 .. passing it the patch on stdin (or add -i file.patch)
himdel
on 11/22/08

so, if you already have a fbxkb-0.6/ in you current directory, just docd fbxkb-0.6/wget -qO- http://himdel.mine.nu/fbxkb.patch | patch -Np1and then you can happily compile your shiny new fbxkb :)
himdel
on 11/22/08

thanks for this! :)
robert_siska
on 6/25/11
