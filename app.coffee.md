    TOC_URL = "http://parahumans.wordpress.com/table-of-contents/"
    


Set up and create structure
===========================

This will be our data structure that we use to hold the book's contents

It looks like `book.arc.chapter` using the full names as specified in the ToC
So 1.1 would be `book["Arc 1: Gestation"]["1.1"]` (see below about normalising chapter numbers)

Within each chapter, the object has a title and a contents based on the scraped page.
This means that when the ToC title disagrees with the title on the page, we will keep
both values (which seems like the closest thing to author intent we can get)

    book = {}
    ids = {}

First, download the table of contents page.

    $ = require 'cheerio'
    request = require 'request'

    console.log "Downloading table of contents..."
    request TOC_URL, (err, res, body) ->
      throw err if err
      console.log "Done."
      html = $.load body

Set up Arc objects from the ToC headers. We assume they contain the word 'Arc'.
There will be duplicates, but that doesn't matter because we can just set the object multiple times

      html(".entry-content :contains('Arc')").each (i, el) ->
        arcTitle = $(el).text().split('\n')[0]
        return unless match = arcTitle.match /^Arc (\d+)/
        arcNum = match[1]
        book[arcTitle] =
          chapters: {}
          id: "arc#{arcNum}"
          path: "arc#{arcNum}/index.xhtml"

Parse out the links and chapter titles. We have to be a bit sneaky here because earlier chapters used numbers like 1.01 and later ones use 14.1, plus it's a bit difficult to match arc titles to chapter titles, so we split the title up, normalise and re-generate it. This will totally break if the chapter title format changes.

      html('.entry-content a:not(.share-icon)').each (i, el) ->
        link = $(el).attr('href')
        title = $(el).text().trim()
        return unless link and title

Match `numbers.anything anything else`

        match = title.match /^(\d+)\.(.*?)(?:\s+(.*))?$/
        return unless match

        [_, arcNum, chapterNum, extendedTitle] = match

        #console.log [_, arcNum, chapterNum, extendedTitle]

Normalise the chapter number

        chapterNum = ''+parseInt(chapterNum) if !isNaN parseInt(chapterNum)
        chapterTitle = "#{arcNum}.#{chapterNum}"
        chapterTitle += " #{extendedTitle}" if extendedTitle

This is a bit of a silly way to match the arc number, but it should be pretty robust and the performance doesn't matter since we only do it once

        arc = null
        for arcTitle, _arc of book
          if arcTitle.match("Arc #{arcNum}")
            arc = _arc
            break
        
        throw "Couldn't find Arc #{arcNum} from #{chapterTitle} - probably a problem with the parser" unless arc

We need a unique ID for later. The chapter numbers aren't necessarily unique (there's a lot of blah.x)

        id = "chapter#{arcNum}.#{chapterNum}"
        n = 1
        while ids[id]
          id = "chapter#{arcNum}.#{chapterNum}-#{n++}"

Add our chapter object.

        arc.chapters[chapterTitle] =
          first: (parseInt(chapterNum) is 1)
          link: link
          id: id
          path: "arc#{arcNum}/#{id}.xhtml"

      delete book[k] for k of book when k isnt "Arc 1: Gestation"
      #delete book["Arc 1: Gestation"].chapters[k] for k of book["Arc 1: Gestation"].chapters when k isnt "1.1"

      downloadChapters()

And now our book object is populated properly. We just need to download and parse the contents.

    downloadChapters = ->
      remaining = 0
      for arcName, arc of book
        for chapterName, chapter of arc.chapters
          do (chapterName, chapter) ->
            remaining++
            console.log "Downloading", chapterName, "..."
            request chapter.link, (err, req, body) ->
              console.log "Finished", chapterName, "(#{remaining-1} remaining)"
              throw err if err

Merge the parsed result into our chapter object

              result = parseChapter body
              chapter[k] = v for k, v of result

              done()
              

      done = ->
        return unless --remaining is 0
        console.log "All chapters downloaded!"
        generateBook()

    



Parse/Generate Chapter Content
==============================

Parse the HTML we're given.

    parseChapter = (text) ->
      html = $.load text

      title = html('h1.entry-title').text()

Take the entry content and make it a new document (so we can call .xml on it later)

      innerhtml = $.load html('.entry-content').html()

Remove navigation links

      innerhtml("a:contains('Chapter')").closest('p').remove()

Remove share panel

      innerhtml('.sharedaddy').remove()

For some reason there are weird dir=ltr attributes everywhere

      innerhtml('[dir=ltr]').removeAttr('dir')

Make any necessary transformations (TODO: what are these?)

  * Inline square box breaks?
  * Inline headers like dates and such

Generate new XHTML from what we have. ePub requires valid XHTML and we have no guarantee that the input will be valid.

      contents = innerhtml.xml()

      {title, contents}


Generate ePub
=============

    AdmZip = require 'adm-zip'
    eco = require 'eco'
    fs = require 'fs'

    generateBook = ->
      console.log "Generating ePub file..."

Initialise a new zip container.

      zip = new require('node-zip')()

Add some boring requisite files.

      zip.file 'mimetype', fs.readFileSync('files/mimetype', 'utf8'), compression: 'STORE'

      zip.file 'META-INF/container.xml', fs.readFileSync('files/container.xml', 'utf8')

For some stupid reason eco clobbers the object I give it.

      stupidClone = (o) -> JSON.parse JSON.stringify o

Generate content.opf - book manifest

      zip.file 'OEBPS/content.opf', eco.render(fs.readFileSync('templates/content.opf', 'utf8'), {arcs: book})

Generate toc.ncx - table of contents

      zip.file 'OEBPS/toc.ncx', eco.render(fs.readFileSync('templates/toc.ncx', 'utf8'), {arcs: book})

Add title page

      zip.file 'OEBPS/title_page.xhtml', fs.readFileSync('files/title_page.xhtml', 'utf8')

Generate document per chapter

      for arcName, arc of book
        zip.file "OEBPS/#{arc.path}", eco.render(fs.readFileSync('templates/arc.xhtml', 'utf8'), arc: arc, title: arcName)
        for chapterName, chapter of arc.chapters
          zip.file "OEBPS/#{chapter.path}", eco.render(fs.readFileSync('templates/chapter.xhtml', 'utf8'), chapter: chapter)

Write the zip container out to an epub file

      data = zip.generate base64: false, compression: 'DEFLATE'
      fs.writeFileSync 'worm.epub', data, 'binary'

      console.log "Finished!"
