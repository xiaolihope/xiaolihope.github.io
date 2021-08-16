## git 常用命令
```
1. git clone
2. git checkout -b  //创建分支
3. git branch -a //查看分支
4. git commit  or git commit --amend
5. git add -A  
6. git review -R
7. git fetch
```
8. how to commit --amend with the same changed id
```
git review -d  id
git commit --amend 
git review -R
git reset --hard
```  
9. git cherry pick
```
cd /opt/stack/horizon/
git branch
git pull
git checkout master
git checkout -B liberty/horizon origin/stable/liberty      (git checkout -b 是基于本地分支创建branch)
git cherry-pick -x d1c3b4787b792fe6e20a3fd6e692015fd576a5f1  (后面加上commit ID not change ID)
git stauts
vim openstack_dashboard/api/heat.py
vim openstack_dashboard/dashboards/project/stacks/forms.py
git add -A
git stauts
git cherry-pick --continue
git config --global user.email dixiaobj@cn.ibm.com
git status
git cherry-pick --continue
git status
git review stable/liberty -n   (-n 是先在本地实验是否能够git review成功.)
git review stable/liberty
git commit --amend
git review stable/liberty  
git review osee-icehoust   //  当向远程分支提交code时,需要加上远程分支的名字
git remote add gerrit  url   //  在本地添加名字为gerrit的远程branch
```
10. git review
```
yum install git and git-review
git clone git://github.com/facebook/git-review.git
cd git-review
python setup.py install
git config --global user.name dixiaobj
git config --global user.email dixiaobj@cn.ibm.com
git config --list
ssh-keygen -t rsa -C "dixiaobj@cn.ibm.com"
copy the id_rsa.pub to http://gerrit.rtp.raleigh.ibm.com/#/settings/ssh-keys
git review -s
Could not connect to gerrit.
Enter your gerrit username: dixiaobj
Trying again with ssh://dixiaobj@gerrit.rtp.raleigh.ibm.com:29418/es-infra/jackalope.git
Creating a git remote called "gerrit" that maps to:
        ssh://dixiaobj@gerrit.rtp.raleigh.ibm.com:29418/es-infra/jackalope.git
This repository is now set up for use with git-review. You can set the
default username for future repositories with:
  git config --global --add gitreview.username "dixiaobj"
# git config --global --add gitreview.username "dixiaobj"
# git branch -a
# git checkout -b fix-hyperv-network
change code 
# git status 
# git add -A
# git commit / git commit --amend
# git show   
# git review -R 
remote: Resolving deltas: 100% (6/6)
remote: Processing changes: new: 1, done
remote:
remote: New Changes:
remote:   http://gerrit.rtp.raleigh.ibm.com/8878
remote:
To ssh://dixiaobj@gerrit.rtp.raleigh.ibm.com:29418/es-infra/jackalope.git
* [new branch]      HEAD -> refs/publish/master/fix-hyperv-network
# git commit --amend
# git log
# git format-patch 764b228e1ba626e607a84e1492c7a1d21c334209
# mv 0001-fix-the-neutron-part-on-hyperv.patch ~
# git reset 764b228e1ba626e607a84e1492c7a1d21c334209
```
11. squash-your-latests-commits-into-one

12. windows上git bash中文显示乱码

右键->options->Text->Locale->UTF-8->Apply
then
`git config --global core.quotepath false`

13. windows上用git bash提交代码到git repositories

git push时，填入personal access tokens即可。

github -> settings -> Developer settings -> Personal access tokens, choose Generate new token.

