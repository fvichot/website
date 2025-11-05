+++
title = "A Zip archive from GZip files"
date = 2025-09-24
[taxonomies]
tags = ["code"]
[extra]
toc = true
+++

Ever had to write an endpoint for compressing several user-selected files and returning them as a single archive ? If you have, you've probably grappled with the various trade-offs: how to make this endpoint not use too much memory, how to avoid paying the compression cost everytime etc.

Here I want to share a neat trick, whereby we store several large files in GZip format somewhere (in a database for example), and if we need to retrieve several of them at once, we create a Zip archive on the fly **without decompressing the Gzip files**, but instead by extracting the compressed section of each GZip file and sticking it as-is in the Zip archive.

This allows:
1. Paying the compression cost once at file insertion, and saving storage space in the database and on the wire between the DB and backend
1. Sending individual gzip file with the `Content-Encoding` HTTP header set to `gzip`, causing it to be transferred compressed from the DB to the client without a decompression step, and be uncompressed transparently by the remote client
1. Creating a Zip archive cheaply, as we only need to generate the header and footer, but not actually compress the files
1. Stream all of this to the client, so we only need a constant amount of memory

## How ?

This hinges on the realisation that the Zip archive format supports the `ZIP_DEFLATE` algorithm for individual files, and that this is effectively the same algorithm as the one in GZip. And since browsers natively support `gzip` as a `Content-Encoding` compression algorithm, we get all the benefits listed above.

The GZip file format is the following (from [RFC-1952](https://datatracker.ietf.org/doc/html/rfc1952#page-5)):
```
         +---+---+---+---+---+---+---+---+---+---+
         |ID1|ID2|CM |FLG|     MTIME     |XFL|OS | (more-->)
         +---+---+---+---+---+---+---+---+---+---+
      (if FLG.FEXTRA set)
         +---+---+=================================+
         | XLEN  |...XLEN bytes of "extra field"...| (more-->)
         +---+---+=================================+
      (if FLG.FNAME set)
         +=========================================+
         |...original file name, zero-terminated...| (more-->)
         +=========================================+
      (if FLG.FCOMMENT set)
         +===================================+
         |...file comment, zero-terminated...| (more-->)
         +===================================+
      (if FLG.FHCRC set)
         +---+---+
         | CRC16 |
         +---+---+
         +=======================+
         |...compressed blocks...| (more-->)
         +=======================+
           0   1   2   3   4   5   6   7
         +---+---+---+---+---+---+---+---+
         |     CRC32     |     ISIZE     |
         +---+---+---+---+---+---+---+---+
```

The only annoying part here is the conditional flags, but it's also not terribly hard to parse them
and adjust the header length accordingly:

```python,linenos
def extract(data):
    # We need to parse the Gzip header to skip it, as it has optional fields
    header_len = 10
    id1, id2, cm, flg, mtime, xfl, os = struct.unpack("<BBBBIBB", data[:10])
    if id1 != 0x1f or id2 != 0x8b:
        raise ValueError("Bad IDs")
    if flg & (1 << 2):  # FEXTRA
        xlen = struct.unpack("<H", data[header_len:header_len + 2])
        header_len += xlen + 2
    if flg & (1 << 3):  # FNAME
        while data[header_len] != 0:
            header_len += 1
    if flg & (1 << 4):  # FCOMMENT
        while data[header_len] != 0:
            header_len += 1
    if flg & (1 << 1):  # FHCRC
        header_len += 2

    footer_len = 8
    # Return the compressed data, stripped of its GZip header and footer
    return data[header_len:-footer_len]
```

Now to create a `Zip` archive, we create a `ZipFileFromGzip` class that derives from `zipfile.Zipfile`,
and override as necessary:

```python,linenos,hl_lines=24-27 40-41
class ZipFileFromGzip(zipfile.ZipFile):
    class GzipStripper:
        def compress(self, data):
            # We need to parse the Gzip header to skip it, as it has optional fields
            header_len = 10
            id1, id2, cm, flg, mtime, xfl, os = struct.unpack("<BBBBIBB", data[:10])
            if id1 != 0x1f or id2 != 0x8b:
                raise ValueError("Bad IDs")
            if flg & (1 << 2):  # FEXTRA
                xlen = struct.unpack("<H", data[header_len:header_len + 2])
                header_len += xlen + 2
            if flg & (1 << 3):  # FNAME
                while data[header_len] != 0:
                    header_len += 1
            if flg & (1 << 4):  # FCOMMENT
                while data[header_len] != 0:
                    header_len += 1
            if flg & (1 << 1):  # FHCRC
                header_len += 2

            # We save the CRC32 and the uncompressed size from the footer, as we'll need them when
            # writing the data to the zip file
            footer_len = 8
            self.crc32, self.isize = struct.unpack("<II", data[-footer_len:])

            # Return the compressed data, stripped of its GZip header and footer
            return data[header_len:-footer_len]

        def flush(self, *args, **kwargs):
            return bytes()

    def _write(self, data):
        # Call _ZipWriteFile.write
        self.oldwrite(data)
        # Patch the values for the CRC and file_size, which would otherwise be those of the
        # compressed data, instead of the uncompressed data
        self._crc = self._compressor.crc32
        self._file_size = self._compressor.isize
        return self._file_size

    def _open_to_write(self, zinfo, force_zip64=False):
        zip = super()._open_to_write(zinfo, force_zip64)
        # We replace the DEFLATE compressor with a fake compressor that simply extracts the
        # DEFLATE'd data from GZip
        zip._compressor = ZipFileFromGzip.GzipStripper()
        zip.oldwrite = zip.write
        zip.write = MethodType(ZipFileFromGzip._write, zip)
        return zip
```

We override the `_open_to_write` method so we can replace the compressor implementation with ours,
which is a small modification of the `extract()` method from before, with an added step to save the
original (uncompressed) file size (`isize`) and the CRC32, which are both needed to be patched in the
`_write` method, to make sure they are the ones from the uncompressed data, not the ones calculated
by the Python implementation on the gzip'd data.

It does mean we waste some cycles calculating a CRC32 we don't use. I haven't investigated how to
avoid that as the cost seemed marginal.


## Why `gzip` and not `deflate`?

Technically, we could also compress files using `DEFLATE`, as `Content-Encoding` also supports `deflate`, and since we use `ZIP_DEFLATE` in the Zip archive, maybe this would be more straight forward.

Sadly no: the `deflate` content encoding actually means a `DEFLATE` compressed stream, [inside a Zlib data format](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Encoding#deflate), whereas Zip expects the raw `DEFLATE` stream with no encoding. So we would have to do the same as for `gzip`: extract the compressed section and skip the header and footer. So an equivalent amount of work (though the Zlib format is simpler than GZip).

But then, we'd run into another issue: the Zip archive format wants to know the size of the data **before** compression, which is not retained in the Zlib format (but is in the GZip format as we saw above). So we would have to save that in the DB somehow, which adds complexity.

## What is this tested on ?

Obviously, because the code above relies on patching the internals of `zipfile.Zipfile`, it's a bit
brittle, but I did test on cpython 3.8 to 3.13, so it's got decent support:
```
$ for v in {8..13}; do \
    echo "Python 3.$v:"; \
    uvx --python 3.$v pytest -qsx ./test_utils_zip.py ; \
  done
Python 3.8:
...
3 passed in 0.69s
Python 3.9:
...
3 passed in 0.71s
Python 3.10:
...
3 passed in 0.68s
Python 3.11:
...
3 passed in 0.70s
Python 3.12:
...
3 passed in 0.70s
Python 3.13:
...
3 passed in 0.35s
```