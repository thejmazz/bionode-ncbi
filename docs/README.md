# bionode-ncbi

Handwritten docs.

## Modules

Why document this? I like to have a general idea of what each required module
does. Most of these are near self-explanatory.

- `fs`, `path`, `mkdirp`, `debug`, `url`
- `async`, `request`
- `through2`, `tool-stream`, `concat-stream`, `pumpify`
- `xml2js`
- `nugget` (`wget` clone for node)
- `cheerio` (like jQuery without DOM)
- `bionode-fasta`

## Preamble

Need to add `http://cors.inb.io` if `window` is undefined (i.e. serverside).
API root is `http://eutils.ncbi.nlm.nih.gov/entrez/eutils/`. Default params is
`retmode=json&version=2.0`.

Where these used?
```javascript
var XMLPROPERTIES = {
  'sra': ['expxml', 'runs'],
  'biosample': ['sampledata'],
  'assembly': ['meta' ]
}
```

`LASTSTREAM` has `sra` which is a function that returns a stream checking if
`runs.Run` is an array and filters `total_bases` using `runs.Run`.

`RETURNMAX` is 50.



## Inner Functions

**createApiSearchUrl**

Returns an object transform stream that takes `obj` and returns a `query`:

```javascript
function transform (obj, enc, next) {
  var query = [
    APIROOT + 'esearch.fcgi?',
    DEFAULTS,
    'db=' + db,
    'term=' + encodeURI(obj.toString().replace(/['"]+/g, '')),
    'usehistory=y'
  ].join('&')
  debug('esearch request', query)
  this.push(query)
  next()
}
```

**createAPIPaginateURL**

`opts.throughput` will be used instead of `RETURNMAX`. Also
```javascript
if (opts.limit < throughput) { throughput = opts.limit }
```

As well returns an object transform stream:
```javascript
function transform (obj, enc, next) {
  var esearchRes = obj.body.esearchresult
  if (esearchRes === undefined
      || esearchRes.webenv === undefined
      || esearchRes.count === undefined) {
    var msg = 'NCBI returned invalid results, this could be a temporary' +
              ' issue with NCBI servers.\nRequest URL: ' + obj.url
    this.emit('error', new Error(msg))
    return next()
  }
  var count = opts.limit || esearchRes.count
  if (parseInt(esearchRes.count, 10) === 1) {
    this.push(obj.url)
    return next()
  }
  var urlQuery = URL.parse(obj.url, true).query
  var numRequests = Math.ceil(count / throughput)
  for (var i = 0; i < numRequests; i++) {
    var retstart = i * throughput
    var query = [
      APIROOT + 'esearch.fcgi?',
      DEFAULTS,
      'db=' + urlQuery.db,
      'term=' + urlQuery.term,
      'query_key=1',
      'WebEnv=' + esearchRes.webenv,
      'retmax=' + throughput,
      'retstart=' + retstart
    ].join('&')
    debug('paginate request', query)
    this.push(query)
  }
  next()
}
```

**createAPIDataURL**

```javascript
function transform (obj, enc, next) {
  var idsChunkLen = 50
  var idlist = obj.body.esearchresult.idlist
  if (!idlist || idlist.length === 0) { return next() }
  for (var i = 0; i < idlist.length; i += idsChunkLen) {
    var idsChunk = idlist.slice(i, i + idsChunkLen)
    var urlQuery = URL.parse(obj.url, true).query
    var query = [
      APIROOT + 'esummary.fcgi?',
      DEFAULTS,
      'db=' + urlQuery.db,
      'id=' + idsChunk.join(','),
      'usehistory=y'
    ].join('&')
    debug('esummary request', query)
    this.push(query)
  }
  next()
}
```

**fetchByID**

This is where `XMLPROPERTIES` gets used.

```javascript
function fetchByID (db) {
  var xmlProperties = XMLPROPERTIES[db] || through.obj()
  var lastStream = LASTSTREAM[db] || through.obj
  var stream = pumpify.obj(
    requestStream(true),
    tool.extractProperty('body.result'),
    tool.deleteProperty('uids'),
    tool.arraySplit(),
    tool.XMLToJSProperties(xmlProperties),
    lastStream()
  )
  return stream
}
```

**createAPILinkURL**

Takes `(srcDB, destDB)`, if `srcDB === 'tax'`, will make it `taxonomy`.

```javascript
function transform (obj, enc, next) {
  var query = [
    APIROOT + 'elink.fcgi?',
    'dbfrom=' + srcDB,
    'db=' + destDB,
    'id=' + obj.toString()
  ].join('&')
  this.push(query)
  next()
}
```

**createLinkObj**

```javasript
function transform (obj, enc, next) {
  var self = this
  var query = URL.parse(obj.url, true).query
  var result = {
    srcDB: query.dbfrom,
    destDB: query.db,
    srcUID: query.id
  }
  xml2js(obj.body, gotParsed)
  function gotParsed (err, data) {
    if (err) { self.emit('error', err); return next() }
    if (!data.eLinkResult.LinkSet[0].LinkSetDb) { return next() }
    data.eLinkResult.LinkSet[0].LinkSetDb.forEach(getMatch)
    self.push(result)
    next()
  }
  function getMatch (link) {
    var linkName = query.dbfrom + '_' + query.db
    if (link.LinkName[0] !== linkName) { return }
    var destUIDs = []
    link.Link.forEach(getLink)
    function getLink (link) { destUIDs.push(link.Id[0]) }
    result.destUIDs = destUIDs
  }
}
```

**download**

Takes `db`.

```javascript
function transform (obj, enc, next) {
  var self = this
  var folder = obj.uid + '/'

  var extractFiles = {
    'sra': function () { return obj.url },
    'gff': function () { return obj.genomic.gff },
    'gbff': function () { return obj.genomic.gbff },
    'gpff': function () { return obj.protein.gpff },
    'assembly': function () { return obj.genomic.fna },
    'fasta': function () { return obj.genomic.fna },
    'fna': function () { return obj.genomic.fna },
    'faa': function () { return obj.protein.faa },
    'repeats': function () { return obj.rm.out },
    'md5': function () { return obj.md5checksums.txt }
  }
  var url = extractFiles[db]()

  var path = folder + url.replace(/.*\//, '')

  var log = {
    uid: obj.uid,
    url: url,
    path: path
  }

  mkdirp(obj.uid, {mode: '0755'}, gotDir)
  function gotDir (err) {
    if (err) { self.emit('error', err) }
    debug('downloading', url)
    var options = { dir: folder, resume: true }
    var dld = nugget(PROXY + url, options, function (err) {
      if (err) return self.destroy(err)
      fs.stat(path, gotStat)
      function gotStat (err, stat) {
        if (err) return self.destroy(err)
        log.status = 'completed'
        log.speed = 'NA'
        log.size = Math.round(stat.size / 1024 / 1024) + ' MB'
        self.push(log)
        next()
      }
    })
    dld.on('progress', logging)
  }

  function logging (data) {
    log.status = 'downloading'
    log.total = data.transferred
    log.progress = data.percentage
    log.speed = data.speed
    self.push(log)
  }
}
```

**createFTPURL**

Takes `db`. Uses `async` for `eachSeries`. Users `cheerio` to parse FTP pages.
**This should be replaced by an internal NCBI API!!**

```javascript
function createFTPURL (db) {
  var stream = through.obj(transform)
  return stream

  function transform (obj, enc, next) {
    var self = this
    var parseURL = {
      sra: sraURL,
      assembly: assemblyURL
    }

    parseURL[db]()

    function sraURL () {
      var runs = obj.runs.Run
      async.eachSeries(runs, printSRAURL, next)
      function printSRAURL (run, cb) {
        var acc = run.acc
        var runURL = [
          'http://ftp.ncbi.nlm.nih.gov/sra/sra-instant/reads/ByRun/sra/',
          acc.slice(0, 3) + '/',
          acc.slice(0, 6) + '/',
          acc + '/',
          acc + '.sra'
        ].join('')
        self.push({url: runURL, uid: obj.uid})
        cb()
      }
    }

    function assemblyURL () {
      if (obj.meta.FtpSites) {
        var ftpPath = obj.meta.FtpSites.FtpPath
        var ftpArray = Array.isArray(ftpPath) ? ftpPath : [ ftpPath ]
        var httpRoot = ftpArray[0]._.replace('ftp://', 'http://') // NCBI seems to return GenBank and RefSeq accessions for the same thing. We only need one.
        request({ uri: PROXY + httpRoot, withCredentials: false }, gotFTPDir)
      } else { return next() }
      function gotFTPDir (err, res, body) {
        if (err) { self.emit('error', err) }
        if (!res || res.statusCode !== 200) { self.emit('err', res) }
        if (!body) { return next() }
        var $ = cheerio.load(body)

        var urls = { uid: obj.uid }

        $('a').map(attachToResult)
        function attachToResult (i, a) {
          var href = a.attribs.href
          var base = path.basename(href)
          var fileNameProperties = base.replace(/.*\//, '').split('_')
          var fileNameExtensions = fileNameProperties[fileNameProperties.length - 1].split('.')
          var fileType = fileNameExtensions[0]
          var fileFormat = fileNameExtensions[1] || 'dir'
          if (!urls[fileType]) { urls[fileType] = {} }
          urls[fileType][fileFormat] = httpRoot + '/' + href
        }
        self.push(urls)
        next()
      }
    }
  }
}
```

**requestStream**

Takes `returnURL`.

```javascript
function transform (obj, enc, next) {
  var self = this
  get()
  self.tries = 1
  function get () {
    if (self.tries > 20) { console.warn('try ' + self.tries + obj) }
    request({ uri: obj, json: true, timeout: timeout, withCredentials: false }, gotData)
    function gotData (err, res, body) {
      if (err
      || !res
      || res.statusCode !== 200
      || !body
      || (body.esearchresult && body.esearchresult.ERROR)
      || (body.esummaryresult && body.esummaryresult[0] === 'Unable to obtain query #1')
      || body.error
      ) {
        self.tries++
        return setTimeout(get, interval)
      }
      debug('request response', res.statusCode)
      debug('request results', body)
      var result = returnURL ? {url: obj, body: body} : body
      self.push(result)
      setTimeout(next, interval)
    }
  }
}
```

**stringifyExtras**

```javascript
function stringifyExtras (opts) {
  var extraOptsLine = ''

  for (var k in opts) {
    if ((k !== 'term') && (k !== 'db')) {
      extraOptsLine += k + '=' + opts[k] + '&'
    }
  }

  return extraOptsLine.slice(0, -1)
}
```


**createAPIFetchUrl**

```
function createAPIFetchUrl (opts, extraOpts) {
  var stream = through.obj(transform)
  return stream

  function transform (obj, enc, next) {
    var idsChunkLen = 50
    var idlist = obj.body.esearchresult.idlist
    if (!idlist || idlist.length === 0) { return next() }
    for (var i = 0; i < idlist.length; i += idsChunkLen) {
      var idsChunk = idlist.slice(i, i + idsChunkLen)
      var urlQuery = URL.parse(obj.url, true).query
      var query = [
        APIROOT + 'efetch.fcgi?',
        'version=2.0',
        'db=' + urlQuery.db,
        'id=' + idsChunk.join(','),
        extraOpts,
        'userhistory=y'
      ].join('&')
      debug('efetch request', query)
      this.push(query)
    }
    next()
  }
}
```

**parseResult**

```
function parseResult (resFmt) {
  var lastStream = (resFmt === 'fasta') ? fasta.obj : through.obj

  var stream = pumpify.obj(
      requestStream('true'),
      preProcess(),
      lastStream()
  )

  return stream

  function preProcess () {
    var stream = through.obj(transform)
    return stream

    function transform (chunk, enc, cb) {
      var self = this
      if (resFmt === 'xml') {
        xml2js(chunk.body, function (err, data) {
          if (err) { self.emit('error', err); return cb() }
          self.push(data)
          cb()
        })
      } else if (resFmt === 'fasta') {
        self.push(chunk.body)
        cb()
      } else {
        self.push({result: chunk.body})
        cb()
      }
    }
  }
}
```

## search

Takes `(db, term, cb)`. If `db` is a string then make `opts = { db, term }`,
else `opts = db`. If no `cb`, use `term` as `cb`.

```javascript
var stream = pumpify.obj(
  createAPISearchUrl(opts.db, opts.term),
  requestStream(true),
  createAPIPaginateURL(opts),
  requestStream(true),
  createAPIDataUrl(),
  fetchByID(opts.db)
)
```

## link

```javascript
var stream = pumpify.obj(
  createAPILinkURL(srcDB, destDB),
  requestStream(true),
  createLinkObj()
)
```

## plink

```javascript
ncbi.plink = function (property, destDB) {
  var srcDB = property.split('.').pop()
  var destProperty = destDB + 'id'
  var stream = through.obj(transform)
  return stream

  function transform (obj, enc, next) {
    var self = this
    var id = tool.getValue(obj, property + 'id')
    if (!id) {
      self.push(obj)
      return next()
    }
    if (!obj[destProperty]) { obj[destProperty] = [] }
    ncbi.link(srcDB, destDB, id, gotData)
    function gotData (data) {
      if (data.destUIDs) { obj[destProperty] = data.destUIDs }
      self.push(obj)
      next()
    }
  }
}
```

## download

```javascript
ncbi.download = function (db, term, cb) {
  var stream = pumpify.obj(
    ncbi.urls(db),
    download(db)
  )
  if (term) { stream.write(term); stream.end() }
  if (cb) { stream.pipe(concat(cb)) } else { return stream }
}
```

## urls

```javascript
var stream = pumpify.obj(
  ncbi.search(opts),
  createFTPURL(opts.db)
)
```

## expand

Takes `property` and `destProperty`.

```javascript
function transform (obj, enc, next) {
  var self = this
  var ids = tool.getValue(obj, property + 'id')
  if (!ids) {
    self.push(obj)
    return next()
  }

  // Taxonomy doesn't work just with ID number
  if (db === 'taxonomy') { ids = ids + '[uid]' }

  if (Array.isArray(ids)) {
    async.map(ids, search, gotData)
  } else {
    search(ids, gotData)
  }

  function search (term, cb) {
    var stream = ncbi.search(db)
    stream.write(term)
    stream.on('data', function (data) { cb(null, data) })
    stream.on('end', next)
  }

  function gotData (err, data) {
    if (err) { throw new Error(err) }
    obj[destProperty] = data
    self.push(obj)
    next()
  }
}
```

## fetch

Takes `db` and `term`.

```javascript
ncbi.fetch = function (db, term, cb) {
  var opts = typeof db === 'string' ? { db: db, term: term } : db
  cb = typeof term === 'function' ? term : cb

  var rettypes = {
    bioproject: 'xml',
    biosample: 'full',
    biosystems: 'xml',
    gds: 'summary',
    gene: '',
    homologene: 'fasta',
    mesh: 'full',
    nlmcatalog: 'xml',
    nuccore: 'fasta',
    nucest: 'fasta',
    nucgss: 'fasta',
    protein: 'fasta',
    popset: 'fasta',
    pmc: '',
    pubmed: '',
    snp: 'fasta',
    sra: 'full',
    taxonomy: ''
  }

  var retmodes = {
    fasta: 'fasta',
    'native': 'xml',
    full: 'xml',
    xml: 'xml',
    '': 'xml',
    'asn.1': 'asn.1'
  }

  opts.rettype = opts.rettype || rettypes[opts.db]
  opts.retmode = retmodes[opts.rettype] || 'text'

  var stream = pumpify.obj(
      createAPISearchUrl(opts.db, opts.term),
      requestStream(true),
      createAPIPaginateURL(opts),
      requestStream(true),
      createAPIFetchUrl(opts, stringifyExtras(opts)),
      parseResult(opts.retmode)
  )

  if (opts.term) { stream.write(opts.term); stream.end() }
  if (cb) { stream.pipe(concat(cb)) } else { return stream }
}
```
