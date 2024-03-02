---
title: "Usefulness of Go templates in command line tools"
date: 2023-08-09T08:10:08+05:30
type: post
draft: false
tags: [Go, "Command line"]
---

Almost every time I see `awk` in a shell script, it is to extract a particular column - like `awk '{print $1}'`.

Much has been written and said about how the plain text output of command line tools isn't very flexible. Therefore powershell uses an object paradigm, where nobody understands what is going on. It appears a few tools, like Azure CLI have just defaulted to printing JSON. But this also breaks some long-developed intuitions like using `grep` or other UNIX tools.

A happy middle ground is giving the user more control of formatting the output. Many tools like `docker` use Go templates syntax for this task. For those who don't use Go, its standard library ships with a text templating DSL (Domain Specific Language).

The original purpose of this DSL seems to be creating server rendered web pages. However, it has found some application in cloud-related command line tools.

For example, default output of `docker ps` looks something like this.

```
$ docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                      NAMES
5c645402c13c   postgres   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds   127.0.0.1:5432->5432/tcp   psql-postgres-1
2f2f4ace447a   adminer    "entrypoint.sh php -…"   4 seconds ago   Up 2 seconds   127.0.0.1:8080->8080/tcp   psql-adminer-1
```

This is a bit complicated output to parse. Suppose we only need to print image names for use by some other script or tool.

```bash
$ docker ps --format '{{.Image}}'
adminer
postgres
```

It's also possible for the command line tool's author to provide extra functions to be used in templates. Docker [provides a few](https://docs.docker.com/config/formatting/).

I have used the same pattern in a small command line tool I wrote. [`rrip`](https://github.com/mahesh-hegde/rrip) is a command line tool which bulk-downloads images from Reddit. Using user-defined Go templates has been useful in quite a few places.

* rrip can log the posts it downloads to a file. Using go templates, you can provide a custom format for this.

* Change the filename format. Default format is "subreddit [post_id]", but user can specify a custom naming scheme using templates on command line.

* Not just formatting but also filtering - you can provide a template as `--template-filter`. If this template evaluates to "0", "nil" or "false", the post will be skipped.

Before writing the template, one will need to know which fields are there. In case of `docker`, you can pass `--format 'json'`. (Pipe the output to `jq` for neatly formatted and syntax-highlighted view) and inspect the available fields.

In case of `rrip` I implemented a `--print-post-data` option which prints the post JSON and does not really download it. So first you can inspect the JSON using this option, and know which fields are actually there.

```
$ rrip --print-post-data --max-files=1 r/Wallpaper
{
  "all_awardings": [],
  "allow_live_comments": false,
  "approved_at_utc": null,
  // ........................................... Output redacted
  "ups": 163,
  "upvote_ratio": 0.98,
  "url": "https://i.redd.it/6z7qmhwjrvgb1.png",
  "url_overridden_by_dest": "https://i.redd.it/6z7qmhwjrvgb1.png",
  "user_reports": [],
  "view_count": null,
  "visited": false,
  "whitelist_status": "all_ads",
  "wls": 6
}
Meep [3840x2160] [15lgw60].png                                                                      [Dry Run]
------------------------------------------------------------------------------------------------------------------------
Processed Posts:  1
Already Downloaded:  0
Failed:  0
Saved:  1
Other:  0
------------------------------------------------------------------------------------------------------------------------
Approx. Storage Used: 0B
------------------------------------------------------------------------------------------------------------------------
```

After knowning which fields are there, here are examples of few things we can do using templates.

```
## Example: only download gilded posts
rrip --template-filter='{{gt .gilded 0.0}}' --max-files=20 --sort=top-year r/Wallpapers

## Example: only download posts by a given author, say u/temporary_08
rrip --template-filter='{{eq .author "temporary_08"}}' --max-files=20  r/AMOLEDBackgrounds

## Example: Log links to a file with author, upvote ratio, and quoted title.
## Use dry run (-d) to skip download
rrip -d --data-output-file=wallpaper.txt --data-output-format='{{.upvote_ratio}} {{.author}} {{.quoted_title}}' r/Wallpapers

## Example: Change file name format using Go templates.
rrip --filename-format='{{.subreddit}} - {{.title}}' --search "Extraction 2"
```

I find it to be a pretty useful DSL when writing or using command line programs in Go.