# An Efficient Workflow for Managing BibTeX Entries  

In this article, I discuss the issues I encountered when handling with BibTeX entries and the solution I came up with.
I assume that readers have some experience with LaTeX, BibTeX, JabRef, Git and Bash.

### Background
Every time I download a journal paper, I like to use the BibTeX entry provided by the website to add a new entry into my BibTeX database and link the pdf file using JabRef.
In this way, I build a catalogue of all the papers that I have ever downloaded on my computer over time.
This can be handy for finding a paper that I know I have seen before.
When I need to cite a paper, I can also easily generate a reference list using the database.
However, as my database grows, I realize that to make good use of my database in a research project, there are some issues that I need to tackle.

### The Issues
For both version control purpose and portability, it is good to have a copy of my BibTeX database in the project folder just for that specific piece of work on hand. However, problems arise when I have several BibTeX databases on my computer.

1. When I need to add new BibTeX entries for the project, should I directly put them into the database for the project or put them into "the catalogue" first where I hold everything in one place?

2. It often happens that I discover some errors (eg., "-" vs. "--") or outdated information (eg., "year" for a working paper) in the BibTeX entries as I work on the project. In the heat of the moment, it is easier to directly fix them in the database for the project. But, how can I ensure that I also fix the errors in my original database to avoid repeating the same errors in the future?

3. As my main database, "the catalogue", grows over time, it may well contain over a thousand entries in the future. However, I will only need a small subset of the entries when working on a project. Carrying around all the entries will distract me from the most relevant ones.

### My Solution
The main idea is that rather than copying my main database into a project folder, I maintain a central database in one place and selectively generate a subdatabase programmatically based on a project-specific criterion. In other words, I don't directly handle the .bib file for the subdatabase. Instead, I write down the procedure for making the subdatabase and let the computer program generate the subdatabase from the main one following the procedure. Thus, all new entries and error-fixing are first applied to the main database and project-specific subdatabases are generated whenever I need them.

For my solution to work, in addition to JabRef, I need a command line tool called `bib2bib`, which is included in the [BibTeX2HTML](https://www.lri.fr/~filliatr/bibtex2html/) package. I have written Bash scripts to implement the procedures. Although the procedure below is written for a MacOS environment, only minor modifications are required for Linux and Windows platforms.

Here are the details:
1. I maintain a main database/catalogue as before. In addition, for entries relevant to a project on hand, I add them into a group using the grouping function of JabRef. For demonstration purpose, let's call this group DEMO.

2. For convenience, I define an environmental variable `$BIB_BASE` which is the path of the folder containing the main BibTeX database.

3. In the project folder, I have a "bib" subfolder where I put my .bib file to generate the reference list for the LaTeX document. In this "bib" folder, I put a shell script named "gen-bib". An example is shown below:
```bash
#!/bin/bash

# Run this script to generate a BibTeX sub-database from the main catalogue using bib2bib based on filter conditions
# To make this file executable, run "chmod +x bib/gen-bib" from project repo

# $BIB_BASE is an environmental variable added to ~/.bash_profile
CATALOGUE="$BIB_BASE/catalogue.bib"

# Verify CATALOGUE exists and has a length greater than zero 
if  [ ! -s "$CATALOGUE" ]; then
    echo "ERROR: CATALOGUE is not found!"
    exit 1
fi

# "$(dirname "$0")" ensures ref.bib and gen-bib are in the same directory
# --no-comment avoids printing out CATALOGUE path in output file
bib2bib --no-comment -ob "$(dirname "$0")"/ref.bib -c 'groups : "DEMO"' "$CATALOGUE"
```

Note that JabRef saves the grouping information by creating a field called "groups" in relevant entries. So, by setting the filter condition to be `-c 'groups : "DEMO"'`, only the entries included in the DEMO group are included in the new subdatabase. For usage of `bib2bib`, see the documentation [here](https://www.lri.fr/~filliatr/bibtex2html/doc/manual.html#sec13).

4. Since I may need to update the subdatabase whenever I modify the main database, the `gen-bib` script will be run frequently. To make things easier, I put the script below into `/usr/local/bin` (by creating a symbolic link) to create a Bash command `gnb` (which is the name of the script file). Now, whenever I need to update the subdatabase, I only need to navigate to the project folder and type the command `gnb`.
```bash
#!/bin/bash

# This script run the "gen-bib" file located either in the working directory or in the "bib" subdirectory
# Put this script in /usr/local/bin so that it can be called from anywhere
# Alternatively, set a symbolic link by running
# ln -sf <SOURCE_PATH> <TARGET_PATH>

# Test the existence of "gen-bib" and then run the script
if  [ -s bib/gen-bib ]; then
    ./bib/gen-bib
    exit 0
elif [ -s gen-bib ]; then
    ./gen-bib
    exit 0
else
    echo "ERROR: gen-bib is not found!" >&2
    exit 1
fi
```

5. The previous steps are sufficient for most of the scenarios. In case that one needs to modify entries in a special way for a particular project (which rarely happens I believe), one can do that by implementing the modifications through a stream editor called `sed`.

6. Regarding to the potential of varying BibTeX citation keys over time, I believe that this is not an issue. Since I always use JabRef to automatically assign keys, the only scenario that can cause keys to change over time is that I add new entries for the same author-year combination, which rarely happens.

### Advantages over Existing Solutions

I believe that many people have encountered similar problems as I have. For example, see the discussions [here](https://tex.stackexchange.com/questions/297075/latex-git-bibliography) and [here](http://andrius.velykis.lt/2012/06/master-bibtex-file-git-submodules/). Comparing with the method of using Git Submodules or texmf tree, my solution has the following advantages:

1. **Simple to follow**. The only rule to be remembered is that all new entries and modifications go into the main database. Thanks to JabRef, grouping entries for each project is convenient. (Note that you can let JabRef only display entries in a group or not in a group, which I find very helpful.) After grouping all the relevant entries together, a project-specific subdatabase is prepared in the blink of an eye by typing the `gnb` shell command.  There is no hassle to do git push and git pull to synchronize between two databases. Since the synchronization process does not rely on Git at all, I can update the subdatabase immediately even without doing git commit to the main database.

2. **Flexibly synchronized**. Even if there are multiple projects with multiple versions of subdatabases, each BibTeX entry for the same piece of citation will be exactly the same because they are all generated from the same main database. Need project-specific modifications? No problem! In the `gen-bib` script file for that project, use `sed` command to replace the piece of entry that you want to modify just for the project. The `gen-bib` file is the place to hold the recipe for each subdatabase.

3. **Keep things relevant**. Thanks to the `bib2bib` command, it is easy to maintain only a subset of the growing database in my project folder while still ensure that all the entries are up-to-date. As a result, there is no distraction from irrelevant entries when working on a project. This is especially helpful when working with coauthors in a shared project folder. Though this feature seems to be basic, it would be complicated to implement if one works with Git Submodules.
