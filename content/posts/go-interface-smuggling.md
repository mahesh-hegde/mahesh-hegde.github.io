---
title: "A tale of Interface Smuggling in Go net/http"
date: 2024-03-01T18:59:00+05:30
draft: false
type: post
tags: [Go, HTTP, Debugging]
showTableOfContents: true
---

## The humble beginnings

Once upon a time, I had a use case for serving the contents inside of a zip file over HTTP, preferably without unpacking the `.zip` to a temporary directory.

Go is a language made for writing HTTP servers. Since we can represent zip file as a filesystem `fs.FS`, how hard would it be to write a file server to serve files from it?

Well, it was harder than I thought.

For sake of this demonstration, let's create a zip file.

```bash
echo "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed vel velit non ipsum condimentum ornare. Sed quis tellus facilisis, rhoncus purus eget, dapibus libero. Sed sed diam semper, malesuada turpis non, vestibulum urna. Quisque et consequat libero. Vestibulum tempor elementum diam, a consectetur risus hendrerit sed." > sample_text
zip -r sample.zip sample_text
```

Now this is very straightforward Go code - to open a zip file, wrap it in `http.FileSystem` and attach the handler to HTTP server.

```go
package main

import (
	"archive/zip"
	"log"
	"net/http"
	"os"
)

func main() {
	zipPath := os.Args[1]
	zipFile, _ := os.Open(zipPath)
	zipFileStat, _ := zipFile.Stat()
	zipReader, _ := zip.NewReader(zipFile, zipFileStat.Size())

	mux := http.NewServeMux()
	fileserver := http.FileServer(http.FS(zipReader))

	mux.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
		defer func() {
			err := recover()
			if err != nil {
				log.Printf("%v [%s] %v: %v\n", req.RemoteAddr, req.Method, req.RequestURI, err)
			}
		}()
		fileserver.ServeHTTP(w, req)
	})
    log.Println("Listening on http://127.0.0.1:8080")
	log.Fatal(http.ListenAndServe("127.0.0.1:8080", mux))
}
```

Pretty straightforward. Now let's try it.

```
[~/Code/zip-server] $ go run main.go sample.zip
2024/03/01 21:14:24 Listening on http://127.0.0.1:8080
# On other terminal
[~/Code/zip-server] $ curl http://localhost:8080/sample_text
seeker can't seek
```

What nonsense?

Upon reading the error message, we get a clue - that HTTP server is trying to seek inside the files we are providing - which are not real files but entries inside zip files. Since these are compressed, they are not seekable.

However, it's said nowhere in the interface signatures that the files need to be seekable, we are providing `fs.FS`, and `Open()` on an `FS` will return `fs.File`, which is only required to implement `Read`, `Stat` and `Close`.

Digging around a bit, it's possible to find the [documentation](https://pkg.go.dev/net/http#FS) which says filesystem provided to `http.FS` must return seekable files. (However it doesn't explain why).

Now I'm curious, why? (After all, I am just linearly downloading files, I am not making any HTTP range requests.)

```
[~/.local/go/src] $ grep -rn "seeker can't seek"
net/http/fs.go:213:var errSeeker = errors.New("seeker can't seek")
net/http/fs.go:245:                             Error(w, "seeker can't seek", StatusInternalServerError)
```

Clicking around, I find `serveFile` and `ServeContent`. Using the VSCode debugger, I find out it's actually `serveFile` which is called during our `fileserver.ServeHTTP(w, req)` call.

![ServeHTTP in VSCode debugger](/images/go-interface-smuggling/servefile-debug.png)

`serveFile` in turn calls `serveContent`, where the seek-back happens after calling `DetectContentType`. This `DetectContentType` is called when the file type can't be determined using file extension. [^seek_end]

Turns out seeking is used to determine a. Content-Length header, and the MIME type of the file in certain cases.

Now I am tired of clicking around, back to the stone age tools. Since `fs.File` does not implement seeker, someone's gotta be casting it to `io.Seeker` or a related type. It's probably the `http.FS` because it's returning us the filesystem.

I am tired of clicking around anyway, and more than that, perhaps, fascinated by the stone age tools. Using `grep` again:

```
[~/.local/go/src] $ grep -rn 'io.Seeker)'
archive/tar/reader.go:861:      if sr, ok := r.(io.Seeker); ok && n > 1 {
net/http/fs.go:793:     s, ok := f.file.(io.Seeker)
embed/internal/embedtest/embed_test.go:213:     seeker := file.(io.Seeker)
internal/zstd/zstd.go:316:      if seeker, ok := r.r.(io.Seeker); ok {
```

And here we go, in `net/http/fs.go`:

```go
func (f ioFile) Seek(offset int64, whence int) (int64, error) {
	s, ok := f.file.(io.Seeker)
	if !ok {
		return 0, errMissingSeek
	}
	return s.Seek(offset, whence)
}
```

`ioFile` is the file type returned by `ioFS`, which is the wrapper type returned by `http.FS` as an implementer of `http.FileSystem`.

```go
func FS(fsys fs.FS) FileSystem {
	return ioFS{fsys}
}
```

__TL;DR - `net/http` expects `fs.File` objects returned by our "virtual" filesystem, to implement `io.Seeker`, which is not the part of `fs.File` interface contract.__

![That is not part of our original agreement](/images/go-interface-smuggling/ragnar-lothbrok-not-original-agreement.png)

## Interlude
This pattern, where the library is casting a more general interface into a more specific type, is called [Interface Smuggling](https://utcc.utoronto.ca/~cks/space/blog/programming/GoInterfaceSmuggling).

It's done extensively in Go's standard library, often to give a performance boost to generic operations in frequent cases, without breaking the abstractions. Here's a line of documentation for `io.WriteString`.

```
func WriteString(w Writer, s string) (n int, err error)
    WriteString writes the contents of the string s to w, which accepts a slice
    of bytes. If w implements StringWriter, its WriteString method is invoked
    directly. Otherwise, w.Write is called exactly once.
```

This is because the WriteString may be implemented more efficiently in `StringWriter`, compared to the standard way of converting string to bytes and writing it. For instance, a `strings.Builder` may just store a reference to the string instead of converting it to bytes.

This type of interface smuggling does not usually break the results of the code. But what we encountered here is more severe - the expectation of files being seekable is not communicated in the type system.

Anyway, coming back to the problem..

## The solution

Now all we have to do is fake a seek method in zip entries (we don't have a vocabulary for that in Go (unlike Java which calls them `ZipEntry`), since `zip.Reader` returns plain generic `fs.File`).

There are two ways to fake a seek method on a non-seekable, virtual file. One is easy - just load the entire file into memory, and return a bytes.Reader which is already given by the stdlib. This works reasonably well.

You would not want to read this much code in nested markdown blocks, so [here](https://github.com/mahesh-hegde/zserv/blob/main/zip_reader.go) is a link to full implementation on github, along with some "tests".

The other way is to look at the implementation of `'net/http'`, and implement just enough `Seek` behavior, leaving the rest of it undefined.

If we look at `ServeContent`, all it tries is seeking to start and to end. I implemented a proxy `ReadSeeker` which can pretend to do that. We can also observe in the code that it can read at most `sniffLen` bytes from the beginning, which is defined as 512 bytes. I just added a small buffer to my implementation which stored first 1024 bytes, making the seek possible. Seek to end will just return the size of the Zip entry.

The code for this is here: [streaming_zip_reader.go](https://github.com/mahesh-hegde/zserv/blob/main/streaming_zip_reader.go). I will just paste the interesting part:

```go
type StreamingZipEntryReader struct {
	fs.File
	*bytes.Reader
}

func (s *StreamingZipEntryReader) Read(buffer []byte) (int, error) {
	if s.Reader.Len() == 0 {
		return s.File.Read(buffer)
	}
	res, err := s.Reader.Read(buffer)
	if err != nil && errors.Is(err, io.EOF) {
		return res, nil
	}
	return res, err
}

func (s *StreamingZipEntryReader) Seek(offset int64, whence int) (int64, error) {
	if whence == io.SeekStart && offset == 0 {
		return s.Reader.Seek(0, io.SeekStart)
	}
	if whence == io.SeekEnd && offset == 0 {
		stat, err := s.File.Stat()
		if err != nil {
			return -1, err
		}
		return stat.Size(), nil
	}
	return -1, fmt.Errorf("unsupported seek parameters")
}
```

Both implementations seem to perform well when zip contains small HTML / PNG files.

However, when I tested with a zip file containing 1 GiB entry, the streaming version obviously performed much better, both in terms of speed and maximum resident set size (i.e peak memory usage).

![some measurements (and some shell scripting stunts)](/images/go-interface-smuggling/stats_zserv.png)

## Appendix / Miscellaneous
I used the filename `sample_txt` because if we named it `sample.txt`, the operation for Detecting MIME type would've never beeen triggered.

This was around an year and half ago. Since then, I made a small tool out of this code, and named it [`zserv`](https://github.com/mahesh-hegde/zserv), in the proud tradition of hard-to-pronounce and short unix tool names.

There has also [been](https://github.com/golang/go/issues/61791) [some](https://github.com/golang/go/issues/61791) [discussion](https://github.com/golang/go/issues/42173) on this behavior of `net/http` in the Go issue tracker.


[^seek_end]: (Interestingly there's also `ServeContent` (public method), which takes a `io.ReadSeeker` uses a "seek-to-end" operation to determine file size, to set in `Content-Length`. Since we have an `fs.File` in this case, which supports `stat`, this is not necessary.)
