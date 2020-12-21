1. git clone
2. git checkout -b  //创建分支
3. git branch -a //查看分支
4. git commit  or git commit --amend
5. git add -A  
6. git review -R
7. git fetch
8. how to commit --amend with the same changed id
git review -d  id
git commit --amend 
git review -R
git reset --hard  
9. git cherry pick
cherry pick
  504  cd /opt/stack/horizon/
  505  git branch
  506  git pull
  509  git checkout master
  512  git checkout -B liberty/horizon origin/stable/liberty      (git checkout -b 是基于本地分支创建branch)
  513  git cherry-pick -x d1c3b4787b792fe6e20a3fd6e692015fd576a5f1  (后面加上commit ID not change ID)
  514  git stauts
  515  git status
  516  vim openstack_dashboard/api/heat.py
  517  vim openstack_dashboard/dashboards/project/stacks/forms.py
  518  git add -A
  519  git stauts
  520  git status
  521  git cherry-pick --continue
  522  git config --global user.email dixiaobj@cn.ibm.com
  523  git status
  524  git cherry-pick --continue
  525  git status
  527  git review stable/liberty -n   (-n 是先在本地实验是否能够git review成功.)
  528  git review stable/liberty
  529  git commit --amend
  530  git review stable/liberty  
10. git review osee-icehoust   //  当向远程分支提交code时,需要加上远程分支的名字
     git remote add gerrit  url   //  在本地添加名字为gerrit的远程branch
     

## git review
yum install git and git-review
git clone git://github.com/facebook/git-review.git
cd git-review
python setup.py install

                                                        # git config --global user.name dixiaobj
[root@controller1-4a2755a3 jackalope]# git config --global user.email dixiaobj@cn.ibm.com
[root@controller1-4a2755a3 jackalope]# git config --list
 ssh-keygen -t rsa -C "dixiaobj@cn.ibm.com"
copy the id_rsa.pub to http://gerrit.rtp.raleigh.ibm.com/#/settings/ssh-keys


[root@controller1-4a2755a3 jackalope]# git review -s
Could not connect to gerrit.
Enter your gerrit username: dixiaobj
Trying again with ssh://dixiaobj@gerrit.rtp.raleigh.ibm.com:29418/es-infra/jackalope.git
Creating a git remote called "gerrit" that maps to:
        ssh://dixiaobj@gerrit.rtp.raleigh.ibm.com:29418/es-infra/jackalope.git
This repository is now set up for use with git-review. You can set the
default username for future repositories with:
  git config --global --add gitreview.username "dixiaobj"
[root@controller1-4a2755a3 jackalope]# git config --global --add gitreview.username "dixiaobj"
git branch -a
git checkout -b fix-hyperv-network
change code 
git status 
git add -A
 
git commit / git commit --amend
git show   
git review -R 

[root@controller1-4a2755a3 jackalope]# git review -R
remote: Resolving deltas: 100% (6/6)
remote: Processing changes: new: 1, done
remote:
remote: New Changes:
remote:   http://gerrit.rtp.raleigh.ibm.com/8878
remote:
To ssh://dixiaobj@gerrit.rtp.raleigh.ibm.com:29418/es-infra/jackalope.git
* [new branch]      HEAD -> refs/publish/master/fix-hyperv-network
git commit --amend

http://www.cnblogs.com/linjiqin/p/3772681.html  learn 
http://blog.csdn.net/shatty/article/details/10200207


git log
1078  git format-patch 764b228e1ba626e607a84e1492c7a1d21c334209
1079  mv 0001-fix-the-neutron-part-on-hyperv.patch ~
1080  git reset 764b228e1ba626e607a84e1492c7a1d21c334209

## squash-your-latests-commits-into-one
https://w3-connections.ibm.com/blogs/drf/entry/Cherry_picking_two_patches_together?lang=en
https://ariejan.net/2011/07/05/git-squash-your-latests-commits-into-one/
Cherry-picking two patches together
Douglas R. Fish | Today 12:23 AM ‎ | 6 Views
 
Like
Sometimes it happens that we need to cherry-pick 2 or more fixes to an old release. We want to do this is a reliable way that preserves the references to the original commits. The git cherry-pick -x and rebase -i commands can be used to accomplish this.
Here's a recent example of a fix that needs this kind of treatment:
http://gerrit.rtp.raleigh.ibm.com/#/c/10135/
Note that the fix requires both updated code from https://review.openstack.org/#/c/241700/ as well as updated unit tests from https://review.openstack.org/#/c/287543/ Neither patch will work successfully in stable/liberty without the other; they need to go in together.
First we should get the code fix. In https://review.openstack.org/#/c/241700/ Let's look for a stable/liberty backport by clicking Topic bug/1512564
it exists, so click the abandoned patch at https://review.openstack.org/#/c/260436/
click download/cherry-pick to get most of the command that we will need for the cherry-pick. The "-x" option will need to be added manually to get this command:
git fetch https://review.openstack.org/openstack/horizon refs/changes/36/260436/2 && git cherry-pick -x FETCH_HEAD
Then go through the same process for https://review.openstack.org/#/c/287543/
There is no stable backport to cherry-pick from (it's not suitable to create one because the tests don't fail there), so the download/cherry-pick command becomes
git fetch https://review.openstack.org/openstack/horizon refs/changes/43/287543/1 && git cherry-pick -x FETCH_HEAD
 
At this point there are two changes that need to go out. Unit tests will pass, but there isn't really a way to get this into gerrit because the first patch will fail. These two patches can be squashed together and treated as a single patch. Right now the history looks like:
 
ubuntu@devstack-drf:/opt/stack/horizon$ git log -3
commit c08ee272a4dd52573bbeda341427f4e0b5704858
Author: David Lyle <david.lyle@intel.com>
Date:   Wed Mar 2 18:11:09 2016 -0700
    Fixing heatclient release compat issues
    
    The new release of python-heatclient broke tests that were using
    hardcoded sample templates. For the json based templates, the new
    line was escaped. For yaml templates, a comment section was added to the
    templates.
    
    My working theory is that the exception handling changed and we were
    missing an error before that is being exposed now. That is to say, our
    hardcoded templates were wrong before, we just weren't seeing it.
    
    Tested with latest version of python-heatclient 1.0.0 and last version
    0.9.0.
    
    Closes-Bug: #1552400
    Change-Id: Ibdf985654ebfa60205068b167a37600f7ed4c1f4
    (cherry picked from commit dcc838128e0812a760733f24a65f09bcd047a399)
commit 09c4cdb73cd6b7099fc1dc17220d37ea7ca9bd86
Author: dixiaoli <dixiaobj@cn.ibm.com>
Date:   Wed Nov 4 17:32:29 2015 +0000
    Add handle get_file when launch stack from horizon
    
    when get_file is contained in template, the stack create/update/preview
    will fail due to No content found in the "files" section.
    So added handle get_file code.
    
    Change-Id: I6f125f9e5f3f53f630ab0d4f3f00631e6850e905
    Closes-Bug: #1512564
    (cherry picked from commit d1c3b4787b792fe6e20a3fd6e692015fd576a5f1)
    (cherry picked from commit 687dcb04597273950d21d44a7fa94be063f91299)
commit 8b7a312e89a85ae52bccd0e79e17a3709bc0ca68
Merge: 1720d4f 53c618c
Author: Doug Fish <drfish@us.ibm.com>
Date:   Tue Oct 20 13:31:13 2015 +0000
    Merge branch 'stable/liberty' into osee-liberty
ubuntu@devstack-drf:/opt/stack/horizon$
 
Those first two patches need to become one. This is accomplished by using git rebase -i then squashing the last commit into the previous one. Let's work with just the last 3 commits
git rebase -i HEAD~3
 
Use your favorite editor to change the patches to be
pick 53c618c Switch to post-versioning
pick 09c4cdb Add handle get_file when launch stack from horizon
squash c08ee27 Fixing heatclient release compat issues
 
Save and Exit. You'll get a chance to edit the final commit message. Be sure to add the line to the related RTC change! Save and exit again. Run Unit Tests and double check the commit message. It should look similar to
ubuntu@devstack-drf:/opt/stack/horizon$ git log -1
commit 08a72ba258d5f8c1c2efb8f12c44b5956a477e33
Author: dixiaoli <dixiaobj@cn.ibm.com>
Date:   Wed Nov 4 17:32:29 2015 +0000
    Add handle get_file when launch stack from horizon
    
    when get_file is contained in template, the stack create/update/preview
    will fail due to No content found in the "files" section.
    So added handle get_file code.
    
    Change-Id: I6f125f9e5f3f53f630ab0d4f3f00631e6850e905
    Closes-Bug: #1512564
    (cherry picked from commit d1c3b4787b792fe6e20a3fd6e692015fd576a5f1)
    (cherry picked from commit 687dcb04597273950d21d44a7fa94be063f91299)
    
    Fixing heatclient release compat issues
    
    The new release of python-heatclient broke tests that were using
    hardcoded sample templates. For the json based templates, the new
    line was escaped. For yaml templates, a comment section was added to the
    templates.
    
    My working theory is that the exception handling changed and we were
    missing an error before that is being exposed now. That is to say, our
    hardcoded templates were wrong before, we just weren't seeing it.
    
    Tested with latest version of python-heatclient 1.0.0 and last version
    0.9.0.
    
    RTC: The proper RTC number goes here
    ibm-only
    
    Closes-Bug: #1552400
    Change-Id: Ibdf985654ebfa60205068b167a37600f7ed4c1f4
    (cherry picked from commit dcc838128e0812a760733f24a65f09bcd047a399)
ubuntu@devstack-drf:/opt/stack/horizon$
