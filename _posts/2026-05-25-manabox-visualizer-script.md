---
layout: post
categories: mtg python
title: "A script to visualize my MTG collection from a bird's-eye view"
description: "A script to visualize my MTG collection from a bird's-eye view"
keywords: "mtg, python, manabox"
---
## Table Of Contents

* TOC
{:toc}

## Motivation

I track my MTG collection in the [ManaBox](https://manabox.app/) app. My collection is moderately large (around 13-15k cards), I sell cards on CardTrader and CardMarket, and I wanna always know which card is where. The ManaBox team made a handy app with a great scanner, and I'm fully happy with it.
I'm trying to keep my cards in order, so I can quickly find them by box or binder when I need to. I have three types of boxes: small ones (~500 cards), medium (~1000 cards), and large (~4000), and at my collection size I'm already running into logistics problems: I wanna keep cards grouped (I usually want to group commons and uncommons of the same set together), but at the same time I don't wanna use too many boxes.

![part of collection, small boxes](/static/img/posts/mtg-collection.jpg)

And here comes the problem of arranging cards across boxes in an optimal way: I need to keep boxes filled to 90-95% (so the cards don't move inside too loosely), and at the same time keep the grouping, so I can find the cards quickly. Bonus requirement: I also wanna group the boxes themselves, so the most in-demand sets are close to each other, which saves time when I pick cards for an order.

I wasn't able to find that kind of information in the ManaBox interface, so I decided to make a script that visualizes the data in a way that works for me.

## Requirements

The simpler, the better. So I vibecoded a single Python script that produces a single HTML file, which can be just opened in browser.

![report page. still need to reorganize storage for SOS and EOE sets](/static/img/posts/mtg-collection-report.png)

With minimal changes it should work with any CSV file, not only with ManaBox exports - you probably just need to change the field names and adapt some small things.

## How I built it

Claude Code. For small projects I have the standard flow:

* I write a free-form spec and hand it to Claude
* A formal spec is created from it
* Then I read this spec carefully and fix the parts that the LLM misunderstood or made up
* The corrected spec goes back to Claude, and then the code is written based on it
* Then I review the result, and if something is wrong, I update the spec and repeat the cycle again

## Result

[ManaBox Collection birdview](https://github.com/strangeman/manabox-birdview/)

In the end I managed to reorganize my storage so that most of the bulk cards follow the "one box - one set" rule. The cards most people buy from me are from remastered sets, so they were collected into one large box (and TDM ended up there too, since I have the most of those).
