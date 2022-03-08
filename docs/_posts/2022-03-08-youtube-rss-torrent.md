---

# @formatter:off
layout: post 
title:  "From YouTube channel to RSS video podcast, backed with BitTorrent"
date:   2022-03-08 20:00:00 +1100 
categories: blog
# @formatter:on
---

Wouldn't it be great to publish video content, but not have to pay for all the infrastructure necessary to deliver it to
your subscribers? By combining youtube-dl, RSS feeds, and BitTorrent, we can deliver our video content with the power of
a peer-to-peer network.

This post assumes that you are familiar with both [BitTorrent](https://en.wikipedia.org/wiki/BitTorrent)
and [RSS](https://en.wikipedia.org/wiki/RSS).

## Method

There aren't a lot of steps involved to convert a YouTube channel into an RSS feed:

1. Download all videos from the channel
2. Create a torrent of each video
3. Generate an RSS feed
4. Host the RSS feed
5. Import the torrents and videos into a BitTorrent client

Ideally, we do this in a way that it's easy to reuse prior work, e.g. we don't download the same video twice. We'll also
automate this as much as possible, so we don't have to do a lot of manual work.

### Download all videos from the channel

Luckily for us there is already a great tool out there that will do this job perfectly for
us. [youtube-dl](https://youtube-dl.org/) will not only download the videos, but also the metadata like the title,
description, and publication date. We'll want to include these in our RSS feed.

To get the videos, we just give youtube-dl our channel URL, let's use the BBC News
channel `https://www.youtube.com/c/BBCNews` as an example here.

Since we need some metadata for the RSS feed, like the title, description, and publication date, we
add `--write-info-json` to collect that in a separate JSON file.

We also don't want to download the same file twice if we are interrupted or want to update our feed later, so we'll also
add `--download-archive downloaded.txt`. This tells youtube-dl to record each video it downloads in the downloaded.txt
file, and won't download one again if it is listed there.

Our final youtube-dl command:

```bash
youtube-dl \
  --write-info-json \
  --download-archive downloaded.txt \
  https://www.youtube.com/c/BBCNews
```

Here we can see the files that youtube-dl has created (for the sake of this example, I limited the number of videos it
downloaded to 3):

```text
$ ls
downloaded.txt                                                           
Fighting misinformation in Russia's invasion of Ukraine - BBC News-HqipD0-OXqw.info.json
Fighting misinformation in Russia's invasion of Ukraine - BBC News-HqipD0-OXqw.mp4
Heavy fighting near Ukraine's capital Kyiv  - BBC News-CbfqprwQpls.info.json
Heavy fighting near Ukraine's capital Kyiv  - BBC News-CbfqprwQpls.mp4
War in Ukraine - Seven days that changed the world - BBC News-BzR_K2tEXSo.info.json
War in Ukraine - Seven days that changed the world - BBC News-BzR_K2tEXSo.mp4
```

We can also view the JSON metadata:

```shell
$ jq '{title,description,upload_date,webpage_url}' "Fighting misinformation in Russia's invasion of Ukraine - BBC News-HqipD0-OXqw.info.json"
```

```json
{
  "title": "Heavy fighting near Ukraine's capital Kyiv  - BBC News",
  "description": "People in the town of Irpin have been seen running to escape Russian bombardment.\n\nAt least three people are reportedly killed fleeing the town of Irpin, after Russian mortar shells targeted a damaged bridge they were using\n\nThe Ukrainian military has been helping residents get to safety.\n\nSituated 20km (12 miles) north-west of Kyiv, Irpin is near the strategic Hostomel airfield, and the front of the huge Russian convoy assembled near the capital.\n\nPlease subscribe HERE http://bit.ly/1rbfUog\n\n#BBCNews",
  "upload_date": "20220306",
  "webpage_url": "https://www.youtube.com/watch?v=CbfqprwQpls"
}
```

### Create a torrent of each video

Now that we have our video and metadata files, we need to generate torrents of them<!-- Get it? -->. Torrent files allow
users to download the videos through the BitTorrent network. Since we may have a lot of videos in our channel we should
try and automate the process.

I opted to use the Python [reference implementation](http://bittorrent.org/beps/bep_0052_torrent_creator.py) available
in [BEP 52](http://bittorrent.org/beps/bep_0052.html), since it will generate metadata in a hybrid format that will be
compatible with both v1 and v2 formats. For Bencoding, I used [bencoder](https://github.com/utdemir/bencoder), a simple
decoder-encoder for the torrent file format.

With these dependencies, and using the `.info.json` files we got from youtube-dl, we can generate a `.torrent` file for
each video.

```shell
$ ls *.torrent
```

```text
Fighting misinformation in Russia's invasion of Ukraine - BBC News-HqipD0-OXqw.mp4.torrent  
Heavy fighting near Ukraine's capital Kyiv  - BBC News-CbfqprwQpls.mp4.torrent
War in Ukraine - Seven days that changed the world - BBC News-BzR_K2tEXSo.mp4.torrent
```

Using [torrentool](https://github.com/idlesign/torrentool), a CLI tool for working with torrent files, we can see a bit
more information about these torrent files.

```shell
$ torrentool torrent info "Fighting misinformation in Russia's invasion of Ukraine - BBC News-HqipD0-OXqw.mp4.torrent"
```

```text
Name: Fighting misinformation in Russia's invasion of Ukraine - BBC News-HqipD0-OXqw.mp4
Files:
Fighting misinformation in Russia's invasion of Ukraine - BBC News-HqipD0-OXqw.mp4
Hash: 368469a45f9d936fd0f7d45ff231d8f0d62443b6
Size: 19.76 MB (20716627)
Magnet: magnet:?xt=urn:btih:368469a45f9d936fd0f7d45ff231d8f0d62443b6
```

### Generate an RSS feed

Now that we have our torrent files, we need to generate an RSS feed. This is simple enough to do in Python
with [rfeed](https://github.com/svpino/rfeed). Using the JSON metadata, we can translate this into our required channel
RSS XML.

We also want to generate a [magnet link](https://en.wikipedia.org/wiki/Magnet_URI_scheme) for each file, which will
allow RSS subscribers to start a BitTorrent download without us having to provide the torrent files to them via
traditional hosting. They will instead fetch them via the BitTorrent client.

Our final RSS feed, `feed.xml`:

```xml
<?xml version="1.0" ?>
<rss version="2.0">
    <channel>
        <title>BBC News</title>
        <link>https://www.youtube.com/channel/UC16niRr50-MSBwiO3YDb3RA</link>
        <description/>
    </channel>
    <items>
        <item>
            <title>Fighting misinformation in Russia's invasion of Ukraine - BBC News</title>
            <link>https://www.youtube.com/watch?v=HqipD0-OXqw</link>
            <description>
                Russia's invasion of Ukraine has been accompanied by false or misleading viral videos and images about
                the war [...]
            </description>
            <pubDate>Sun, 06 Mar 2022 00:00:00 GMT</pubDate>
            <enclosure type="application/x-bittorrent"
                       url="magnet:?dn=Fighting+misinformation+in+Russia%27s+invasion+of+Ukraine+-+BBC+News&amp;xt=urn%3Abtih%3Acffb3d1ecc663aaaf9061528e89edf3dc49ba1d1&amp;xt=urn%3Abtmh%3A1220997045ef4607522f7368ac55ce5937e7275ce9744bff4be61eaef655b2e82e79"
                       length="0"/>
            <guid isPermaLink="true">https://www.youtube.com/watch?v=HqipD0-OXqw</guid>
        </item>
        ...
    </items>
</rss>
```

### Host the RSS feed

An RSS file can be cheaply hosted on any HTTP server. For testing purposes, we can host it locally with Python.

```shell
$ python -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

### Import the torrents and videos into a BitTorrent client

Now if we go to our BitTorrent client of choice ([qBitTorrent](https://www.qbittorrent.org/)) we should be able to
access the feed at `http://localhost:8000/feed.xml`.

![qBitTorrent client listing three BBC feed items, with 'double-click' to download as a header](/assets/blog/youtube-rss-torrent/qbittorrent-rss.png)

If we select one of the items, we can start downloading it.

![qBitTorrent client listing three BBC videos as 'Downloading metadata'](/assets/blog/youtube-rss-torrent/qbittorrent-pending.png)

The screenshot above shows that the download is stuck in the 'Downloading metadata' stage. Since we're the creator of
the torrent, we need to provide the client with a bit more information.

We need to manually import the torrent metadata files, and the corresponding video files into our BitTorrent client so
that we can start seeding both of these to other users on the network. It's as simple as copying the video files into
the download directory and manually opening the metadata files with the client, the client will automatically detect the
video files, and start seeding.

![qBitTorrent client listing three BBC videos as 'Seeding'](/assets/blog/youtube-rss-torrent/qbittorrent-seeding.png)

We can double-click one of the completed video files to watch it.

![VLC Media Player showing the first frame of the BBC News example video](/assets/blog/youtube-rss-torrent/vlc-bbc.png)

We're done! Anyone who subscribes to this RSS feed now can download any of the content from our BitTorrent client. As
more people download, we will be required less and less to upload data, as pieces will be shared amongst all
participants on the network.

## Conclusions

So, hopefully I've proved that you _could_ convert your YouTube channel into a video podcast. Now let's talk about why
you probably shouldn't. Or at least, why you shouldn't if your target audience member is an existing podcast consumer.

### Podcast applications don't support torrents

The biggest problem I see in doing this is that whilst you can easily subscribe to a torrent RSS feed with any
BitTorrent client, the same can't be said for podcast applications. The typical user won't be able to subscribe to your
RSS feed, because torrents are just not supported. At best, the podcast client will complain that there's no media file
linked to the feed item, and at worst it will outright reject the feed.

BitTorrent clients aren't designed for podcast consumption and management, and I don't think people are going to switch
away from a podcast-focused application because of the inconvenience it would cause in consuming media. Even BitTorrent
users would agree, as many use other programs manage their media, using the BitTorrent client only as a
download manager via API calls.

### Podcast platform technical requirements are a blocker

The lack of torrent support with RSS will also affect your ability to get your podcast listed in a directory, like Apple
podcasts for example. Most podcast platforms that I am aware of have technical requirements on how media is delivered
to the end user, requiring content be streamable over HTTP.

Using peer-to-peer to host media content with RSS feeds isn't a new concept though. Added to
the [Media RSS specification](https://www.rssboard.org/media-rss#media-peerlink) in 2009, a `peerLink` element which
allows specifying a peer-to-peer link to a media object. Unfortunately there's no need for podcast applications or
platforms to include peer-to-peer support, since the client-server approach works well enough and is simpler to
implement.

### An alternative to YouTube, or other video platforms

While not an outright replacement for these large video platforms, it is possible to use RSS feeds and torrent-served
media to distribute your content. The problem is the other features that these hosting platforms provide which RSS does
not give you, namely search and discovery. You are going to need to publish your feed somewhere where it can be indexed,
searched, and discovered by others.

You are also going to lose out on analytics and activities like views, likes, dislikes. You could use BitTorrent
downloads, peer and seed numbers as a proxy for some of these, but are probably going to be pretty inaccurate.

I don't think it's possible to be truly independent of these big platforms, but if you want your content to still be
accessible by some means if removed from a platform, this could work.

### Improvements

Some other things I thought of adding but weren't necessary to prove the concept.

While this feed will never meet
the [Apple Podcast RSS feed requirements](https://podcasters.apple.com/support/823-podcast-requirements), which requires
that content be streamable via an HTTP server, it also doesn't meet
the [metadata requirements](https://help.apple.com/itc/podcasts_connect/#/itcb54353390). This requires additional
information about the content that we can't easily determine with just youtube-dl. This will either need to be supplied
manually by the user, or maybe scraped from YouTube via an API.

There are many BitTorrent Enhancement Proposals (BEPs), with various levels of adoption, which provide some neat
additional features that we could enjoy:

- With [BEP 39](http://www.bittorrent.org/beps/bep_0039.html), we could include a link to our RSS feed inside the
  torrent file so subscribers could automatically see new episodes or revisions to our content. If someone found one of
  our torrents without subscribing to our RSS feed, they could still subscribe to new items, or view older items in the
  feed.

- With [BEP 46](http://www.bittorrent.org/beps/bep_0046.html), we don't even need an RSS feed and could instead publish
  items via the [DHT](https://en.wikipedia.org/wiki/Mainline_DHT).

- With [BEP 53](http://www.bittorrent.org/beps/bep_0053.html), we could create one torrent file with multiple video
  qualities, and include metadata in the RSS feed of the quality of each file, deep linking to each one via magnet link.
  Consumers could then automatically download in the quality that they choose. Files are still served via a single
  torrent swarm, and can share the metadata between them.

## Code

If you are interested and would like to play around with this, you
can [view the code here on GitHub](https://github.com/chhs1/youtube-rss-torrent). You are free to do whatever you wish
with it.

Thanks for reading.
