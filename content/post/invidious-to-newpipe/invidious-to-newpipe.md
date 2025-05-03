---
title: Exporting from Invidious and Importing to NewPipe
description: Import subscriptions from Invidious to NewPipe
slug: invidious-to-newpipe
date: 2025-05-03 00:00:00+0000
image: /images/could-not-load-video.png
categories:
    - How-To
tags:
    - YouTube
    - Invidious
    - Self-Hosting
    - NewPipe
    - AntennaPod
    - RSS
    - Privacy
draft: false
---

[Click here to skip to the instructions.](#exporting-from-invidious)

## Invidious vs. YouTube

No, I'm not going to compare invidious and YouTube, this section title is an apt description of what's going on at the moment. In the fight fight for a truly private method of watching independent content creators, it's really invidious vs. YouTube. Or really any project that tries to improve on YouTube vs. Google. This week I was reminded of that as YouTube switched from DASH to SABR and broke both my Invidious Sig-Helper and Invidious-Companion servers leaving me once more unable to watch my collection. Well, besides my 2TB of videos downloaded on my media server via [Pinchflat](https://github.com/kieraneglin/pinchflat) and accessible via [Jellyfin](https://jellyfin.org/), but everyone knows that's simply not enough.

![A screenshot showing Pinchflat's main stats page displaying a 2.18TB library](/images/pinchflat_stats.png)

Now, I don't know about you, but I work long hours staring at computer screens willing aforementioned computers to behave. This requires a great deal of fortitude and focus as often the computers choose not to behave, making my job difficult but ensuring a steady income. This breakage with invidious has caused me to administrate the various systems under my purview without the precious background noise that is so important to my sanity and workflow. I have been personally victimized by Google this week, and I will be seeking redress in the form of a [donation to the Invidious project](https://invidious.io/donate/) as should you.

While a [fix was deployed in time for me to watch the WAN show](https://github.com/iv-org/invidious/issues/4251#issuecomment-2847807862), I've decided I'll spend my time watching the WAN show on the third monitor while I set up an additional YouTube alternative I can literally keep in my back pocket. NewPipe, an android client for YouTube. All I need to do is export my invidious subscriptions, and import them to NewPipe, easy peasy!

## Not so peasy

Unfortunately, no. As anyone who has logged into Invidious, gone to settings, selected import/export data, and exported any of the options and tried to import them into NewPipe you know this data flow is completely broken. This method will bring you nothing but pain. And possibly, like a "friend of mine", you managed to destroy your [AntennaPod](https://antennapod.org/) subscriptions by trying to import the OPML and your podcast app was recognized as the destination. So now I- I mean "my friend" - is out two of their background noise producers. Man I'm glad I'm not as dumb as they are. I'm even more glad that if that ever happened to me I'm not syncing AntennaPod with [gpodder sync](https://apps.nextcloud.com/apps/gpoddersync) so my other devices had their subscriptions wrecked as well, that would've just sucked, I don't know how I'd look myself in the mirror every morning if that happened to me.

![A screenshot showing the three export options available in the invidious settings](/images/invidious-export-options.png)

So... yeah. [The invidious export is incompatible with the NewPipe client](https://github.com/iv-org/invidious/issues/2991), but don't worry! [A conversion tool was created!](https://github.com/wheresalice/invidious2newpipe)

![A screenshot showing the github 404 page on the invidious2NewPipe project](/images/invidious2newpipe.png)

## Fine... I'll do it myself.

Well, fine. At this point I've given up looking for anything that will do it for me and I'm just gonna figure this out on my own. I'm only ever going to do this once (famous last words), so rather than spend the several hours automating the task, I'll spend the 15 minutes to do it once in a really poor manner and then spend hours writing a blog post about it so people can make fun of me for not bothering to figure out how to do this on the CLI. Let's go!

### Exporting from Invidious
First off, go to your invidious server's webpage (you can't do this through any of the clients as far as I'm aware). Log in and navigate to settings > import/export data and choose `Export Invidious data as JSON` and save the .json file somewhere you can access it easily.

### Figuring out the formatting differences

Now, if you did a test export with some random subscriptions in NewPipe and looked at the .json output, you would've seen something like the following:

```json
{"app_version":"0.27.6","app_version_int":1003,"subscriptions":[{"service_id":"0","url":"https://www.YouTube.com/channel/UC-2YHgc363EdcusLIBbgxzg","name":"Joe Scott"}]
```

So ignoring the headers, we've got `service_id` which as far as I can tell is always `0`. We've got the `url` and then we've got a `name`.  If you were to open up the Invidious file you would see something like the following instead (if you clean out all the favorites, watch later, and other fluff data in the export and are just looking at the subscriptions):

```json
{"subscriptions":["UC-2YHgc363EdcusLIBbgxzg","UCy0tKL1T7wFoYcxCe0xjN6Q","UC-nDNn0pKEyOGH5d0_ttqSA","UC0vBXGSyV14uvJ4hECDOl0Q","UC-SrCCzkGq0wmSAuRs7EBFg","UC-FHoOa_jNSZy3IFctMEq2w","UC-1TUi3OPc2jSVWLNM2ARwg","UC-nPM1_kSZf91ZGkcgy_95Q","UC06E4Y_-ybJgBUMtXx8uNNw"]}
```

You'll likely notice this is the channel-ID portion of the YouTube Channel's URL. If you were to go to `https://youtube.com/channel/UC-2YHgc363EdcusLIBbgxzg` you would be taken to Joe Scott's YouTube channel. But no name! Darn! That's alright, we can massage this data and get it working how we need for NewPipe. All we need to do to start is preface each of these entries with `https://youtube.com/channel/`

### Completing the channel url
You're going to need a text editor capable of working with escape sequences. On Windows (which I must be paid to use, and I unfortunately am for 40 hours each week) I recommend [Notepad++](https://notepad-plus-plus.org/). On Linux (which I would pay to use) I use [Kate](https://kate-editor.org/) as I'm a sucker for KDE Plasma.

These instructions will use Kate, but whatever you choose, open up `subscription_manager.json` and do a find and replace (`CTRL` + `R` on Kate). Set the mode to "Escape Sequences" and find `,` and replace it with `\n`. This will make your document probably several hundred lines long if you're like me.

![Deleting everything below watch history](/images/delete_below_watch.png)

Next, search for `watch_history` and delete every line below it. Then find `"\n` and replace it with `\n`. Lastly, find `"` and replace it with `https://youtube.com/channel/`

![setting_the_url](/images/setting_the_url.png)

### Creating the CSV

Now we've got the rest of the url created, we need to add the other fields. Trust me on this, as of writing I have tried just converting this to a JSON and NewPipe didn't do anything with it. The `service_id` and `name` fields are important for some reason. Let's open libreoffice (or excel, or google sheets, or whatever, the steps should be the same), and make a nice .csv. First enter your headers on row one and then paste the urls you just created from your subscription_manager.json into the csv under the `url` header.

![a screenshot of libreoffice showing the above sequence](/images/url_pasted.png)

Paste `0` as the value for each row in the `service_id` column, and type 1, 2, 3, 4.... for the value for the `name` tag. After you enter a few rows in the spreadsheet, you can double click the bottom of the cell to extend the pattern you've entered so far to all remaining rows to save you some time. The name will not impact anything as far as I can tell when you import into NewPipe, but boy does it not work if you don't have something in that field.

![extending the values down the row](/images/extend_libreoffice.png)

Format all of the cells as `text` and once done, go ahead and save the file as `subscription_manager.csv`. When libreoffice pops up asking for what options you'd like to use, select the options shown in the below screenshot. 

Field delimiter should be `,`
String delimiter should be empty
Save cell content as shown should be checked
Quote all text cells should be checked

### Converting the CSV to JSON

This is the laziest part. I used [csvjson.com](https://csvjson.com/csv2json). Open up your newly created CSV in your text editor, copy the content to your clipboard, paste the CSV onto that website, click "convert." The site seems fine, and we're not downloading anything from it anyway or transferring any sensitive information (unless your threat model includes YouTube subscriptions, in which case I would advise you to re-think your threat model and stop watching YouTube).

![csv2json converter site example](/images/csv2json)

Next create a new file in your text editor, paste the following into it, and then copy the output from csvjson.com where it says `PASTE_HERE` below.

```json
{"app_version":"0.27.6","app_version_int":1003,"subscriptions":PASTE_HERE}
```

Save this file as `man_why_didnt_this_dude_write_a_script_for_this.json` or whatever you want so long as it ends in `.json`, and transfer it to your phone. Lastly, open the NewPipe app on your phone, go to the subscriptions tab, click the 3 dots, select "import from", choose "previous export" and then select `man_why_didnt_this_dude_write_a_script_for_this.json`. Wait a few moments for the full import to complete, you should see your subscriptions populate in a moment. You're now done, enjoy watching creators on NewPipe!