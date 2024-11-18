---
sidebar_position: 1
---

# What is BTN

Before we start to understand what BTN is, we first need to understand various anti-leeching methods in history.

## The history of Anti-leeching

### Prehistoric Era

In the beginning, people used blacklists based on PeerID and ClientName to block a specific type of client, preventing them from connecting to the downloader to anti-leech.
This anti-leeching method is crude and effective, but it assumes that the downloader will not disguise PeerID and ClientName. Once disguised, this function is completely ineffective.

### Traffic Detection Era

Later, downloaders like BitComet introduced traffic monitoring methods. If a Peer does not send any segments to the downloader for a period of time while downloading a file, it is considered leeching and is banned.
However, for the leechers starting with 101., they would give very small upload traffic, but enough to deceive this kind of traffic monitoring anti-leeching method. Moreover, this method is useless for seeders.

### Tit-for-tat Algorithm Anti-leeching Era

LibTorrent introduced a tit-for-tat, confrontational anti-leeching algorithm. If a Peer is unwilling to provide file segments, this Peer's priority in LT will gradually decrease until it completely stops transmitting data.
Similar to traffic monitoring, a little upload is enough to deceive the detection, and it is useless for seeders (because seeding does not download any segments).

### Heuristic Detection Algorithm

The heuristic detection algorithm (Progress Checker/PCB) proposed by PeerBanHelper can effectively detect cheating methods such as progress rollback and excessive downloading. But it also has its own shortcomings - slow response.
By the time the detection is triggered, the leecher has already maliciously downloaded a considerable amount of data. Therefore, although this method of anti-leeching is effective, it is inefficient. It can only be used to minimize losses.

## So What to Do

The answer is: shared IP rules.

At first, we passed high-risk IP segments to each other. But with the start of guerrilla warfare, you block my IP, I change it, and unfortunately, this is a very time-sensitive thing. Before more people know about the new rules, the leecher may have already switched to the next IP address.
Therefore, a data analysis and IP blocking technology that can respond quickly and timely must appear.

## The Answer is BTN

BTN (full name BitTorrent Threat Network), it consists of BTN servers and BTN clients. Data is exchanged through the [BTN protocol](https://github.com/PBH-BTN/BTN-Spec).

After the BTN client discovers an abnormal Peer through methods such as PCB, it will report to the server regularly through the BTN protocol. In this way, we can collect data from a large number of clients, generate a list of abnormal IPs, and distribute them to different clients through the BTN protocol, blocking them out before they start leeching.

## What Can BTN Do?

### Collect, Analyze, and Ban Reports

BTN can receive ban reports reported by clients. If an IP (/segment) is banned by multiple people, then it is a high-risk IP address. BTN will automatically ban it and notify other clients to ban it together.

### Prevent Dispersed Leeching

Some leechers are smart, they download the same file from different people.
From the perspective of each person, the leecher only downloaded the file once from them, but in fact, the leecher has downloaded a lot of data far beyond the file size.

BTN analyzes the total download volume of each IP (segment) on each torrent, checks for dispersed leeching, and automatically bans them.

### Discover New Clients

When reporting data, BTN will also check whether it is a new client that has never been seen before. And record the first appearance and the last time seen.
This helps to infer the approximate time range of malicious leeching activities.

In August of the 24th year, it also helped us discover a leeching client using a random PeerID.

### Avoid PBH Being Bypassed

In addition to banning data, the BTN client will also take a real-time snapshot of the status on the downloader from time to time. This way we can detect whether there are malicious leechers bypassing the existing detection through new methods.

## Concerns about Privacy and Security

It sounds like a lot of data needs to be uploaded to the server. The answer is: yes.

To perform these analysis operations, BTN must collect and upload a large amount of necessary data, all types of uploaded data can be found in the [BTN protocol specification](https://github.com/PBH-BTN/BTN-Spec).

It is particularly worth mentioning that: when uploading, we will not upload the info_hash and name of the torrent. Instead, we use the irreversible hashed torrent
