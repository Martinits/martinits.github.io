## Git Cheat Sheet

- 找到当前HEAD在某一个branch中的下一个commit

  `git log --reverse --pretty=%H ${BRANCH} | grep -A 1 $(git rev-parse HEAD) | tail -n1`

