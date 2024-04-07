---
layout: post
title: file integrity scanner
tags: file integrity scan hash
categories: code
---

_Tracking file integrity and system file changes_

Here is some [python code](https://github.com/T4ngles/systools/tree/systools_dev/file_integrity) I've put together with the goal of tracking file integrity on my system and be more aware of my environment. Reading about file hashes and the I in CIA gave me the very basic idea of tracking file hashes on a computer to understand how the system is changing so that I could have a baseline of what is normal change such as a windows update, downloading some photos, installing a new program vs a worm infecting files or any malware installing additional files/tools.

I'll do a quick run through of each component and then how I plan to go for the future.

### file_hasher.py

This script is a basic wrapper around the standard python hashlib so that I can do some custom error handling, filepath parsing and also lets me use it to check file downloads against provided hashes.

### file_walker.py

The file walker script is also a wrapper around os.walk where I've added some console printing to show directory traversal and also change what I yield to the `sys_health.py` script.

### sys_health.py

So this is the heavy lifting script which is due for splitting into seperate modules if not just functions. The basic flow of the script is to call the file_walker to traverse through a given path, my C drive, and yield all files over to the file_hasher to create file hashes for storage in a `.csv` file containing both the file hash and the filepath. This script also reads in the latest log created which is used to compare against the current file hashes it finds, adding any new fiels and hashes to a new_files dictionary for export to a `.txt` file at the end for analysis. Similarly, if duplicates of hashes are found these are also stored in a duplicate_files dictionary for printing out to the console at the end. I commented out writing the duplicates to a file as it was a large file and didn't seem to be that useful for security analysis, and there were a lot of duplicates, a lot.

After running this script daily for a few days I noticed my system files change frequently, and new files ranged from 2000 to 116000 when a new windows update is installed. Wanting to ignore some of the file paths that housed many standard new files/file changes and also wanting to avoid the [labyrinth]({% post_url 2024-03-08-labyrinth %}) I set up, I also added functionality to parse a pathsToIgnore.txt file to ignore certain paths. Reading the logs of new files daily gave me more of a familiarity with how my system changes and also alerted me to several programs (now uninstalled) which I didn't use but were still updating many files.

## Future Work

The general goal now is to do the file scan quicker, analyse it more efficiently and store it better. This boils ddown to the primary focus on the following:

- Integrate VirusTotal or other file hash checking service to be run on new files
- Ditch the `.csv` files and use a sql database instead, maybe sqlite3 for log access
- Improve os.walk speed by parallelising jobs with threads, splitting directory traversal, maybe see how file scanners or ransomware does this
- Incorporate visualisation of data somehow like heatmap of directories that change often or charting new files count over time.
