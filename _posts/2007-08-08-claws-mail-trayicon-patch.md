---
layout: post
title: "claws-mail trayicon patch"
date: 2007-08-08
categories: claws linux mail patch sylpheed trayicon
---

My mailer is claws-mail (formerly known as sylpheed-claws, a fork of sylpheed) and I love it.
But there's one thing I miss: I can't make the trayicon plugin count only messages in the folders I want instead of all folders. (So it'd tell me about new messages in inbox + important but not in newsgroups and spam.)

Well, here's a quick and dirty proto patch for 2.10.0. The folder names are hardcoded into trayicon.c and necesary changes were made to folder.c . It works for me but of course I should pass a list of folders including paths instead of just a string like "folder1 folder2 ...folderN" and there should be a way to change the folders in the GUI. I'll do it later.

<a id="clawspatchbtn" href="#clawspatchbtn" onclick="javascript:document.getElementById('clawspatchsrc').style.display = 'block';">-- see claws-mail-2.10.0.trayicon.patch --</a>
<pre id="clawspatchsrc" style="display:none">diff -Naur claws-mail-2.10.0/src/folder.c claws-mail-2.10.0.trayicon/src/folder.c
--- claws-mail-2.10.0/src/folder.c 2007-07-02 08:32:16.000000000 +0000
+++ claws-mail-2.10.0.trayicon/src/folder.c 2007-08-08 19:54:21.000000000 +0000
@@ -967,6 +967,17 @@
return retval;
}

+struct PartialMsgCount
+{
+ guint new_msgs;
+ guint unread_msgs;
+ guint unreadmarked_msgs;
+ guint marked_msgs;
+ guint total_msgs;
+ char *folders;
+};
+
+
struct TotalMsgCount
{
guint new_msgs;
@@ -1016,6 +1027,23 @@
}
}

+static void folder_count_partial_msgs_func(FolderItem *item, gpointer data)
+{
+ struct PartialMsgCount *count = (struct PartialMsgCount *)data;
+ char *s = strstr(count->folders, item->name);
+ if (!s)
+  return;
+
+ if (((s == count->folders) || (*(s - 1) == ' '))
+  && ((s + strlen(s) == count->folders + strlen(count->folders)) || (*(s + 1) == ' '))) {
+  count->new_msgs += item->new_msgs;
+  count->unread_msgs += item->unread_msgs;
+  count->unreadmarked_msgs += item->unreadmarked_msgs;
+  count->marked_msgs += item->marked_msgs;
+  count->total_msgs += item->total_msgs;
+ }
+}
+
static void folder_count_total_msgs_func(FolderItem *item, gpointer data)
{
struct TotalMsgCount *count = (struct TotalMsgCount *)data;
@@ -1134,6 +1162,26 @@
return ret;
}

+void folder_count_partial_msgs(guint *new_msgs, guint *unread_msgs, 
+        guint *unreadmarked_msgs, guint *marked_msgs,
+        guint *total_msgs, char *folders)
+{
+ struct PartialMsgCount count;
+
+ count.new_msgs = count.unread_msgs = count.unreadmarked_msgs = count.total_msgs = 0;
+ count.folders = folders;
+
+ debug_print("Counting total number of messages...\n");
+
+ folder_func_to_all_folders(folder_count_partial_msgs_func, &count);
+
+ *new_msgs = count.new_msgs;
+ *unread_msgs = count.unread_msgs;
+ *unreadmarked_msgs = count.unreadmarked_msgs;
+ *marked_msgs = count.marked_msgs;
+ *total_msgs = count.total_msgs;
+}
+
void folder_count_total_msgs(guint *new_msgs, guint *unread_msgs, 
guint *unreadmarked_msgs, guint *marked_msgs,
guint *total_msgs)
diff -Naur claws-mail-2.10.0/src/plugins/trayicon/trayicon.c claws-mail-2.10.0.trayicon/src/plugins/trayicon/trayicon.c
--- claws-mail-2.10.0/src/plugins/trayicon/trayicon.c 2007-07-02 08:32:20.000000000 +0000
+++ claws-mail-2.10.0.trayicon/src/plugins/trayicon/trayicon.c 2007-08-08 19:54:27.000000000 +0000
@@ -197,8 +197,9 @@
guint new, unread, unreadmarked, marked, total;
gchar *buf;
TrayIconType icontype = TRAYICON_NOTHING;
-
- folder_count_total_msgs(&new, &unread, &unreadmarked, &marked, &total);
+ 
+ folder_count_partial_msgs(&new, &unread, &unreadmarked, &marked, &total, "inbox INBOX");
+ //folder_count_total_msgs(&new, &unread, &unreadmarked, &marked, &total);
if (removed_item) {
total -= removed_item->total_msgs;
new -= removed_item->new_msgs;
</pre>

EDIT: As of claws-3.something, vanilla claws has this option in the menu, so the patch is no longer needed.
