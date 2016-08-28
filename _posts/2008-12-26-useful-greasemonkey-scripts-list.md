---
layout: post
title: "useful greasemonkey scripts list"
date: 2008-12-26
categories: browser ff firefox greasemonkey userscripts
---

These are all the greasemonkey scripts I'm currently using. Greasemonkey is a firefox addon that allows you to have custom scripts that change the behaviour of any site. Most of them are rather useful so enjoy! :)

<ul>
<li><a href="http://userscripts.org/scripts/show/24371">Google Reader: Show Feed Favicons</a> - shows favicons of the site the feed comes from instead of the generic rss icon in google reader</li>

<li><a href="http://userscripts.org/scripts/show/3851">Convert hCalendar to Google Calendar</a> - does what it says</li>

<li><a href="#mbank-gm" onclick="document.getElementById('mbank-gm').style.display = 'inline';">mbank login</a> - the mBank login page has autocomplete=off on the login form fields .. and I hate to have to look up my customer id every time .. so just replace the "foobar" in the script with your number and there you go. (It'd work for password too.)
<span id="mbank-gm" style="display: none">

// ==UserScript==
// @name           mbank login
// @namespace      http://localhost
// @include        https://cz.mbank.eu/
// ==/UserScript==

document.getElementById('customer').autocomplete = 'on';
document.getElementById('customer').value = 'foobar';
</span></li>

<li><a href="http://userscripts.org/scripts/show/13720">Show Password on Click</a> - shows the password when you click on a password input field</li>

<li><a href="http://userscripts.org/scripts/show/33042">YouTube Enhancer</a> - so the videos on youtube don't start playing automatically (which sucks when you open many videos at once) and load the higher quality version by default</li>

</ul>
