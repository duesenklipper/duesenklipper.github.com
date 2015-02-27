---
layout: post
title:  "Bringing order to my papers with Linux-based document scanning"
categories: linux scanning documents ocr howto
---
I have considerable difficulty with keeping papers and documents in order and well-filed. Yes, I'm talking about
dead-tree documents. They still exist, and are still important for many things. Getting them filed in a timely manner
and in a way that would allow me to find a needed document quickly has been a challenge. I had a separate binder each
for tax-related things, bank/insurance things and one folder for other things like invoices and warranty documents. All
except the last one were simply sorted chronologically, with the last binder being sorted by year, then by company.

## Dealing with paper in an electronic world

This setup sorta kinda worked, but I was never really happy with it. It was a hassle, it required many decisions
upfront, and sometimes a good bit of head-scratching. Did I put this document into "tax" or "bank/insurance"? When did I
buy this device, so where is the invoice for it? I ended up often spending more time than I liked looking for things,
and often not filing things quickly, which made the problem worse.

Some online services like Evernote let you scan/photograph documents and store them for you online, where they do OCR to
make the documents searchable. While that is convenient, it's also rather questionable when it comes to privacy. I
rather value my privacy, so I didn't go with those services.

## What do I need to keep private data private?

So, I needed something that would keep my data on my own machines. I have an HP Officejet 6500, which even has an ADF.
Sounds good, right? Let's look at what I want:

1. I want to just dump documents into the scanner and not have to futz around with them afterwards.
2. I don't want to do anything special for double-sided documents.
3. I want to be able to do full-text search on all documents to find what I'm looking for.
4. All files should be in my local network - that means either on my desktop, or, preferably, on the little Linux-based
   file server that I had running anyway.
5. If possible, I want to do an indexed search, so it won't take exceedingly long once I have more than a few files.
6. I want to do all that with free software (specifically on my Linux systems).

Unfortunately, the Officejet fails on the first two already. It can't just dump the scanned documents onto the file
server, it needs an interactive computer. If you have a Windows desktop, you can halfway get around this, because you
can just send the file to HP's Windows desktop software. This still means you have to have the desktop running, and it
fails requirement number 6.

It also fails item 2, because even though it has an ADF, it can only do one-sided scanning. This means that the ADF is
basically useless once you have a double-sided document, because you have to do everything manually to get the pages in
the right order.

However, there are dedicated document scanners on the market these days.

## Can I reasonably do this with Free Software?

At this point, I took a (very short) look at the available document scanners to see if there would be anything
affordable - if there weren't, I'd shelve this project for now. Luckily, a quick search found a few possible devices in
the <500€ range.

But before spending more time (and eventually money) on hardware, I wanted to look at the software side of things, to
see whether my imagined setup could work at all.

Some manufacturers like HP and Brother do provide Linux drivers for their scanners, but you don't get any useful desktop
software with them. Most of the really advanced features are done on the computer rather than the scanner, like "Scan to
Email" or OCR. So I did some searching for possible alternatives in the Linux world.

### OCR

Pretty much everything I found on the topic of OCR said that tesseract is the best open source package in this area.
It's really just the engine and a command line client though, nothing really fancy with a GUI and shiny graphics.

