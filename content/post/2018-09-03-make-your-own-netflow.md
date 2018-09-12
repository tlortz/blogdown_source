---
title: Make Your Own Netflow
author: ~
date: '2018-09-03'
slug: make-your-own-netflow
categories:
  - cybersecurity
  - data sources
tags:
  - cybersecurity
  - data sources
type: post
description: description
keywords:
  - key
  - words
topics:
  - topic 1
---

# What is Netflow, and Why Would I Want It?

Netflow is a network traffic summary mechanism originally developed by Cisco for the purpose of monitoring network traffic loads for billing. In its simplest form, Netflow summarizes all of the "flows" - roughly the equivalent of a TCP/IP session - seen at a router and summarizes them by source and destination address and port and the layer 3 or 4 protocol. The summary metrics typically count both packets and bytes contained. Unlike PCAP, Netflow does not contain any content, and is not nearly as voluminous as unsampled PCAP, but it can still be massive when collected even on a moderate-sized network.

Netflow serves several important network and security monitoring purposes. It can tell all the IP addresses present on the network and which ports and protocols they're using. Expanded versions of Netflow also expose the network interfaces and subnets in use. And the byte counts can give estimates of network traffic loads at specific points in the network. 

For a data scientist serving a network or security operations center, Netflow is a fanstastic source of data that lends itself especially well to graph analytics and time series modeling. 

# Where can I get Netflow?

I've supported a few projects where we anticipated getting Netflow data, and wanted to get our data pipeline and analytic tool stack in place ahead of time. But finding decent Netflow samples has turned out to be pretty hard. I suppose this makes sense; if an organization made its Netflow data public, it could pose a significant security risk. It gives the layout of the active addresses, ports and protocols on your network, which is like a poor man's version of open nmap against your network, which any good system administrator or SOC operator would not allow. So from this author's vantage point, there isn't much out good Netflow data on the web. 

However, there are a few ways to get your own. The rest of this post will walk through one of the easiest and safest ways to do it. 

# Setting Up a Netflow Generator

For our purposes, a Netflow data pipeline has three basic components: the __Netflow daemon__ running on the router, an __nfcapd__ process elsewhere listening for and configured to receive the raw Netflow datagrams, and an __nfdump__ batch job that converts the raw Netflow binaries from nfcapd into human-readable .csv files. However, because we don't want to share our real Netflow with the world, we are going to use a sythetic Netflow generator that will replace the Netflow daemon. This generator comes complements of the [nflow-generator project on Github](https://github.com/nerdalert/nflow-generator). 

## Netflow Generator

The project's installation instructions list two possible methods of use: via Docker or by local installation and execution. Here we will go through the latter, which first requires installing the Go language on your machine if not already installed. It can be downloaded and installed from https://golang.org/dl/, after which, you might need to update your PATH variable (e.g. in your .bashrc or .bash_profile file) by adding /usr/local/go/bin to it. Then, from the command line,

```bash
go get -u golang.org/x/crypto/ssh/terminal
go get -u golang.org/x/sys/unix
```

Once you have Go installed, you can proceed to install nflow-generator in a directory of your choosing:
```bash
git clone https://github.com/nerdalert/nflow-generator.git
cd nflow-generator
go build
```

We will come back shortly to actually run the generator once we have the receiving tools in place. 

## nfcapd

Both nfcapd and nfdump must be loaded onto your system from a repository. On a Mac system, this is as easy as 
```bash 
brew install nfdump
```

Once that's done, you need to turn on the nfcapd processor to listen for Netflow on a specific port, and tell it where and how to write the output. I opted to use a subdirectory structure of year/month/day/hour, specified by the -S 2 option, and to bind on host 127.0.0.1 using the -b flag, but otherwise left the default settings intact.

```bash
nfcapd -l ~/Data/netflow/nfcapd -S 2 -b 127.0.0.1
```

Now, with nfcapd listening, we can turn on the Netflow generator, going back to the directory where we've installed it. In a Linux or Mac environment, if you're running both the generator and nfcapd on the same machine, just kick off the two processes in separate terminal windows. 
```bash
./nflow-generator -t 127.0.0.1 -p 9995
```

When you're done collecting the Netflow samples, you can just kill the generator process in the terminal window, and do the same for nfcapd. 

## nfdump

In a production setting, nfdump works well as a crontab job on the same machine as nfcapd set at regular intervals, typically at least five minutes apart. But the Netflow generator we're using is not going to generate so much traffic that you can't read several hours of it into a standard Python or R session, so you can do something as simple as run nfdump on an hour or two of the generated Netflow.

As I mentioned before, there are many options for formatting the output files. You can read all about them at the [man page](https://www.mankier.com/1/nfdump). Here, we'll use the default format in .csv files, with no reporting or summary statistics added (only source/dest IP and ports, plus protocol, and the byte and packet counts, for each flow).

```bash
nfdump -r nfcapd/2018/06/06/17/nfcapd.201806061738 -o csv  > nfdump/201806061738.csv 
#for one file
nfdump -R nfcapd/2018/06/06 -o csv  > nfdump/20180606.csv 
#for all files; -R is for recursive reads
```

# Next Steps

Conducting analysis on Netflow is great fun, but beyond the scope of this post. I will say that I've found Apache Spark to be a natural choice for doing any analysis on Netflow at scale. I also highly recommend Michael Collins' book, [Network Security Through Data Analysis](https://www.amazon.com/Network-Security-Through-Data-Analysis/dp/1449357903), which devotes an entire chapter to the topic. 

I hope this was useful for getting started on Netflow setup and analysis. I'd love any feedback you have!