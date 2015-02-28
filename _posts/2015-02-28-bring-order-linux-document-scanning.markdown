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

So, I needed something that would keep my data on my own machines. I have an HP Officejet 6500, which even has an
<abbr title="automatic document feeder">ADF</abbr>. Sounds good, right? Let's look at what I want:

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

Pretty much everything I found on the topic of OCR said that [`tesseract`](https://code.google.com/p/tesseract-ocr/) is
the best open source package in this area. It's really just the engine and a command line client though, nothing
really fancy with a GUI and shiny graphics. But that suits me just fine: Command line means it's scriptable, and I
don't want to interact with it anyway.

So my first idea was to simply let a document scanner dump a scanned image on the server and then script `tesseract`
to extract the text and put it in a file next to the image. That way, I could `grep` for it and find my document.

But it turns out someone else had already done some scripting and graciously put it on github:
[`pdfocr`](https://github.com/gkovacs/pdfocr) by [Geza Kovacs](https://github.com/gkovacs). This Ruby script does
basically what I described, but even more. It not only performs the OCR, but it actually adds a text layer to the PDF
containing the image, so the PDF itself becomes searchable. That text layer is invisible, so when you open the PDF
in a viewer, you still see the scanned image, but you can search for text and the results will show up exactly where
the letters are in the image. Brilliant!

So now I manually scanned a document with my existing HP OfficeJet and fed that to `pdfocr` to see whether it really
worked. It did - so now I knew the OCR part of my problem was covered.

### Automation

Of course, I did not want to invoke this manually each time I got a new document. Assuming a document scanner can
dump a scanned file (a PDF) on my Linux machine, can I get it to be picked up automatically? I knew I could just set
up a cron job to check the target directory every few minutes, but that seemed a bit silly. I know `inotify` can be
used to watch file system events, so I googled a bit for anything that used that. I found
[`incron`](http://packages.ubuntu.com/trusty/incron), which works basically like good old cron, except you define
`inotify` events instead of times, for example like this:

    /home/calle/Documents/scan IN_CLOSE_WRITE /home/calle/bin/pdfocr-incoming.fish $@/$# /home/calle/Documents/scanned/$#

I recommend `incron`'s pretty good man page for details on the configuration. This line tells `incron` to watch the
directory `/home/calle/Documents/scan` for files that have been written to and are now closed (`IN_CLOSE_WRITE`) and
then invoke `pdfocr-incoming.fish`, a small wrapper script for `pdfocr`:

    #!/usr/bin/fish
    set -l infile $argv[1]
    set -l outfile $argv[2]
    if pdfocr.rb -L deu+eng -i $infile -o $outfile
    	rm $infile
    end

Yes, I've started using [`fish`](http://fishshell.com/) for at-home use. It's beautiful. That said, all this little
script does is invoke `pdfocr` and, if it was successful, remove the original file.

The `-L deu+eng` argument tells `pdfocr` to pass `deu+eng` as the language parameter to `tesseract`. The `+` syntax
tells `tesseract` to use both German and English dictionaries for its text recognition. Unfortunately, the original
`pdfocr` script chokes on that `+`, so I had to extend it. In addition to the original `-l` argument my version
also understands `-L` which simply passes the argument straight through to `tesseract`. You can find that currently
at my [fork of `pdfocr`](https://github.com/duesenklipper/pdfocr).

So now this works as well - I can have freshly scanned documents be automatically picked up, OCR'ed and dropped into
an archive directory. Excellent.

### Searching

An archive is only as useful as its searchability. Having all the documents stored in digitized form I need a way to
actually find any document that I need. My first approach was a simple `grep`-like script that I called `multipdfgrep`:

    #!/usr/bin/fish
    find . -name '*.pdf' -exec sh -c 'pdftotext "{}" - | grep --with-filename -i --label="{}" --color '"$argv" \;

This is very basic but gets the job done in a pinch. It takes all PDF files, extracts the text with `pdftotext` (from
the `poppler-utils` package) and then `grep`s on that. Since it searches all files all the time it is not very fast once
you have an interesting number of files. I kept it around for emergencies, but it obviously would not scale.

An indexed search would be much more useful. I ended up setting up [`recoll`](http://www.lesbonscomptes.com/recoll/),
an open source search engine that comes with a reasonably useful GUI but can also be used on the command line.
Instead of having its indexing daemon running all the time, I simply set up a second watch in my `incrontab` that
would add the OCR'ed files to the index:

    /home/calle/Documents/scanned/ IN_DELETE,IN_CLOSE_WRITE,IN_MOVE /usr/bin/flock /home/calle/.update-document-index.lock /home/calle/bin/update-document-index $% $@/$#

This monitors my archive directory for files that are moved into it (`IN_MOVE`), written to (`IN_CLOSE_WRITE`) or
removed (`IN_DELETE`) and calls another small script. It uses `flock` and a lock file to ensure that these operations
are serialized and there are no race conditions, e.g. when a file is moved into the directory and quickly removed again.

The following script processes the various events. It needs to be a script because `incron` can only have one
configuration line per directory, so that one line must cover all events.

    #!/usr/bin/fish
    set LOGFILE /home/calle/update-document-index.log
    echo (date --iso-8601=seconds) $argv >>$LOGFILE

    function log
      echo -n "  " >>$LOGFILE
      echo $argv >>$LOGFILE
    end

    if test (count $argv) -ne 2
      echo "Usage: update-document-index EVENTNAME FILE"
      log exit for bad arguments
      exit 1
    end

    set EVENTNAME $argv[1]
    log event $EVENTNAME

    set FILENAME (readlink -m $argv[2])
    log canonical filename $FILENAME

    switch $EVENTNAME
      case IN_CLOSE_WRITE IN_MODIFY IN_MOVED_TO
        log adding to index
        log (recollindex -if $FILENAME)
      case IN_DELETE IN_MOVED_FROM
        log removing from index
        log (recollindex -e $FILENAME)
      case '*'
        echo Unknown event: $EVENTNAME
        log Unknown event: $EVENTNAME
        exit 2
    end
    log success.
    exit 0

There's a bit of extra stuff in there, such as setting up a log that I mostly used for debugging. What the script
mainly does is canonicalizing the filename and then feeding it to `recollindex`, either adding or removing it from
the index, depending on the filesystem event.

#### Desktop searching

The `recoll` GUI can be configured to use what it calls "external indexes", i.e. indexes that are not updated by its
own indexer but are just used for reading. So on my desktop I pointed `recoll` to use the index I had mounted from
the server. Depending on your particular setup, you may have to also configure a "path translation" in `recoll`. That
translates the path stored in the server-side index to a path relative to the mount directory on your local machine,
so you can actually open the files from the GUI. Note that the version of `recoll` in Ubuntu 14.04 does not have that
feature yet. I added the `recoll-backports` PPA to my machine to get it.

## Hardware!

So now all the software things are in place. I can OCR a document fully automatically and have it added to a
searchable index. Time to look for hardware to automate as much of the document acquisition as possible. As mentioned
earlier, while the HP OfficeJet has a good scanner and even an ADF, it only scans one-sided, so it is not really
useful for automated acqusition of double-sided documents.

I ended up buying a [Brother ADS-1600W](http://welcome.brother.com/sg-en/products-services/scanners/ads-1600W.html)
because it was quite affordable and has Linux drivers and the quality seemed acceptable. It turned out I didn't use
the Linux drivers at all. It connects to the wireless network and at the push of a button can write to a Samba share
or an FTP server. I already had Samba set up for some other appliances, so I simply added another user for the scanner.

All this was surprisingly easy. Thanks to `pdfocr` already existing, I spent in total maybe an afternoon tinkering
with the software side of things until all that worked the way I wanted it to. Adding the scanner into the mix took
no more than another half hour. Of course, there was some tinkering in the following weeks to adjust a few things,
but that was to be expected. For example, I have set up a few different scanning profiles, so documents that are
tax-relevant end up in a different directory, for easier collecting when it's time to do the taxes.

My typical document workflow now looks this:

1. Receive document and read it.
2. If it needs to be acted upon, put it in the big inbox and work on it no later than the following weekend. If not,
   or once that is done, proceed to step 3.
3. Drop the document into the scanner, hit the appropriate profile, and let it scan.
4. File the document into a binder - no further categorization is needed, just file it chronologically.

When I need a document, I use the `recoll` GUI to find it and look at it, which is usually all I need. If I actually
need the physical document, I look at the date on the document and use that to simply pick the right binder and retrieve
it from there. This simplifies my task enormously, since I don't have to decide on where to file anything anymore.
The only decisions I need to make are "Do I need to act on this?" and "Is this tax-relevant?".

Several months down the road from setting this up, I can say that I am very happy with this setup. It works very near
perfectly and has greatly lowered the perceived burden of dealing with documents.
