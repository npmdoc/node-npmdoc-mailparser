# api documentation for  [mailparser (v2.0.2)](https://github.com/andris9/mailparser#readme)  [![npm package](https://img.shields.io/npm/v/npmdoc-mailparser.svg?style=flat-square)](https://www.npmjs.org/package/npmdoc-mailparser) [![travis-ci.org build-status](https://api.travis-ci.org/npmdoc/node-npmdoc-mailparser.svg)](https://travis-ci.org/npmdoc/node-npmdoc-mailparser)
#### Parse e-mails

[![NPM](https://nodei.co/npm/mailparser.png?downloads=true)](https://www.npmjs.com/package/mailparser)

[![apidoc](https://npmdoc.github.io/node-npmdoc-mailparser/build/screenCapture.buildNpmdoc.browser._2Fhome_2Ftravis_2Fbuild_2Fnpmdoc_2Fnode-npmdoc-mailparser_2Ftmp_2Fbuild_2Fapidoc.html.png)](https://npmdoc.github.io/node-npmdoc-mailparser/build/apidoc.html)

![npmPackageListing](https://npmdoc.github.io/node-npmdoc-mailparser/build/screenCapture.npmPackageListing.svg)

![npmPackageDependencyTree](https://npmdoc.github.io/node-npmdoc-mailparser/build/screenCapture.npmPackageDependencyTree.svg)



# package.json

```json

{
    "author": {
        "name": "Andris Reinman"
    },
    "bugs": {
        "url": "https://github.com/andris9/mailparser/issues"
    },
    "dependencies": {
        "addressparser": "1.0.1",
        "he": "1.1.1",
        "html-to-text": "3.1.0",
        "iconv-lite": "0.4.15",
        "libmime": "3.1.0",
        "mailsplit": "4.0.0",
        "marked": "0.3.6"
    },
    "description": "Parse e-mails",
    "devDependencies": {
        "eslint-config-nodemailer": "^1.0.0",
        "grunt": "^1.0.1",
        "grunt-cli": "^1.2.0",
        "grunt-contrib-nodeunit": "^1.0.0",
        "grunt-eslint": "^19.0.0",
        "libbase64": "^0.1.0",
        "libqp": "^1.1.0",
        "random-message": "^1.1.0"
    },
    "directories": {},
    "dist": {
        "shasum": "ba0ef1a67cd9e2de582edba0a37823fcd0d4d51f",
        "tarball": "https://registry.npmjs.org/mailparser/-/mailparser-2.0.2.tgz"
    },
    "engines": {
        "node": ">=6.0.0"
    },
    "gitHead": "3bb6942a5c1e1034ad7fe55a92154995c43c1b27",
    "homepage": "https://github.com/andris9/mailparser#readme",
    "license": "EUPL-1.1",
    "main": "index.js",
    "maintainers": [
        {
            "name": "andris",
            "email": "andris@node.ee"
        }
    ],
    "name": "mailparser",
    "optionalDependencies": {},
    "readme": "ERROR: No README data found!",
    "repository": {
        "type": "git",
        "url": "git+https://github.com/andris9/mailparser.git"
    },
    "scripts": {
        "test": "grunt"
    },
    "version": "2.0.2"
}
```



# <a name="apidoc.tableOfContents"></a>[table of contents](#apidoc.tableOfContents)

#### [module mailparser](#apidoc.module.mailparser)
1.  [function <span class="apidocSignatureSpan">mailparser.</span>MailParser (config)](#apidoc.element.mailparser.MailParser)
1.  [function <span class="apidocSignatureSpan">mailparser.</span>simpleParser (input, callback)](#apidoc.element.mailparser.simpleParser)



# <a name="apidoc.module.mailparser"></a>[module mailparser](#apidoc.module.mailparser)

#### <a name="apidoc.element.mailparser.MailParser"></a>[function <span class="apidocSignatureSpan">mailparser.</span>MailParser (config)](#apidoc.element.mailparser.MailParser)
- description and source-code
```javascript
class MailParser extends Transform {
    constructor(config) {
        let options = {
            readableObjectMode: true,
            writableObjectMode: false
        };
        super(options);

        this.options = config || {};
        this.splitter = new Splitter();
        this.finished = false;
        this.waitingEnd = false;

        this.headers = false;

        this.endReceived = false;
        this.reading = false;
        this.errored = false;

        this.tree = false;
        this.curnode = false;
        this.waitUntilAttachmentEnd = false;
        this.attachmentCallback = false;

        this.hasHtml = false;
        this.hasText = false;

        this.text = false;
        this.html = false;
        this.textAsHtml = false;

        this.attachmentList = [];

        this.splitter.on('readable', () => {
            if (this.reading) {
                return false;
            }
            this.readData();
        });

        this.splitter.on('end', () => {
            this.endReceived = true;
            if (!this.reading) {
                this.endStream();
            }
        });

        this.splitter.on('error', err => {
            this.errored = true;
            if (typeof this.waitingEnd === 'function') {
                return this.waitingEnd(err);
            }
            this.emit('error', err);
        });
    }

    readData() {
        if (this.errored) {
            return false;
        }
        this.reading = true;
        let data = this.splitter.read();
        if (data === null) {
            this.reading = false;
            if (this.endReceived) {
                this.endStream();
            }
            return;
        }

        this.processChunk(data, err => {
            if (err) {
                if (typeof this.waitingEnd === 'function') {
                    return this.waitingEnd(err);
                }
                return this.emit('error', err);
            }
            setImmediate(() => this.readData());
        });
    }

    endStream() {
        this.finished = true;
        if (typeof this.waitingEnd === 'function') {
            this.waitingEnd();
        }
    }

    _transform(chunk, encoding, done) {
        if (!chunk || !chunk.length) {
            return done();
        }

        if (this.splitter.write(chunk) === false) {
            return this.splitter.once('drain', () => {
                done();
            });
        } else {
            return done();
        }
    }

    _flush(done) {
        setImmediate(() => this.splitter.end());
        if (this.finished) {
            return this.cleanup(done);
        }
        this.waitingEnd = () => this.cleanup(done);
    }

    cleanup(done) {
        if (this.curnode && this.curnode.decoder) {
            this.curnode.decoder.end();
        }
        setImmediate(() => {
            this.push(this.getTextContent());
            done();
        });
    }

    processHeaders(lines) {
        let headers = new Map();
        (lines || []).forEach(line => {
            let key = line.key;
            let value = ((libmime.decodeHeader(line.line) || {}).value || '').toString().trim();
            value = Buffer.from(value, 'binary').toString();
            switch (key) {
                case 'content-type':
                case 'content-disposition':
                case 'dkim-signature':
                    value = libmime.parseHeaderValue(value);
                    Object.keys(value && value.params || {}).forEach(key => {
                        try {
                            value.params[key] = libmime.decodeWords(value.params[key]);
                        } catch (E) {
                            // ignore, keep as is
                        }
                    });
                    break;
                case 'date':
                    value = new Date(value);
                    if (!value || value.toString() === 'Invalid Date' || !value.getTime()) {
                        // date parsing failed :S
                        value = new Date();
                    } ...
```
- example usage
```shell
n/a
```

#### <a name="apidoc.element.mailparser.simpleParser"></a>[function <span class="apidocSignatureSpan">mailparser.</span>simpleParser (input, callback)](#apidoc.element.mailparser.simpleParser)
- description and source-code
```javascript
(input, callback) => {
    let promise;
    if (!callback) {
        promise = new Promise((resolve, reject) => {
            callback = callbackPromise(resolve, reject);
        });
    }

    let mail = {
        attachments: []
    };

    let parser = new MailParser();

    parser.on('headers', headers => {
        mail.headers = headers;
    });

    parser.on('data', data => {
        if (data.type === 'text') {
            Object.keys(data).forEach(key => {
                if (['text', 'html', 'textAsHtml'].includes(key)) {
                    mail[key] = data[key];
                }
            });
        }

        if (data.type === 'attachment') {
            mail.attachments.push(data);

            let chunks = [];
            let chunklen = 0;
            data.content.on('readable', () => {
                let chunk;
                while ((chunk = data.content.read()) !== null) {
                    chunks.push(chunk);
                    chunklen += chunk.length;
                }
            });

            data.content.on('end', () => {
                data.content = Buffer.concat(chunks, chunklen);
                data.release();
            });
        }
    });

    parser.on('end', () => {
        ['subject', 'references', 'date', 'to', 'from', 'to', 'cc', 'bcc', 'message-id', 'in-reply-to', 'reply-to'].forEach(key => {
            if (mail.headers.has(key)) {
                mail[key.replace(/\-([a-z])/g, (m, c) => c.toUpperCase())] = mail.headers.get(key);
            }
        });

        parser.updateImageLinks((attachment, done) => done(false, 'data:' + attachment.contentType + ';base64,' + attachment.content
.toString('base64')), (err, html) => {
            if (err) {
                return callback(err);
            }
            mail.html = html;

            callback(null, mail);
        });
    });

    if (typeof input === 'string') {
        parser.end(Buffer.from(input));
    } else if (Buffer.isBuffer(input)) {
        parser.end(input);
    } else {
        input.pipe(parser);
    }

    return promise;
}
```
- example usage
```shell
n/a
```



# misc
- this document was created with [utility2](https://github.com/kaizhu256/node-utility2)
