# UtahSec Packet Analysis Workshop


---

## Overview

Download the file lookForFTP

---

## Challenge Background Information

A employee from your company has seemed to download something suspicious from a server!

We managed to get the employee's network traffic data.

Find out what credentials the employee used to log-in. 

Find out what was in the file the employee downloaded!

## Background Information for Challenge 2

Just find what websites I went on.

---

## Questions and Hints

### Question 1
If we know that the employee downloaded something from a server, what protocol would likely have been used?

<details>
  <summary>Click to reveal answer</summary>

  FTP. Apply the 'ftp' filter in wireshark and follow the stream by right-clicking the first packet -> follow -> TCP stream
</details>

---

### Question 2
How do we get the data in the file?

<details>
  <summary>Click to reveal answer</summary>

  There is another filter. 'ftp-data'. Apply this filter and click on packets to see their contents.
</details>

---

### Question 3 (Second File from google drive)
How do you find what websites I went on?
A.K.A what network protocol is responsible for converting website names into IP addresses?

<details>
  <summary>Click to reveal answer</summary>

  DNS protocol. Apply the filter 'dns' and look through to find names of websites.
</details>

---
