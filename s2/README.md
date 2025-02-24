# S2 Compression

S2 is an extension of [Snappy](https://github.com/google/snappy).

S2 is aimed for high throughput, which is why it features concurrent compression for bigger payloads.

Decoding is compatible with Snappy compressed content, but content compressed with S2 cannot be decompressed by Snappy.
This means that S2 can seamlessly replace Snappy without converting compressed content.

S2 is designed to have high throughput on content that cannot be compressed.
This is important, so you don't have to worry about spending CPU cycles on already compressed data. 

## Benefits over Snappy

* Better compression
* Concurrent stream compression
* Faster decompression
* Ability to quickly skip forward in compressed stream
* Compatible with reading Snappy compressed content
* Offers alternative, more efficient, but slightly slower compression mode.
* Smaller block size overhead on incompressible blocks.
* Block concatenation
* Automatic stream size padding.
* Snappy compatible block compression.

## Drawbacks over Snappy

* Not optimized for 32 bit systems.
* Uses slightly more memory (4MB per core) due to larger blocks and concurrency (configurable).

# Usage

Installation: `go get -u github.com/klauspost/compress/s2`

Full package documentation:
 
[![godoc][1]][2]

[1]: https://godoc.org/github.com/klauspost/compress?status.svg
[2]: https://godoc.org/github.com/klauspost/compress/s2

## Compression

```Go
func EncodeStream(src io.Reader, dst io.Writer) error {
    enc := s2.NewWriter(dst)
    _, err := io.Copy(enc, src)
    if err != nil {
        enc.Close()
        return err
    }
    // Blocks until compression is done.
    return enc.Close() 
}
```

You should always call `enc.Close()`, otherwise you will leak resources and your encode will be incomplete.

For the best throughput, you should attempt to reuse the `Writer` using the `Reset()` method.

The Writer in S2 is always buffered, therefore `NewBufferedWriter` in Snappy can be replaced with `NewWriter` in S2.
It is possible to flush any buffered data using the `Flush()` method. 
This will block until all data sent to the encoder has been written to the output.

S2 also supports the `io.ReaderFrom` interface, which will consume all input from a reader.

As a final method to compress data, if you have a single block of data you would like to have encoded as a stream,
a slightly more efficient method is to use the `EncodeBuffer` method.
This will take ownership of the buffer until the stream is closed.

```Go
func EncodeStream(src []byte, dst io.Writer) error {
    enc := s2.NewWriter(dst)
    // The encoder owns the buffer until Flush or Close is called.
    err := enc.EncodeBuffer(buf)
    if err != nil {
        enc.Close()
        return err
    }
    // Blocks until compression is done.
    return enc.Close()
}
```

Each call to `EncodeBuffer` will result in discrete blocks being created without buffering, 
so it should only be used a single time per stream.
If you need to write several blocks, you should use the regular io.Writer interface.


## Decompression

```Go
func DecodeStream(src io.Reader, dst io.Writer) error {
    dec := s2.NewReader(src)
    _, err := io.Copy(dst, dec)
    return err
}
```

Similar to the Writer, a Reader can be reused using the `Reset` method.

For the best possible throughput, there is a `EncodeBuffer(buf []byte)` function available.
However, it requires that the provided buffer isn't used after it is handed over to S2 and until the stream is flushed or closed.  

For smaller data blocks, there is also a non-streaming interface: `Encode()`, `EncodeBetter()` and `Decode()`.
Do however note that these functions (similar to Snappy) does not provide validation of data, 
so data corruption may be undetected. Stream encoding provides CRC checks of data.

It is possible to efficiently skip forward in a compressed stream using the `Skip()` method. 
For big skips the decompressor is able to skip blocks without decompressing them.

## Single Blocks

Similar to Snappy S2 offers single block compression. 
Blocks do not offer the same flexibility and safety as streams,
but may be preferable for very small payloads, less than 100K.

Using a simple `dst := s2.Encode(nil, src)` will compress `src` and return the compressed result. 
It is possible to provide a destination buffer. 
If the buffer has a capacity of `s2.MaxEncodedLen(len(src))` it will be used. 
If not a new will be allocated. 

Alternatively `EncodeBetter`/`EncodeBest` can also be used for better, but slightly slower compression.

Similarly to decompress a block you can use `dst, err := s2.Decode(nil, src)`. 
Again an optional destination buffer can be supplied. 
The `s2.DecodedLen(src)` can be used to get the minimum capacity needed. 
If that is not satisfied a new buffer will be allocated.

Block function always operate on a single goroutine since it should only be used for small payloads.

# Commandline tools

Some very simply commandline tools are provided; `s2c` for compression and `s2d` for decompression.

Binaries can be downloaded on the [Releases Page](https://github.com/klauspost/compress/releases).

Installing then requires Go to be installed. To install them, use:

`go install github.com/klauspost/compress/s2/cmd/s2c && go install github.com/klauspost/compress/s2/cmd/s2d`

To build binaries to the current folder use:

`go build github.com/klauspost/compress/s2/cmd/s2c && go build github.com/klauspost/compress/s2/cmd/s2d`


## s2c

```
Usage: s2c [options] file1 file2

Compresses all files supplied as input separately.
Output files are written as 'filename.ext.s2'.
By default output files will be overwritten.
Use - as the only file name to read from stdin and write to stdout.

Wildcards are accepted: testdir/*.txt will compress all files in testdir ending with .txt
Directories can be wildcards as well. testdir/*/*.txt will match testdir/subdir/b.txt

File names beginning with 'http://' and 'https://' will be downloaded and compressed.
Only http response code 200 is accepted.

Options:
  -bench int
    	Run benchmark n times. No output will be written
  -blocksize string
    	Max  block size. Examples: 64K, 256K, 1M, 4M. Must be power of two and <= 4MB (default "4M")
  -c	Write all output to stdout. Multiple input files will be concatenated
  -cpu int
    	Compress using this amount of threads (default 32)
  -faster
    	Compress faster, but with a minor compression loss
  -help
    	Display help
  -pad string
    	Pad size to a multiple of this value, Examples: 500, 64K, 256K, 1M, 4M, etc (default "1")
  -q	Don't write any output to terminal, except errors
  -rm
    	Delete source file(s) after successful compression
  -safe
    	Do not overwrite output files
  -slower
    	Compress more, but a lot slower
  -verify
    	Verify written files  

```

## s2d

```
Usage: s2d [options] file1 file2

Decompresses all files supplied as input. Input files must end with '.s2' or '.snappy'.
Output file names have the extension removed. By default output files will be overwritten.
Use - as the only file name to read from stdin and write to stdout.

Wildcards are accepted: testdir/*.txt will compress all files in testdir ending with .txt
Directories can be wildcards as well. testdir/*/*.txt will match testdir/subdir/b.txt

File names beginning with 'http://' and 'https://' will be downloaded and decompressed.
Extensions on downloaded files are ignored. Only http response code 200 is accepted.

Options:
  -bench int
    	Run benchmark n times. No output will be written
  -c	Write all output to stdout. Multiple input files will be concatenated
  -help
    	Display help
  -q	Don't write any output to terminal, except errors
  -rm
    	Delete source file(s) after successful decompression
  -safe
    	Do not overwrite output files
  -verify
    	Verify files, but do not write output                                      
```

## s2sx: self-extracting archives

s2sx allows creating self-extracting archives with no dependencies.

By default, executables are created for the same platforms as the host os, 
but this can be overridden with `-os` and `-arch` parameters.

Extracted files have 0666 permissions, except when untar option used.

```
Usage: s2sx [options] file1 file2

Compresses all files supplied as input separately.
If files have '.s2' extension they are assumed to be compressed already.
Output files are written as 'filename.s2sfx' and with '.exe' for windows targets.
By default output files will be overwritten.

Wildcards are accepted: testdir/*.txt will compress all files in testdir ending with .txt
Directories can be wildcards as well. testdir/*/*.txt will match testdir/subdir/b.txt

Options:
  -arch string
        Destination architecture (default "amd64")
  -c    Write all output to stdout. Multiple input files will be concatenated
  -cpu int
        Compress using this amount of threads (default 32)
  -help
        Display help
  -os string
        Destination operating system (default "windows")
  -q    Don't write any output to terminal, except errors
  -rm
        Delete source file(s) after successful compression
  -safe
        Do not overwrite output files
  -untar
        Untar on destination
```

Available platforms are:

 * darwin-amd64
 * darwin-arm64
 * linux-amd64
 * linux-arm
 * linux-arm64
 * linux-mips64
 * linux-ppc64le
 * windows-386
 * windows-amd64                                                                             

### Self-extracting TAR files

If you wrap a TAR file you can specify `-untar` to make it untar on the destination host.

Files are extracted to the current folder with the path specified in the tar file.

Note that tar files are not validated before they are wrapped.

For security reasons files that move below the root folder are not allowed.

# Performance

This section will focus on comparisons to Snappy. 
This package is solely aimed at replacing Snappy as a high speed compression package.
If you are mainly looking for better compression [zstandard](https://github.com/klauspost/compress/tree/master/zstd#zstd)
gives better compression, but typically at speeds slightly below "better" mode in this package.

Compression is increased compared to Snappy, mostly around 5-20% and the throughput is typically 25-40% increased (single threaded) compared to the Snappy Go implementation.

Streams are concurrently compressed. The stream will be distributed among all available CPU cores for the best possible throughput.

A "better" compression mode is also available. This allows to trade a bit of speed for a minor compression gain.
The content compressed in this mode is fully compatible with the standard decoder.

Snappy vs S2 **compression** speed on 16 core (32 thread) computer, using all threads and a single thread (1 CPU):

| File                                                                                                | S2 speed | S2 Throughput | S2 % smaller | S2 "better" | "better" throughput | "better" % smaller |
|-----------------------------------------------------------------------------------------------------|----------|---------------|--------------|-------------|---------------------|--------------------|
| [rawstudio-mint14.tar](https://files.klauspost.com/compress/rawstudio-mint14.7z)                    | 12.70x   | 10556 MB/s    | 7.35%        | 4.15x       | 3455 MB/s           | 12.79%             |
| (1 CPU)                                                                                             | 1.14x    | 948 MB/s      | -            | 0.42x       | 349 MB/s            | -                  |
| [github-june-2days-2019.json](https://files.klauspost.com/compress/github-june-2days-2019.json.zst) | 17.13x   | 14484 MB/s    | 31.60%       | 10.09x      | 8533 MB/s           | 37.71%             |
| (1 CPU)                                                                                             | 1.33x    | 1127 MB/s     | -            | 0.70x       | 589 MB/s            | -                  |
| [github-ranks-backup.bin](https://files.klauspost.com/compress/github-ranks-backup.bin.zst)         | 15.14x   | 12000 MB/s    | -5.79%       | 6.59x       | 5223 MB/s           | 5.80%              |
| (1 CPU)                                                                                             | 1.11x    | 877 MB/s      | -            | 0.47x       | 370 MB/s            | -                  |
| [consensus.db.10gb](https://files.klauspost.com/compress/consensus.db.10gb.zst)                     | 14.62x   | 12116 MB/s    | 15.90%       | 5.35x       | 4430 MB/s           | 16.08%             |
| (1 CPU)                                                                                             | 1.38x    | 1146 MB/s     | -            | 0.38x       | 312 MB/s            | -                  |
| [adresser.json](https://files.klauspost.com/compress/adresser.json.zst)                             | 8.83x    | 17579 MB/s    | 43.86%       | 6.54x       | 13011 MB/s          | 47.23%             |
| (1 CPU)                                                                                             | 1.14x    | 2259 MB/s     | -            | 0.74x       | 1475 MB/s           | -                  |
| [gob-stream](https://files.klauspost.com/compress/gob-stream.7z)                                    | 16.72x   | 14019 MB/s    | 24.02%       | 10.11x      | 8477 MB/s           | 30.48%             |
| (1 CPU)                                                                                             | 1.24x    | 1043 MB/s     | -            | 0.70x       | 586 MB/s            | -                  |
| [10gb.tar](http://mattmahoney.net/dc/10gb.html)                                                     | 13.33x   | 9254 MB/s     | 1.84%        | 6.75x       | 4686 MB/s           | 6.72%              |
| (1 CPU)                                                                                             | 0.97x    | 672 MB/s      | -            | 0.53x       | 366 MB/s            | -                  |
| sharnd.out.2gb                                                                                      | 2.11x    | 12639 MB/s    | 0.01%        | 1.98x       | 11833 MB/s          | 0.01%              |
| (1 CPU)                                                                                             | 0.93x    | 5594 MB/s     | -            | 1.34x       | 8030 MB/s           | -                  |
| [enwik9](http://mattmahoney.net/dc/textdata.html)                                                   | 19.34x   | 8220 MB/s     | 3.98%        | 7.87x       | 3345 MB/s           | 15.82%             |
| (1 CPU)                                                                                             | 1.06x    | 452 MB/s      | -            | 0.50x       | 213 MB/s            | -                  |
| [silesia.tar](http://sun.aei.polsl.pl/~sdeor/corpus/silesia.zip)                                    | 10.48x   | 6124 MB/s     | 5.67%        | 3.76x       | 2197 MB/s           | 12.60%             |
| (1 CPU)                                                                                             | 0.97x    | 568 MB/s      | -            | 0.46x       | 271 MB/s            | -                  |
| [enwik10](https://encode.su/threads/3315-enwik10-benchmark-results)                                 | 21.07x   | 9020 MB/s     | 6.36%        | 6.91x       | 2959 MB/s           | 16.95%             |
| (1 CPU)                                                                                             | 1.07x    | 460 MB/s      | -            | 0.51x       | 220 MB/s            | -                  |

### Legend

* `S2 speed`: Speed of S2 compared to Snappy, using 16 cores and 1 core.
* `S2 throughput`: Throughput of S2 in MB/s. 
* `S2 % smaller`: How many percent of the Snappy output size is S2 better.
* `S2 "better"`: Speed when enabling "better" compression mode in S2 compared to Snappy. 
* `"better" throughput`: Speed when enabling "better" compression mode in S2 compared to Snappy. 
* `"better" % smaller`: How many percent of the Snappy output size is S2 better when using "better" compression.

There is a good speedup across the board when using a single thread and a significant speedup when using multiple threads.

Machine generated data gets by far the biggest compression boost, with size being being reduced by up to 45% of Snappy size.

The "better" compression mode sees a good improvement in all cases, but usually at a performance cost.

Incompressible content (`sharnd.out.2gb`, 2GB random data) sees the smallest speedup. 
This is likely dominated by synchronization overhead, which is confirmed by the fact that single threaded performance is higher (see above). 

## Decompression

S2 attempts to create content that is also fast to decompress, except in "better" mode where the smallest representation is used.

S2 vs Snappy **decompression** speed. Both operating on single core:

| File                                                                                                | S2 Throughput | vs. Snappy | Better Throughput | vs. Snappy |
|-----------------------------------------------------------------------------------------------------|---------------|------------|-------------------|------------|
| [rawstudio-mint14.tar](https://files.klauspost.com/compress/rawstudio-mint14.7z)                    | 2117 MB/s     | 1.14x      | 1738 MB/s         | 0.94x      |
| [github-june-2days-2019.json](https://files.klauspost.com/compress/github-june-2days-2019.json.zst) | 2401 MB/s     | 1.25x      | 2307 MB/s         | 1.20x      |
| [github-ranks-backup.bin](https://files.klauspost.com/compress/github-ranks-backup.bin.zst)         | 2075 MB/s     | 0.98x      | 1764 MB/s         | 0.83x      |
| [consensus.db.10gb](https://files.klauspost.com/compress/consensus.db.10gb.zst)                     | 2967 MB/s     | 1.05x      | 2885 MB/s         | 1.02x      |
| [adresser.json](https://files.klauspost.com/compress/adresser.json.zst)                             | 4141 MB/s     | 1.07x      | 4184 MB/s         | 1.08x      |
| [gob-stream](https://files.klauspost.com/compress/gob-stream.7z)                                    | 2264 MB/s     | 1.12x      | 2185 MB/s         | 1.08x      |
| [10gb.tar](http://mattmahoney.net/dc/10gb.html)                                                     | 1525 MB/s     | 1.03x      | 1347 MB/s         | 0.91x      |
| sharnd.out.2gb                                                                                      | 3813 MB/s     | 0.79x      | 3900 MB/s         | 0.81x      |
| [enwik9](http://mattmahoney.net/dc/textdata.html)                                                   | 1246 MB/s     | 1.29x      | 967 MB/s          | 1.00x      |
| [silesia.tar](http://sun.aei.polsl.pl/~sdeor/corpus/silesia.zip)                                    | 1433 MB/s     | 1.12x      | 1203 MB/s         | 0.94x      |
| [enwik10](https://encode.su/threads/3315-enwik10-benchmark-results)                                 | 1284 MB/s     | 1.32x      | 1010 MB/s         | 1.04x      |

### Legend

* `S2 Throughput`: Decompression speed of S2 encoded content.
* `Better Throughput`: Decompression speed of S2 "better" encoded content.
* `vs Snappy`: Decompression speed of S2 "better" mode compared to Snappy and absolute speed.


While the decompression code hasn't changed, there is a significant speedup in decompression speed. 
S2 prefers longer matches and will typically only find matches that are 6 bytes or longer. 
While this reduces compression a bit, it improves decompression speed.

The "better" compression mode will actively look for shorter matches, which is why it has a decompression speed quite similar to Snappy.   

Without assembly decompression is also very fast; single goroutine decompression speed. No assembly:

| File                           | S2 Throughput | S2 throughput |
|--------------------------------|--------------|---------------|
| consensus.db.10gb.s2           | 1.84x        | 2289.8 MB/s   |
| 10gb.tar.s2                    | 1.30x        | 867.07 MB/s   |
| rawstudio-mint14.tar.s2        | 1.66x        | 1329.65 MB/s  |
| github-june-2days-2019.json.s2 | 2.36x        | 1831.59 MB/s  |
| github-ranks-backup.bin.s2     | 1.73x        | 1390.7 MB/s   |
| enwik9.s2                      | 1.67x        | 681.53 MB/s   |
| adresser.json.s2               | 3.41x        | 4230.53 MB/s  |
| silesia.tar.s2                 | 1.52x        | 811.58        |

Even though S2 typically compresses better than Snappy, decompression speed is always better. 

## Block compression


When compressing blocks no concurrent compression is performed just as Snappy. 
This is because blocks are for smaller payloads and generally will not benefit from concurrent compression.

An important change is that incompressible blocks will not be more than at most 10 bytes bigger than the input.
In rare, worst case scenario Snappy blocks could be significantly bigger than the input.  

### Mixed content blocks

The most reliable is a wide dataset. 
For this we use `webdevdata.org-2015-01-07-subset`, 53927 files, total input size: 4,014,526,923 bytes. 
Single goroutine used.

| *                 | Input      | Output     | Reduction | MB/s   |
|-------------------|------------|------------|-----------|--------|
| S2                | 4014526923 | 1062282489 | 73.54%    | **861.44** |
| S2 Better         | 4014526923 | 981221284  | **75.56%** | 399.54 |
| Snappy            | 4014526923 | 1128667736 | 71.89%    | 741.29 |
| S2, Snappy Output | 4014526923 | 1093784815 | 72.75%    | 843.66 |

S2 delivers both the best single threaded throuhput with regular mode and the best compression rate with "better" mode. 

When outputting Snappy compatible output it still delivers better throughput (100MB/s more) and better compression.

As can be seen from the other benchmarks decompression should also be easier on the S2 generated output.  

### Standard block compression

Benchmarking single block performance is subject to a lot more variation since it only tests a limited number of file patterns.
So individual benchmarks should only be seen as a guideline and the overall picture is more important.

These micro-benchmarks are with data in cache and trained branch predictors. For a more realistic benchmark see the mixed content above. 

Block compression. Parallel benchmark running on 16 cores, 16 goroutines.

AMD64 assembly is use for both S2 and Snappy.

| Absolute Perf         | Snappy size | S2 Size | Snappy Speed | S2 Speed    | Snappy dec  | S2 dec      |
|-----------------------|-------------|---------|--------------|-------------|-------------|-------------|
| html                  | 22843       | 21111   | 16246 MB/s   | 17438 MB/s  | 40972 MB/s  | 49263 MB/s  |
| urls.10K              | 335492      | 287326  | 7943 MB/s    | 9693 MB/s   | 22523 MB/s  | 26484 MB/s  |
| fireworks.jpeg        | 123034      | 123100  | 349544 MB/s  | 273889 MB/s | 718321 MB/s | 827552 MB/s |
| fireworks.jpeg (200B) | 146         | 155     | 8869 MB/s    | 17773 MB/s  | 33691 MB/s  | 52421 MB/s  |
| paper-100k.pdf        | 85304       | 84459   | 167546 MB/s  | 101263 MB/s | 326905 MB/s | 291944 MB/s |
| html_x_4              | 92234       | 21113   | 15194 MB/s   | 50670 MB/s  | 30843 MB/s  | 32217 MB/s  |
| alice29.txt           | 88034       | 85975   | 5936 MB/s    | 6139 MB/s   | 12882 MB/s  | 20044 MB/s  |
| asyoulik.txt          | 77503       | 79650   | 5517 MB/s    | 6366 MB/s   | 12735 MB/s  | 22806 MB/s  |
| lcet10.txt            | 234661      | 220670  | 6235 MB/s    | 6067 MB/s   | 14519 MB/s  | 18697 MB/s  |
| plrabn12.txt          | 319267      | 317985  | 5159 MB/s    | 5726 MB/s   | 11923 MB/s  | 19901 MB/s  |
| geo.protodata         | 23335       | 18690   | 21220 MB/s   | 26529 MB/s  | 56271 MB/s  | 62540 MB/s  |
| kppkn.gtb             | 69526       | 65312   | 9732 MB/s    | 8559 MB/s   | 18491 MB/s  | 18969 MB/s  |
| alice29.txt (128B)    | 80          | 82      | 6691 MB/s    | 15489 MB/s  | 31883 MB/s  | 38874 MB/s  |
| alice29.txt (1000B)   | 774         | 774     | 12204 MB/s   | 13000 MB/s  | 48056 MB/s  | 52341 MB/s  |
| alice29.txt (10000B)  | 6648        | 6933    | 10044 MB/s   | 12806 MB/s  | 32378 MB/s  | 46322 MB/s  |
| alice29.txt (20000B)  | 12686       | 13574   | 7733 MB/s    | 11210 MB/s  | 30566 MB/s  | 58969 MB/s  |


| Relative Perf         | Snappy size | S2 size improved | S2 Speed | S2 Dec Speed |
|-----------------------|-------------|------------------|----------|--------------|
| html                  | 22.31%      | 7.58%            | 1.07x    | 1.20x        |
| urls.10K              | 47.78%      | 14.36%           | 1.22x    | 1.18x        |
| fireworks.jpeg        | 99.95%      | -0.05%           | 0.78x    | 1.15x        |
| fireworks.jpeg (200B) | 73.00%      | -6.16%           | 2.00x    | 1.56x        |
| paper-100k.pdf        | 83.30%      | 0.99%            | 0.60x    | 0.89x        |
| html_x_4              | 22.52%      | 77.11%           | 3.33x    | 1.04x        |
| alice29.txt           | 57.88%      | 2.34%            | 1.03x    | 1.56x        |
| asyoulik.txt          | 61.91%      | -2.77%           | 1.15x    | 1.79x        |
| lcet10.txt            | 54.99%      | 5.96%            | 0.97x    | 1.29x        |
| plrabn12.txt          | 66.26%      | 0.40%            | 1.11x    | 1.67x        |
| geo.protodata         | 19.68%      | 19.91%           | 1.25x    | 1.11x        |
| kppkn.gtb             | 37.72%      | 6.06%            | 0.88x    | 1.03x        |
| alice29.txt (128B)    | 62.50%      | -2.50%           | 2.31x    | 1.22x        |
| alice29.txt (1000B)   | 77.40%      | 0.00%            | 1.07x    | 1.09x        |
| alice29.txt (10000B)  | 66.48%      | -4.29%           | 1.27x    | 1.43x        |
| alice29.txt (20000B)  | 63.43%      | -7.00%           | 1.45x    | 1.93x        |

Speed is generally at or above Snappy. Small blocks gets a significant speedup, although at the expense of size. 

Decompression speed is better than Snappy, except in one case. 

Since payloads are very small the variance in terms of size is rather big, so they should only be seen as a general guideline.

Size is on average around Snappy, but varies on content type. 
In cases where compression is worse, it usually is compensated by a speed boost. 


### Better compression

Benchmarking single block performance is subject to a lot more variation since it only tests a limited number of file patterns.
So individual benchmarks should only be seen as a guideline and the overall picture is more important.

| Absolute Perf         | Snappy size | Better Size | Snappy Speed | Better Speed | Snappy dec  | Better dec  |
|-----------------------|-------------|-------------|--------------|--------------|-------------|-------------|
| html                  | 22843       | 19833       | 16246 MB/s   | 7731 MB/s    | 40972 MB/s  | 40292 MB/s  |
| urls.10K              | 335492      | 253529      | 7943 MB/s    | 3980 MB/s    | 22523 MB/s  | 20981 MB/s  |
| fireworks.jpeg        | 123034      | 123100      | 349544 MB/s  | 9760 MB/s    | 718321 MB/s | 823698 MB/s |
| fireworks.jpeg (200B) | 146         | 142         | 8869 MB/s    | 594 MB/s     | 33691 MB/s  | 30101 MB/s  |
| paper-100k.pdf        | 85304       | 82915       | 167546 MB/s  | 7470 MB/s    | 326905 MB/s | 198869 MB/s |
| html_x_4              | 92234       | 19841       | 15194 MB/s   | 23403 MB/s   | 30843 MB/s  | 30937 MB/s  |
| alice29.txt           | 88034       | 73218       | 5936 MB/s    | 2945 MB/s    | 12882 MB/s  | 16611 MB/s  |
| asyoulik.txt          | 77503       | 66844       | 5517 MB/s    | 2739 MB/s    | 12735 MB/s  | 14975 MB/s  |
| lcet10.txt            | 234661      | 190589      | 6235 MB/s    | 3099 MB/s    | 14519 MB/s  | 16634 MB/s  |
| plrabn12.txt          | 319267      | 270828      | 5159 MB/s    | 2600 MB/s    | 11923 MB/s  | 13382 MB/s  |
| geo.protodata         | 23335       | 18278       | 21220 MB/s   | 11208 MB/s   | 56271 MB/s  | 57961 MB/s  |
| kppkn.gtb             | 69526       | 61851       | 9732 MB/s    | 4556 MB/s    | 18491 MB/s  | 16524 MB/s  |
| alice29.txt (128B)    | 80          | 81          | 6691 MB/s    | 529 MB/s     | 31883 MB/s  | 34225 MB/s  |
| alice29.txt (1000B)   | 774         | 748         | 12204 MB/s   | 1943 MB/s    | 48056 MB/s  | 42068 MB/s  |
| alice29.txt (10000B)  | 6648        | 6234        | 10044 MB/s   | 2949 MB/s    | 32378 MB/s  | 28813 MB/s  |
| alice29.txt (20000B)  | 12686       | 11584       | 7733 MB/s    | 2822 MB/s    | 30566 MB/s  | 27315 MB/s  |


| Relative Perf         | Snappy size | Better size | Better Speed | Better dec |
|-----------------------|-------------|-------------|--------------|------------|
| html                  | 22.31%      | 13.18%      | 0.48x        | 0.98x      |
| urls.10K              | 47.78%      | 24.43%      | 0.50x        | 0.93x      |
| fireworks.jpeg        | 99.95%      | -0.05%      | 0.03x        | 1.15x      |
| fireworks.jpeg (200B) | 73.00%      | 2.74%       | 0.07x        | 0.89x      |
| paper-100k.pdf        | 83.30%      | 2.80%       | 0.07x        | 0.61x      |
| html_x_4              | 22.52%      | 78.49%      | 0.04x        | 1.00x      |
| alice29.txt           | 57.88%      | 16.83%      | 1.54x        | 1.29x      |
| asyoulik.txt          | 61.91%      | 13.75%      | 0.50x        | 1.18x      |
| lcet10.txt            | 54.99%      | 18.78%      | 0.50x        | 1.15x      |
| plrabn12.txt          | 66.26%      | 15.17%      | 0.50x        | 1.12x      |
| geo.protodata         | 19.68%      | 21.67%      | 0.50x        | 1.03x      |
| kppkn.gtb             | 37.72%      | 11.04%      | 0.53x        | 0.89x      |
| alice29.txt (128B)    | 62.50%      | -1.25%      | 0.47x        | 1.07x      |
| alice29.txt (1000B)   | 77.40%      | 3.36%       | 0.08x        | 0.88x      |
| alice29.txt (10000B)  | 66.48%      | 6.23%       | 0.16x        | 0.89x      |
| alice29.txt (20000B)  | 63.43%      | 8.69%       | 0.29x        | 0.89x      |

Except for the mostly incompressible JPEG image compression is better and usually in the 
double digits in terms of percentage reduction over Snappy.

The PDF sample shows a significant slowdown compared to Snappy, as this mode tries harder 
to compress the data. Very small blocks are also not favorable for better compression, so throughput is way down.

This mode aims to provide better compression at the expense of performance and achieves that 
without a huge performance penalty, except on very small blocks. 

Decompression speed suffers a little compared to the regular S2 mode, 
but still manages to be close to Snappy in spite of increased compression.  
 
# Best compression mode

S2 offers a "best" compression mode. 

This will compress as much as possible with little regard to CPU usage.

Mainly for offline compression, but where decompression speed should still
be high and compatible with other S2 compressed data.

Some examples compared on 16 core CPU, amd64 assembly used:

```
* enwik10
Default... 10000000000 -> 4761467548 [47.61%]; 1.098s, 8685.6MB/s
Better...  10000000000 -> 4219438251 [42.19%]; 1.925s, 4954.2MB/s
Best...    10000000000 -> 3667646858 [36.68%]; 35.995s, 264.9MB/s

* github-june-2days-2019.json
Default... 6273951764 -> 1043196283 [16.63%]; 431ms, 13882.3MB/s
Better...  6273951764 -> 949146808 [15.13%]; 547ms, 10938.4MB/s
Best...    6273951764 -> 846260870 [13.49%]; 8.125s, 736.4MB/s

* nyc-taxi-data-10M.csv
Default... 3325605752 -> 1095998837 [32.96%]; 324ms, 9788.7MB/s
Better...  3325605752 -> 954776589 [28.71%]; 491ms, 6459.4MB/s
Best...    3325605752 -> 794873295 [23.90%]; 6.619s, 479.1MB/s

* 10gb.tar
Default... 10065157632 -> 5916578242 [58.78%]; 1.028s, 9337.4MB/s
Better...  10065157632 -> 5649207485 [56.13%]; 1.597s, 6010.6MB/s
Best...    10065157632 -> 5246578570 [52.13%]; 25.696s, 373.6MB/s

* consensus.db.10gb
Default... 10737418240 -> 4562648848 [42.49%]; 882ms, 11610.0MB/s
Better...  10737418240 -> 4542428129 [42.30%]; 1.533s, 6679.7MB/s
Best...    10737418240 -> 4272335558 [39.79%]; 38.955s, 262.9MB/s
```

Decompression speed should be around the same as using the 'better' compression mode. 

# Concatenating blocks and streams.

Concatenating streams will concatenate the output of both without recompressing them. 
While this is inefficient in terms of compression it might be usable in certain scenarios. 
The 10 byte 'stream identifier' of the second stream can optionally be stripped, but it is not a requirement.

Blocks can be concatenated using the `ConcatBlocks` function.

Snappy blocks/streams can safely be concatenated with S2 blocks and streams. 

# Format Extensions

* Frame [Stream identifier](https://github.com/google/snappy/blob/master/framing_format.txt#L68) changed from `sNaPpY` to `S2sTwO`.
* [Framed compressed blocks](https://github.com/google/snappy/blob/master/format_description.txt) can be up to 4MB (up from 64KB).
* Compressed blocks can have an offset of `0`, which indicates to repeat the last seen offset.

Repeat offsets must be encoded as a [2.2.1. Copy with 1-byte offset (01)](https://github.com/google/snappy/blob/master/format_description.txt#L89), where the offset is 0.

The length is specified by reading the 3-bit length specified in the tag and decode using this table:

| Length | Actual Length        |
|--------|----------------------|
| 0      | 4                    |
| 1      | 5                    |
| 2      | 6                    |
| 3      | 7                    |
| 4      | 8                    |
| 5      | 8 + read 1 byte      |
| 6      | 260 + read 2 bytes   |
| 7      | 65540 + read 3 bytes |

This allows any repeat offset + length to be represented by 2 to 5 bytes.

Lengths are stored as little endian values.

The first copy of a block cannot be a repeat offset and the offset is not carried across blocks in streams.

Default streaming block size is 1MB.

# LICENSE

This code is based on the [Snappy-Go](https://github.com/golang/snappy) implementation.

Use of this source code is governed by a BSD-style license that can be found in the LICENSE file.
