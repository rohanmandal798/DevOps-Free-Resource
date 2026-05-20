1.  git init
    → Initialize a new Git repository

2.  git clone <url>
    → Clone a repository

3.  git status
    → Show working tree status

4.  git add <file>
    → Add a file to staging area

5.  git add .
    → Add all changes to staging area

6.  git commit -m "msg"
    → Commit changes with a message

7.  git log
    → Show commit history

8.  git log --oneline
    → Show condensed commit history

9.  git log --graph
    → Show branch history graph

10. git diff
    → Show unstaged changes

11. git diff --staged
    → Show staged changes

12. git diff <file>
    → Show changes in a file

13. git diff <c1> <c2>
    → Show differences between commits

14. git show <commit>
    → Show details of a commit

15. git remote -v
    → List remote repositories

16. git remote add <name> <url>
    → Add a new remote repository

17. git fetch
    → Fetch updates from remotes

18. git pull
    → Fetch and merge changes

19. git push
    → Push changes to remote

20. git push -u origin <branch>
    → Push and set upstream branch

21. git branch
    → List all branches

22. git branch <name>
    → Create a new branch

23. git checkout <branch>
    → Switch to a branch

24. git checkout -b <branch>
    → Create and switch to a new branch

25. git merge <branch>
    → Merge branch into current branch

26. git rebase <branch>
    → Rebase current branch

27. git rebase -i <commit>
    → Interactive rebase

28. git stash
    → Temporarily save changes

29. git stash list
    → List saved stashes

30. git stash pop
    → Apply and remove stash

31. git stash apply
    → Apply stash without deleting it

32. git reset <file>
    → Unstage a file

33. git reset --soft <commit>
    → Reset but keep staged changes

34. git reset --mixed <commit>
    → Reset and unstage changes

35. git reset --hard <commit>
    → Reset and discard all changes

36. git restore <file>
    → Discard changes in file

37. git restore --staged <file>
    → Unstage file (new method)

38. git clean -fd
    → Remove untracked files/folders

39. git tag
    → List tags

40. git tag <name>
    → Create a tag

41. git tag -d <name>
    → Delete a tag

42. git push origin <tag>
    → Push a tag to remote

43. git branch -d <branch>
    → Delete a branch

44. git branch -D <branch>
    → Force delete a branch

45. git cherry-pick <commit>
    → Apply specific commit

46. git revert <commit>
    → Revert a commit safely

47. git bisect start
    → Start bug finding process

48. git bisect bad
    → Mark current commit as bad

49. git bisect good <commit>
    → Mark commit as good

50. git bisect reset
    → End bisect session