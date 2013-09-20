!!DRAFT!!

How we merged an svn repo and a git fork of it back into one git repository
===========================================================================

Some years ago on of our customer had forked a software project, every now and changes they made to this fork were contributed back to the original project.
For several reasons it was decided that the original version and our clients fork should be merged and I was given the task to this.
There were however several issues:
* The fork and the original project had diverged quite a bit
* The fork was on github while the original project was in svn on googlecode
* The fork was in a subdirectory of a 'mother' project
* Applying patches to both code bases was a painfull process

Converting the old svn repot to git
-----------------------------------

It was decided that the merged version should be on GitHub, so the first thing to do was converting the svn repo from google code
to a git repo on github. I will not go to deep into this as this is already described quite well by other people. The simplest way to do this is to use svn2git, just created a dir and run the 
svn2git command: (see: https://github.com/nirvdrum/svn2git)

<code>
mkdir svn-to-git-converted-repo
cd svn-to-git-converted-repo
svn2git --verbose http://original-project.googlecode.com/svn
</code>

Bringing the fork up to date
----------------------------
Before starting the whole merge process it is important to make the fork up to date with the original so it contains all of the functionality of the original which makes merging it back later on a little easier.

zExtracting the fork from the git 'mother' repo
----------------------------------------------
Since the fork was created in a subdir, this part of the clients repo needed to be extracted.
Luckily this can be achieved by using git filter-branch, note that this command acts directly on the repo so it's cloned first. The following command will move all code in subdir/fork to the root of the project. ---prune-empty ensures that empty revisons (revisions that changed something outside this dir) are removed
<code>
git clone mother-repo fork-export
cd $fork export
git filter-branch -f --subdirectory-filter subdir/fork --prune-empty -- --all
</code>

Removing the svn id's from the code
-----------------------------------
Svn has the concept of magic keywords for example $Id$ which are expanded by svn upon file creation. Since the project used these and the fork was created by adding the result of an svn export to the clients git repo these $Id$ strings in expanded format were all over the place. To make the diff (and thus potential merge conflicts) between the svn2git imported original and the fork smaller these had to be removed.
Once again it's git filter-branch to the rescue, it can perform an operation on every file in every revision. This process can be time consuming so two things were done to speed things up:
* Only process php file (we knew no other files contained the keywords)
* Grep only the file that contain an $Id$

The list of php files containing $Id$ tags is fed to sed (using xargs) does an inline replacement to normalize the keywords to their non-expanded form. the 'or true' statement is needed to keep the tree filter going

<code>
# Remove all Svn id's (caused by using an svn export) to reduce the number of differences between OpenConext and OFFICIAL versions
echo "Remove svn id's"
git filter-branch -f --tree-filter "grep -rl '\$Id' --include=*.php | xargs sed -i s/\\\$Id[^\$]*\\\\$/\\\$Id\\\$/g > /dev/null 2>&1 || true" -- --all
<code>

Copy the forked version to the converted repo
---------------------------------------------
Now the forked code is cleaned from svn id's it's time to reintegrate the forked version into the original.
This is done by creating an so called 'orphan' branch in the repo, and pulling the forked code into this branch. 
Note that 'orphan' means it is not based on any other branch in the repo. 

<code>
git checkout --orphan forked-version
git remote add -f fork-origin fork-export
git pull fork-origin forked-version
</code>

Add missing history to the fork
-------------------------------
In the current state the first commit of this orphan branch is a huge commit containing the state of the files on import while in real life this code was created in lots of small commits.
So what needs to be done is fix the history of this branch. This can be done by replacing the parent commit by the history in the original repo that lead to the imported version.
To be able to this you need to know to which commit in the history the original import maps. 
Luckily in this case the person who did the initial import of the forked code mentioned the svn revision the import came from.
Using the commit message of this particular revision it was easy to find the matching git commit in the git-to-svn covnerted version.
When the commit has been found the parent commit of the orphan branch needs to be replaced by this. This can be done using rebasing.


@todo explanin commit order and ~1

<code>
git rebase --onto d763def11c9275f86070da68e805e408dc8e637d 27a67401ecb9149ce9ede127bb5612e205fccd50~1
</code>


Merge fork and original
-----------------------
Now er have two branches one original and fork wich originates from some commit.
While it is possible to simply merge the fork back into the original this would overwrite too much of the history since the original was also merged into the fork at times.
The steps we chose were 
- First looking at the diff between original and fork to determine which features the original has and pick a feature to merge back
- Create a feature branch for a given feature
- Cherry pick all related commits from the fork using the log
- Merge the feature back into the original
- Repeat until there is no diff between both branches

Needless to say some manual fixes and conflict resolving were required 

