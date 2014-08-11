Svn2Git
=======

Migrate from SVN to Git with Fast Export for large/complex repositories.
Content below is copied and adapted from the content from KDE guide below and Jeff Geerling blogpost
https://techbase.kde.org/Projects/MoveToGit/UsingSvn2Git
http://www.midwesternmac.com/blogs/jeff-geerling/switching-svn-repository-svn2git

## Building Svn2Git
You need to have Qt4 and libsvn-dev installed on your machine to build Svn2Git.
Get and install them using first

	$ sudo yum install -y qt qt-devel
	$ sudo yum install -y subversion-devel

Now tool is ready to build. Switch to the code location and issue make. It will create the executable called `svn-all-fast-export`
	$ cd svn2git
	$ qmake && make
	
## How rulesets work
The format for the Svn2Git rules is pretty simple. First and foremost you have to declare some repositories:

	create repository kdelibs
	end repository

This tells Svn2Git that it should create a git repository called "kdelibs" that we can later on use to put commits into it.

The rest of the file are rules matching specific paths in Subversion, each rule specifies what to do with the commits that appeared at the given path. The possible actions are ignoring them or adding them to a particular branch in a particular repository. **Note:** Ignoring is done by simply leaving out the information about the repository and the branch.

As examples are more explanatory, the following rule would put all commits from 123453 to 456789 from the path /trunk/KDE/kdelibs into the master branch of the kdelibs repository:

	match /trunk/KDE/kdelibs/
	  min revision 123453
	  max revision 456789
	  repository kdelibs
	  branch master
	end match

The min and max revision are useful in cases where the same path in SVN contains code for different branches. An example would be KDevelop3, where KDevelop 3.3 was shipped with KDE 3.5 until 3.5.7, 3.5.8 contained KDevelop 3.4 and 3.5.9 contained KDevelop 3.5 and all of those kdevelop versions are now under /branches/KDE/3.5/kdevelop.

The two revision parameters are however not mandatory, if they're left out, then all commits that went to the given matching path in SVN are taken over into the specified branch.

To generate tags with git you use a special format for the branch parameter: refs/tags/<tagname>. So to put all commits from /tags/KDE/4.4.0/kdelibs into the v4.4.0 tag in the kdelibs git repository the rule would be like this:

	match /tags/KDE/4.4.0/kdelibs/
	  repository kdelibs
	  branch refs/tags/v4.4.0
	end match

For more examples see the Svn2Git/samples/ directory and the rules in the [kde-ruleset repository](https://projects.kde.org/projects/playground/sdk/kde-ruleset)

The recurse action is a hack to tell Svn2Git to recurse into a directory it has just copied or that existed because it is of interest. Example: if we are importing kdelibs, it exists in `path|trunk/KDE/kdelibs`. At branching, someone did: `svn cp $SVNROOT/trunk/KDE $SVNROOT/branches/KDE/4.4`

SVN recorded in that commit that branches/KDE/4.4 was the only path changed. 
That means the rule `branches/KDE/[^/]+/kdelibs/` will not match.

We need to tell the tool that something interesting happened inside and it should recurse. Then it will apply again all rules to the files that exist at that point, at which point the rules will match.

## Important Details

* All rules matching directories need to end with a '/', else the tool will crash at some point. This is a known bug. The only exception are the rules using the recurse-action.

* All rules matching files need to use the prefix rule.

* Matching rules can use Regular Expressions (according to the QRegExp syntax) in the match line and can use backreferences in the repository, prefix and branch parameters using \n (n=1,2,3,...) to reduce the amount of rules.

* The rules form an ordered list that the tool goes through while matching the changed paths for each commit. So if two rules match the same path and neither of the two has more matching criteria, then the rule that is written further up in the file wins. This is useful to exclude certain commits from the extraction process, if you look at the existing kde ruleset  you'll notice that at the top some revisions are ignored.

* Each rule file needs to handle all commits, ie. each file should end with a rule which matches everything and does nothing.

## Step-by-Step on writing rules for a module
Please adapt these instructions to suit your needs. Adapted texts from KDE are provided here as an example only

### Analyzing Subversion history to write rules

First of all you should check whether there are already rules for this module in the kde-ruleset repository. If there are rules already please go down to "Running Svn2Git"

If there are no rules yet, lets start with the master (aka trunk) branch. The easiest way to find out history with svn is executing:
`svn log -v --stop-on-copy file:///path/to/svn/trunk/KDE/module`

Please note that '/path/to/svn/' is the path to the 65GB you downloaded, and the 'trunk/KDE/module' is the module you want to write rules for, but those two put together is *not* a path that physically exists on your disk. svn log is smart enough to do what you want.

This will give you a history of the given module in trunk, it'll stop on the first commit that copied the code from somewhere else. The verbose output will allow you to see where this copy came from.

Now we have a starting point to write a rule, we want all commits from this path in our module repository in the master branch:

	match /trunk/KDE/module/
	  repository module
	  branch master
	end match

If the log stops at a commit that copied the module from somewhere, we need to follow this to also get the history imported from the "old" place the module resided. The same svn command can be used with slightly different path argument:

	svn log -v --stop-on-copy file:///path/to/svn/some/other/path@revision

The @revision is important as the original path usually doesn't exist anymore. With this we can write the next rule to the rules file and repeat until we've finally reached the point where the code was initially imported into svn (or probably cvs in the old days)

Now we can take care of the branches, this is a bit more involved as there may be multiple branches scattered over the /branches directory in svn. You can use the same commands as before to find out the history of a branch if you know the path. This time however you can stop following the source of copy-operations once you've found a source that you've already matched in a rule. That way your branch will be connected to the branch it originated from (which is often trunk aka master) in git.

A useful help with finding branches is svn ls in combination with the path@revision syntax, that way you can view the content of a particular svn directory as it was in an older revision. With this you can even find branches that are not visible (have been deleted) in the current revision.

The rule for putting commits into a git branch in the final repository is only slightly different (the example is for a core module):

	match /branches/KDE/4.4/module
	  repository module
	  branch 4.4
	end match

And last are the tags, this works the same as branches and trunk, except for using branch refs/tags/v<tag-version> for the branch parameter.

## Running Svn2Git
This is the easiest, but most time-consuming part. As example lets say that in our current working directory we have the kde rules repository in kde-ruleset subdir, the Svn2Git tool in the Svn2Git subdir and the KDE repository in the svn subdir:

	svn2git/svn-all-fast-export --identity-map kde-ruleset/account-map --rules kde-ruleset/module svn

Here is the syntax
	$ /path/to/svn-all-fast-export --identity-map=path/to/account-map-file --rules=path/to/rule-definition-file --stats --add-metadata path/to/svn-repository

Once its done you should have a new "module" git repository in your current working directory. 

## Post-processing
Some problems that might occur in the converted git repository can be fixed after the actual conversion. More specifically, these are wrong parents and empty commit objects for tags.

Wrong parents can be handled using a parent map. This is a file with each line in the format "parent-id child-id", where both ids are the integer numbers of the corresponding svn commit. You can also add newlines and comments starting with #. Each line in the parent map adds an edge in the commit graph from parent to child. This is frequently necessary to correctly reflect merges in the git history. Occasionally, Svn2Git might add a parent which should not be a parent at all. In this case, you can write a line "clear child-id" to delete that edge, and afterwards add the correct edge as explained above. It is usually easiest to first do a conversion, then look at the result with "gitk --all" and if something is wrong  find the svn ids from the git commit message.

To actually perform the fix, run "kde-ruleset/bin/parent-adder $parent_map_file" in the new git repository. You should probably make a backup of the git repository before, as rerunning the conversion would take a lot of time if something goes wrong.

Empty commit objects for tags can be fixed by running "kde-ruleset/bin/fix-tags" in the git repository.

Both processes leave a backup of the original commit objects. These can be removed with "kde-ruleset/bin/remove-fb-backup-refs.sh".

## Checking for proper history in the new git repository
To check the history of the created git repository simply change into the directory and use your favourite git command like ''git log'' to check if all is well.

A **very easy way to check whether the history was imported properly** is to use the gitk tool from git. It shows you a graphical representation of the history in the git repository which makes it easy to identify where something is wrong.

The tool should be run with the --all switch so it shows all branches.

You can now scroll through the history to check whether things have been imported correctly.

First and foremost there should be the master branch starting at the top with the most recent commit to trunk/ and ending in the oldest commit that imported the code into KDE's svn or cvs repository. 

From the master branch there should be several branches going away for each branch you imported. And eventually also branches that start from another non-master branch.

Things that you should look out for are branches that start "nowhere", that is the first commit in the branch has no parent in another branch or master. This means that Svn2Git didn't see a commit that created this branch from another using a svn cp command. That can mean that you may have forgotton to add a match rule for some path or that the same path was used for different branches in different revisions. The same applies to tags which have a commit without any parent.

This can usually be fixed by using svn log and svn ls to follow the history of the branch. Eventually you might need to apply the min/max revision paramters.

You'll notice that some tags are looking like this:

<pre>
|
* * <v1.2.3>
| |
* *
| /
*
</pre>

Thats normal for our tags even if a bit ugly. The reason is that often compile-fixes are done in trunk/ after the tag has been created and then the commit has been merged over to the tag.

Another thing however are tags that are named vx.y.z_124321. These are tags that have been deleted and re-created later. You can usually see that in the svn log history, these tags can either be manually deleted after the repository creation using git tag or you can add rules that ignore certain revisions of the tag-path before the one putting the commits into the git repository:

<pre>
match /tags/KDE/3.3.2/kdelibs
  min revision 424234
  max revision 424236
end match
match /tags/KDE/3.3.2/kdelibs
  repository kdelibs
  branch refs/tags/3.3.2
end match
</pre>

If you choose to delete them manually please make sure to document this with a textfile or inside the rule file so if someone else does the conversion later again he'll know what manual steps you did.

Also try grepping the output from Svn2Git for the string '"copy from"' (with the double quotation marks). This will give you a list of revisions/paths that Svn2Git could not detect the origin of. That is someone did a svn cp/mv and the old path is not in the generated git repository.

Before publishing the newly created git repository make sure to repack it. This can greatly reduce it's size (i.e. Phonon's git repository could be shrunken from 18 MB to 5.2 MB)

## Troubleshooting

### Recurse action doesn't work with cvs2svn tag commits

You may have to deal with a commit done by cvs2svn to create a tag, for example:
<pre>
r386536 | (no author) | 2005-02-05 22:16:00 +0100 (Sat, 05 Feb 2005) | 2 lines
Changed paths:
   A /branches/beta_0_7_branch (from /trunk:386535)
   D /branches/beta_0_7_branch/art-devel
   D /branches/beta_0_7_branch/arts
   D /branches/beta_0_7_branch/bugs
   D /branches/beta_0_7_branch/devel-home
   D /branches/beta_0_7_branch/developer.kde.org
   D /branches/beta_0_7_branch/enterprise.kde.org
   D /branches/beta_0_7_branch/events.kde.org
   D /branches/beta_0_7_branch/foundation
   D /branches/beta_0_7_branch/kckde
   D /branches/beta_0_7_branch/kde-common
   D /branches/beta_0_7_branch/kde-i18n
   D /branches/beta_0_7_branch/kde-qt-addon
   D /branches/beta_0_7_branch/kde-women.kde.org
   D /branches/beta_0_7_branch/kdeaccessibility
   D /branches/beta_0_7_branch/kdeaddons
   D /branches/beta_0_7_branch/kdeadmin
   D /branches/beta_0_7_branch/kdeartwork
   D /branches/beta_0_7_branch/kdebase
   D /branches/beta_0_7_branch/kdebindings
   D /branches/beta_0_7_branch/kdeedu
   D /branches/beta_0_7_branch/kdeextragear-1
   D /branches/beta_0_7_branch/kdeextragear-2
   M /branches/beta_0_7_branch/kdeextragear-3
   D /branches/beta_0_7_branch/kdeextragear-3/Makefile.am.in
   D /branches/beta_0_7_branch/kdeextragear-3/Makefile.cvs
   D /branches/beta_0_7_branch/kdeextragear-3/README
   D /branches/beta_0_7_branch/kdeextragear-3/configure.in.bot
   D /branches/beta_0_7_branch/kdeextragear-3/configure.in.in
   D /branches/beta_0_7_branch/kdeextragear-3/digikam
   D /branches/beta_0_7_branch/kdeextragear-3/digikamimageplugins
   D /branches/beta_0_7_branch/kdeextragear-3/doc
   D /branches/beta_0_7_branch/kdeextragear-3/filelight
   D /branches/beta_0_7_branch/kdeextragear-3/kcfgcreator
   D /branches/beta_0_7_branch/kdeextragear-3/kconfigeditor
   D /branches/beta_0_7_branch/kdeextragear-3/kdebluetooth
   D /branches/beta_0_7_branch/kdeextragear-3/kdetv
   D /branches/beta_0_7_branch/kdeextragear-3/keurocalc
   D /branches/beta_0_7_branch/kdeextragear-3/kiosktool
   D /branches/beta_0_7_branch/kdeextragear-3/klicker
   D /branches/beta_0_7_branch/kdeextragear-3/kplayer
   D /branches/beta_0_7_branch/kdeextragear-3/pwmanager
   D /branches/beta_0_7_branch/kdeextragear-libs-1
   D /branches/beta_0_7_branch/kdegames
   D /branches/beta_0_7_branch/kdegraphics
   D /branches/beta_0_7_branch/kdeinstaller
   D /branches/beta_0_7_branch/kdejava
   D /branches/beta_0_7_branch/kdekiosk
   D /branches/beta_0_7_branch/kdelibs
   D /branches/beta_0_7_branch/kdemultimedia
   D /branches/beta_0_7_branch/kdenetwork
   D /branches/beta_0_7_branch/kdenonbeta
   D /branches/beta_0_7_branch/kdenox
   D /branches/beta_0_7_branch/kdepim
   D /branches/beta_0_7_branch/kdeplayground-artwork
   D /branches/beta_0_7_branch/kdeplayground-base
   D /branches/beta_0_7_branch/kdeplayground-edu
   D /branches/beta_0_7_branch/kdeplayground-games
   D /branches/beta_0_7_branch/kdeplayground-ioslaves
   D /branches/beta_0_7_branch/kdeplayground-multimedia
   D /branches/beta_0_7_branch/kdeplayground-network
   D /branches/beta_0_7_branch/kdeplayground-pim
   D /branches/beta_0_7_branch/kdeplayground-utils
   D /branches/beta_0_7_branch/kdereview
   D /branches/beta_0_7_branch/kdesdk
   D /branches/beta_0_7_branch/kdesecurity
   D /branches/beta_0_7_branch/kdesupport
   D /branches/beta_0_7_branch/kdetoys
   D /branches/beta_0_7_branch/kdeutils
   D /branches/beta_0_7_branch/kdevelop
   D /branches/beta_0_7_branch/kdewebdev
   D /branches/beta_0_7_branch/kdoc
   D /branches/beta_0_7_branch/kfte
   D /branches/beta_0_7_branch/khtmltests
   D /branches/beta_0_7_branch/klyx
   D /branches/beta_0_7_branch/kmusic
   D /branches/beta_0_7_branch/koffice
   D /branches/beta_0_7_branch/kofficetests
   D /branches/beta_0_7_branch/konstruct
   D /branches/beta_0_7_branch/qt-copy
   D /branches/beta_0_7_branch/quanta
   D /branches/beta_0_7_branch/sysconfig
   D /branches/beta_0_7_branch/valgrind
   D /branches/beta_0_7_branch/www

This commit was manufactured by cvs2svn to create branch
'beta_0_7_branch'.
</pre>
If you do this
<pre>
match /branches/beta_0_7_branch/kdeextragear-3/krecipes/
  repository krecipes
  branch 0.7
end match

match /branches/beta_0_7_branch/
  min revision 386536
  max revision 386536
  action recurse
end match
</pre>
svn-all-fast-export will fail, you'll get an error sayining that '/foo/bar/path' was not found where '/foo/bar/path' is one of the deleted paths in the cvs2svn commit. This is because some paths were deleted in the same commit where you want to do an 'action recurse'. Therefore, to avoid matching the deleted paths you should do an action recurse on each intermediate directory from '/branches/beta_0_7_branch/' to '/branches/beta_0_7_branch/kdeextragear-3/krecipes/' and you should use a final '$' to make sure that the deleted paths will not be considered, thusly:
<pre>
match /branches/beta_0_7_branch/kdeextragear-3/krecipes/
  repository krecipes
  branch 0.7
end match

match /branches/beta_0_7_branch/kdeextragear-3/$
  min revision 386536
  max revision 386536
  action recurse
end match

match /branches/beta_0_7_branch/$
  min revision 386536
  max revision 386536
  action recurse
end match
</pre>


