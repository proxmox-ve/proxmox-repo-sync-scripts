# How the repository sync will be structured



## preface

- this consists only of some quick hacky bash, written in one night
- each night the upstream code will be compared
- only snapshots of HEAD will be imported, not every single commit
- the following steps currently should work (05/10/17) for comparison, without having to pull 1G+ each night



## preparements

1. check free disk space, should be minimum 3G at least, since the base checkout will be about almost 1,5G already

2. get a list of all repositories

`curl -s https://git.proxmox.com/?a=project_index | awk {'print $1'} | sort`

3. create all repos via the github api

`curl -u <user> https://api.github.com/user/repos -d '{ "name": "<reponame>", "description": "<description>" }`

This is done locally, so passwords won't end up on the server.

All the following will be done on the server which will have the cronjobs to run this every night.

4. get list of all filenames to compare in the future, see below

5. clone all repositories, where the snapshots will be extracted into, so changes can be committed later

Make sure these are properly named. (With just the repo name.)

6. initial download of all projects' snapshots

7. extract files into the cloned projects' folder

8. do mass add / commit for all those

9. initial pushes



## cronjobs every night

1. rename 'CURRENT' to `date --date='yesterday' +%F`

1. create all repos from all filenames (so they can actually be compared with the projects upstream)

2. get list of all snapshot links, write to file 

`grep snapshot <(while read LINE; do curl -s https://git.proxmox.com/?p=$LINE\;a=tree; done < <(curl -s https://git.proxmox.com/?a=project_index | awk {'print $1'} | sort) ) | sed -e 's,.*"/?p=,?p=,' -e 's/".*//'`

Output: 

    ?p=aab.git;a=snapshot;h=HEAD;sf=tgz
    ?p=apt.git;a=snapshot;h=HEAD;sf=tgz
    ?p=arch-pacman.git;a=snapshot;h=HEAD;sf=tgz
    ?p=ceph.git;a=snapshot;h=HEAD;sf=tgz
    ...

3. get all current filenames from the Content-Disposition HTTP header, write list to file 'CURRENT'

`curl -vsI 'https://git.proxmox.com/?p=aab.git;a=snapshot;h=HEAD;sf=tgz' |& grep ^Content-disposition | sed -e 's/.*filename="//' -e 's/"//'`

Output:

    aab-HEAD-0cff4ef.tar.gz

4. diff files named `date --date='yesterday' +%F` and zzz_01_CURRENT, create show changes in file 'zzz_02_DIFF'

5. For all the lines in diff, get the real reponame, save to file 'zzz_03_TO_DL'

6. loop over contents in zzz_03_TO_DL and download snapshots with changes

7. extract everything into local repository

8. get all reponames with changes, save to file 'zzz_04_CHANGES'

9. git add all changes

10. push all repositories



## last words

This was created and implemented in one session and was just designed to work at all. Improvements welcome! ;)
