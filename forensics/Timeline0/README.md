# Timeline 0

`Category: Forensics` · `Source: picoCTF` · `Difficulty: Medium`

> Can you find the flag in this disk image? Wrap what you find in the picoCTF flag format.

---

## First look

We get a file called `partition4.img.gz`, so the first
thing to do is unzip it:

```bash
gunzip partition4.img.gz
```

That leaves `partition4.img`, a 467 MB file. We check what it actually is:

```bash
file partition4.img
partition4.img: Linux rev 1.0 ext4 filesystem data ...
```

So it is a Linux disk image, an ext4 filesystem. Windows cannot open that directly, but I found out we can use `debugfs` to 
read an ext4 image without mounting it, so we can use `ls` to list a folder
inside the image and `cat` to print a file:

```bash
debugfs -R 'ls -l /' partition4.img
debugfs -R 'cat /root/.ash_history' partition4.img
```

We poke around the obvious spots this way, the home folders, `/root`, the logs. The history file in
`/root` is called `.ash_history`, not the usual `.bash_history`, so this is a small Alpine system on
the `ash` shell. It only contains `poweroff`, so nothing was edited or hidden through the shell. The
logs are just a plain boot and shutdown. We also search the whole image for the string `picoCTF`:

```bash
strings partition4.img | grep -i picoCTF
```

Nothing comes up.

![Poking around the image](assets/navigate.png)

The flag is not sitting in a file we can just read, so it is hidden some other way.

---

## Switching strategy

I decided here to use the hints given on picoctf, The challenge is called *Timeline*, and the
hints say to build a Sleuth Kit MAC timeline, and that sloppy timestomping can leave strange, very
old timestamps. Timestomping means changing a file's timestamps to hide when it was really created or
touched. So instead of reading files, we just have to look at *when* every file was modified, and watch for an anomaly with Sleuth Kit. `fls` lists every file with its timestamps, and `mactime` sorts them
into a readable timeline:

```bash
fls -r -m "/" partition4.img > timeline.body
mactime -b timeline.body -d > timeline.csv
```

To spot the anomaly without guessing, we look at how the dates are spread out, counting the files
per year:

```bash
awk -F',' 'NR>1 {split($1,a," "); print a[4]}' timeline.csv | sort | uniq -c
```

Almost everything sits in 2021, 2024 and 2025, the dates a normal system would have. One file sits
alone in 1985, decades before anything else. That is the outlier the hint warned about, so we pull
its line out:

```bash
grep "1985" timeline.csv
Tue Jan 01 1985 18:00:00,41,macb,...,4945,"/bin/bcab"
```

![The timestamp outlier](assets/outlier.png)

`/bin/bcab`, 41 bytes, hiding among the normal system binaries but with a date decades older than
everything around it. That is our timestomped file.

---

## Reading the file

Trying to read it the normal way fails:

```bash
debugfs -R 'cat /bin/bcab' partition4.img
cat: Inode checksum does not match inode while reading inode 4945
```

I did not expect that error, but it fits once you look it up. ext4 keeps a checksum on each inode to
catch tampering, and whoever faked the timestamps edited the inode directly without recomputing it,
so ext4 now refuses to trust it. The error is really just more proof the file was messed with. Sleuth
Kit's `icat` reads the raw structures and ignores that check, so it pulls the file out anyway using
the inode number from the timeline (4945):

```bash
icat partition4.img 4945
NzFtMzExbjNfMHU3MTEzcl9oM3JfNDNhMmU3YWYK
```

The content is base64. We decode it:

```bash
icat partition4.img 4945 | base64 -d
71m311n3_0u7113r_h3r_43a2e7af
```

![Extracting and decoding the file](assets/flag.png)

---

## Getting the flag

Wrapping it in the picoCTF format the challenge asked for:

```
picoCTF{71m311n3_0u7113r_h3r_43a2e7af}
```

The leetspeak spells "timeline outlier", which is exactly how we caught it, the one file whose
timestamp did not belong.
