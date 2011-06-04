# monitorfiles

Monitor a set of files for changes and display the changes or execute a command
when (one of) the files change. Handy e.g. for running `make` or your
equivalent when a file changes.

_I was suprised I couldn't find something like this that worked reliably!_ So
here goes...

# Synopsis

Use one of these:

    monitorfiles -c 'make' file1 file2 file3
    monitorfiles --inotify -c 'make' file1 file2 file3
    monitorfiles --inotify -c './someCommand {} ; echo someCommand executed for {}' file1 file2 file3

`find` is your friend:

    find . -name '*.c' -o -name '*.h' | xargs monitorfiles -c make 

# Polling and `inotify` with the same API

I want the same API regardless of whether I use polling or `inotify` on linux.

On linux `inotify` is provided with the `-i` option but that code path is linux
specific, and even on linux, `inotify` doesn't work reliably everywhere (e.g.
over fuse with sshfs)

# Problems / Bugs / Issues

## Polling may show a file changed too soon

It does a stat() on each of the files. So if files changes take a long time,
notification could occur before writing is complete. E.g. during compilation,
notification could occur before compilation completes. That could well be nasty

This could be improved by only waiting for a file to settle: Only notifying
about a file, when it has been stable for <t> seconds. Some beginnings of this
are in the `settle` branch. This is somewhat complicated/useless by the fact
that some file systems (notably ext3) only have file system timestamp
resolutions of 1 second.

`inotify` (including `monitorfiles --inotify`) does not have this problem. It
allows one to listen to the CLOSE_WRITE event, so it doesn't have to guess when
writing is finished, like monitorfiles does.

## inotify on remote mounts

`inotify` (all tools based on inotify - including `monitorfiles --inotify`)
does a great job when the source of the changes is local. On an `sshfs` mount,
it works if the change originates from a process on localhost, but if the
change originates from a process on the remote machine (which it often does for
me), inotify doesn't trigger. For that, use monitorfile's polling mechanism
instead.

`inotify` is *way* more efficient than polling is. But sometimes polling is
needed for reliability, and efficiency doesn't matter.

## Are there other mechanisms like `inotify` on other platforms?

# Alternatives

There are other similar tools out there, and perhaps you want to try them out
instead / in addition. Here at least is why I still think my monitorfiles has
merrit compared to them:

If you find/have better open source alternatives, please let me know. I'd enjoy
nothing more than to cancel this because another more mature project exists
that does polling and inotify with the same API.

## `inotify`-based

### inotifywait

With this you'll achieve the same thing as `monitorfiles --inotify file...`.
I'll claim that monitorfiles API is simpler if what you want is to execute a
command when a file changes. `inotifywait` can be forced to execute an
arbitrary command like this:

    inotifywait -m /some/dir -r -e CLOSE_WRITE --format %w%f | \
    while read f ; do echo "File: $f changed" ; done

On debian, inotifywait is in the inotify-tools package. 

### iWatch

[iWatch](http://iwatch.sourceforge.net/index.html) is an `inotify` based
project. Its focus is somewhat different than mine, but for my purposes, it
didn't seem a good fit. mature. Insists on a XML command structure, but doesn't
e.g. escape '"' chars in its `-c command` parameter, so that this fails with an
XML parser error. I checked the code and the reason was that the XML was
generated directly from user input - no escaping done. So something like thi I
want reliability!

    iwatch -c 'echo "foo"' foo

Last release in 2009.

## gamin ( or less recently, fam )

[Gamin](http://people.gnome.org/~veillard/gamin/) is a file and directory
monitoring system defined to be a subset of the FAM (File Alteration Monitor)
system. This is a service provided by a library which allows to detect when a
file or a directory has been modified.

### fileschanged

This was the only gamin/fam based command line utility I could find. It didn't
work every time for me. Especially not over sshfs mounts, but have also seen it
not work e.g. on my /raid mount (ext3 over mdadm)

## dnotify

Only tracks changes to directories, not files. [inotify replaces
dnotify](http://www.kernel.org/pub/linux/kernel/people/rml/inotify/README).
