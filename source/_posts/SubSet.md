---
title: SubSet
date: 2015-01-05 15:16:24
tags: 算法
---

### Subset

```
public List<List<Integer>> subsets(int[] set) {
        Arrays.sort(set);
        int totalNumber = 1 << set.length;
        List<List<Integer>> collection = new ArrayList<List<Integer>>(totalNumber);
        for (int i=0; i<totalNumber; i++) {
            List<Integer> subSet = new LinkedList<Integer>();
            for (int j=0; j<set.length; j++) {
                if ((i & (1<<j)) != 0) {
                    subSet.add(set[j]);
                }
            }
            collection.add(subSet);
        }
        return collection;
    }
```